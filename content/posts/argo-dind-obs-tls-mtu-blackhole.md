---
title: DinD 中 OBS 请求超时：根因是 MTU 黑洞
date: 2026-09-02
draft: false
tags:
  - Kubernetes
  - Docker
  - Podman
  - 网络
  - 故障排查
categories:
  - 故障排查
description: 华为云 OBS 请求在 DinD 容器中超时，最终定位为 DinD bridge MTU 与 Kubernetes CNI MTU 不匹配导致的 TLS 握手黑洞。
---

在容器中调用华为云 OBS 时，请求在 DNS 解析成功后长时间没有响应，最终超时。相同 endpoint 从 Pod 的 host network 访问可以正常返回 `403`，因此问题集中在容器网络路径，而不是 OBS 服务本身。

## 先看服务端日志

失败容器的 runtime 日志有一条很重要的时间线：

```text
Endpoint=https://obs.cn-north-4.myhuaweicloud.com:443/
internet host address:
  phoenix-autotest.obs.cn-north-4.myhuaweicloud.com/123.60.240.82,
  phoenix-autotest.obs.cn-north-4.myhuaweicloud.com/123.60.240.83
```

这说明 DNS 解析已经完成。之后没有 HTTP 状态码，也没有 OBS 的业务异常。因此，失败点位于 DNS 之后、HTTP 响应之前，更接近 TCP/TLS，而不是 bucket、路径或签名参数。

产品的华为云插件确实直接使用 OBS SDK：

```java
this.obsClient = new ObsClient(
        settings.accessKeyId(),
        settings.secretAccessKey(),
        settings.endpoint());
```

目录操作会调用 `listObjects`。在本机使用相同的 OBS 配置和 SDK 3.25.10 重放，接口可以正常返回。因此没有证据表明 AK/SK 或配置错误。

## 对照不同网络路径

运行服务的 Pod 使用 DinD sidecar：

```yaml
DOCKER_HOST: unix:///var/run/docker/docker.sock
```

Phoenix 容器位于 DinD 的 bridge 网络中。在同一个 Pod 中做只读 HTTPS 探测，结果非常明确：

| 网络位置 | OBS endpoint 结果 |
| --- | --- |
| Pod/host network | 约 70~100 ms 返回 `403` |
| DinD bridge，MTU 1500 | TLS 握手超时 |
| DinD bridge，MTU 1490 | TLS 握手超时 |
| DinD bridge，MTU 1472 | TLS 握手超时 |
| DinD bridge，MTU 1460 | 正常返回 `403` |
| DinD bridge，MTU 1450 | 正常返回 `403` |

这里的 `403` 反而是成功信号：服务端已经收到请求，只是没有提供认证信息。超时则说明请求根本没有走到 OBS 的 HTTP 层。

## MTU 为什么会造成 TLS 超时

当前 Pod 的网络接口是：

```text
eth0 mtu 1450
```

这是 Kubernetes CNI 覆盖网络常见的 MTU。可是 DinD 中的 Docker bridge 默认是：

```text
docker0 mtu 1500
```

Phoenix 容器的路径变成了：

```text
Phoenix 容器 (1500)
  -> Docker bridge (1500)
  -> DinD Pod eth0 (1450)
  -> Kubernetes 节点和外部网络
```

容器发出的较大 TCP/TLS 报文经过 Pod 的 1450 MTU 链路时，需要分片或依靠 PMTU Discovery 降低报文大小。如果中间设备没有把 `Fragmentation Needed` 正确传回，或者相关 ICMP 报文被过滤，TCP 连接可能已经建立，但 TLS 握手在等待一个永远不会到达的报文，于是表现为“连接超时”。

这也解释了为什么 DNS 正常、TCP 看起来有连接、应用却一直收不到 OBS 响应。

## 为什么以前的 Podman 5.8.2 没有暴露这个问题

历史环境使用：

```text
quay.io/podman/stable:v5.8.2-immutable
```

应用容器通过共享 Unix socket 访问 Podman 的 Docker 兼容 API：

```text
DOCKER_HOST=unix:///var/run/podman/docker.sock
podman system service --time=0 unix:///var/run/podman/docker.sock
```

同时挂载了：

```toml
[containers]
log_driver = "json-file"
netns = "bridge"
```

Podman 5 使用 netavark 创建 bridge 网络。netavark 在未指定网络 MTU 时，会读取默认路由接口的 MTU，并将其用于容器 veth。也就是说，在 Pod 的默认网卡是 1450 时，Podman 创建的容器通常也会拿到 1450，而不是固定使用 Docker bridge 的 1500。

## 修复方式

最直接的修复是在 DinD daemon 启动时显式指定 MTU，并与 Pod 的实际接口保持一致：

```yaml
args:
  - "--mtu=1450"
```

如果不同节点的 CNI MTU 不同，不应把 `1450` 当成永久常量。更稳妥的方案是让 DinD 启动脚本读取默认路由网卡 MTU，再生成 Docker daemon 配置，或者统一集群网络 MTU。

## 排查这类问题的顺序

遇到请求超时，可以按下面顺序切分：

```text
DNS
  -> TCP connect
  -> TLS handshake
  -> HTTP status
  -> SDK 认证/业务
```

这次故障的最终结论是：华为云服务正常，Phoenix OBS 请求参数正常，失败发生在 DinD bridge 到 Kubernetes Pod 网络的 MTU 不匹配；修正 Docker bridge MTU 后，问题才真正消失。
