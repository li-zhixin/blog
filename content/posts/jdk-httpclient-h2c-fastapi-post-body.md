---
title: JDK HttpClient 遇上 FastAPI：一次 POST 请求体消失问题的排查
date: 2026-07-30
draft: false
tags:
  - Java
  - HTTP/2
  - h2c
  - OkHttp
  - FastAPI
categories:
  - 工程实践
description: 同一个 POST 请求在 fetch 中成功，在 JDK HttpClient 中却返回 400 或丢失请求体。根因是明文 HTTP 上的 h2c Upgrade 与 Python HTTP parser 不兼容。
---

同一个 POST 请求，用 `fetch`、PowerShell 或 curl 调用都能成功，通过 JDK HttpClient 发出时却得到 `400`。换成 FastAPI 接口后，响应变成 `422`，服务端声称请求体不存在。

问题不在 JSON，而在 HTTP 协议协商。

## 现象

后端使用 Spring `RestTemplate`：

```java
new RestTemplate(new JdkClientHttpRequestFactory());
```

两组对照实验得到相同结论：

| 客户端 | 结果 |
| --- | --- |
| Node `fetch` | `200 OK` |
| PowerShell `Invoke-RestMethod` | `200 OK` |
| 默认 JDK HttpClient | `400 Invalid HTTP request received.` 或 `422 Body missing` |
| 强制 JDK HTTP/1.1 | `200 OK` |

唯一关键变量是 HTTP 版本。

## 根因

`JdkClientHttpRequestFactory` 默认创建 `HttpClient.newHttpClient()`。JDK HttpClient 偏好 HTTP/2；访问明文 `http://` 地址时，会尝试 h2c Upgrade：

```http
Connection: Upgrade, HTTP2-Settings
Upgrade: h2c
HTTP2-Settings: ...
User-Agent: Java-http-client/21
```

部分 Python HTTP parser 不能正确处理这个请求。它们可能直接返回 `400`，也可能让应用层看到一个没有 Body 的 POST，最终由 FastAPI 返回 `422`。

这也是为什么检查 JSON、Content-Type 和字段定义都找不到问题：请求在进入应用层之前已经被错误解析。

Spring Framework 官方仓库的 [#33275](https://github.com/spring-projects/spring-framework/issues/33275) 记录了同类现象。[维护者的结论](https://github.com/spring-projects/spring-framework/issues/33275#issuecomment-2252184391) 也是 JDK 默认 h2c 行为与 Python parser 的组合问题，而不是 Spring 序列化错误。

## 为什么不全局固定 HTTP/1.1

下面的修改可以立即解决问题：

```java
HttpClient client = HttpClient.newBuilder()
    .version(HttpClient.Version.HTTP_1_1)
    .build();
```

但如果整个外部接口客户端都固定为 HTTP/1.1，HTTPS 请求也无法再通过 ALPN 使用 HTTP/2。它适合临时止血，不适合作为全局策略。

对于已经稳定运行的旧版本，按 URL scheme 维护两套 JDK Client 是更实际的维护方案：`http://` 使用 HTTP/1.1，`https://` 保留默认的 HTTP/2 协商。

## OkHttp 的默认行为

OkHttp 4.12.0 的默认协议列表是：

```kotlin
internal val DEFAULT_PROTOCOLS = immutableListOf(HTTP_2, HTTP_1_1)
```

源码：[`OkHttpClient.kt`](https://github.com/square/okhttp/blob/parent-4.12.0/okhttp/src/main/kotlin/okhttp3/OkHttpClient.kt#L1072-L1077)。

建立明文连接时，只有显式配置 `H2_PRIOR_KNOWLEDGE` 才使用 HTTP/2，否则直接选择 HTTP/1.1：

```kotlin
if (route.address.sslSocketFactory == null) {
  if (Protocol.H2_PRIOR_KNOWLEDGE in route.address.protocols) {
    protocol = Protocol.H2_PRIOR_KNOWLEDGE
    startHttp2(pingIntervalMillis)
    return
  }

  protocol = Protocol.HTTP_1_1
  return
}
```

源码：[`RealConnection.kt`](https://github.com/square/okhttp/blob/parent-4.12.0/okhttp/src/main/kotlin/okhttp3/internal/connection/RealConnection.kt#L317-L343)。

HTTPS 则在 TLS 握手中通过 ALPN 提供 `h2` 和 `http/1.1`，没有协商结果时回退 HTTP/1.1：

```kotlin
val maybeProtocol = Platform.get().getSelectedProtocol(sslSocket)
protocol = if (maybeProtocol != null) {
  Protocol.get(maybeProtocol)
} else {
  Protocol.HTTP_1_1
}
```

源码：[`RealConnection.kt`](https://github.com/square/okhttp/blob/parent-4.12.0/okhttp/src/main/kotlin/okhttp3/internal/connection/RealConnection.kt#L362-L424)。

因此它的默认策略是：

```text
http://  -> HTTP/1.1，不主动 h2c Upgrade
https:// -> ALPN 协商 HTTP/2，必要时回退 HTTP/1.1
```

## 与当前客户端的行为差异

当前实现并非直接调用 `java.net.http.HttpClient`，而是经过 `RestTemplate`。它与直接使用 `OkHttpClient` 的差异如下：

| 行为 | RestTemplate + JDK HttpClient | OkHttpClient |
| --- | --- | --- |
| Body 编码 | RestTemplate 的 MessageConverter 负责 JSON、表单和文本转换 | 通过 `RequestBody`、`FormBody` 等类型显式构造 |
| 非 2xx 响应 | 当前自定义 ResponseErrorHandler 将响应转换为异常 | 返回普通 `Response`，可通过状态码或 `isSuccessful` 判断 |
| 重定向 | JDK HttpClient 默认 `Redirect.NEVER` | 默认跟随 HTTP 和 HTTPS 重定向 |

这次问题的本质不是“FastAPI 丢了 Body”，而是不同 HTTP 客户端生成了不同的协议报文。选择客户端时，默认协议策略同样属于应用契约。
