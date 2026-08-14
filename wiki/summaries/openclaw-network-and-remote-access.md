---
id: summary-openclaw-network-and-remote-access
type: source-summary
title: "OpenClaw 网络、代理与远程访问"
status: draft
updated: 2026-08-14
owner: "llm-wiki-expert"
sources:
  - ../../raw/sources/openclaw/solutions/control-ui-http-remote-access.md
  - ../../raw/sources/openclaw/solutions/macos-tcp-connection-storm.md
  - ../../raw/sources/openclaw/solutions/proxy-solutions.md
---

# OpenClaw 网络、代理与远程访问

## 概览

OpenClaw 的网络问题覆盖三个不同层面：Channel 或模型请求如何使用代理，容器如何访问宿主机网络，以及 Control UI 如何安全地暴露给远程浏览器。排障时必须先确定故障层级；“网页打不开”“Telegram 不回复”和“所有外部请求同时失败”通常不是同一种问题。

## 核心内容与机制

### 代理配置

代理可以通过进程环境变量或 Channel 专属配置注入。环境变量适合统一、临时的出口策略，Channel 配置适合只代理 Telegram 等特定流量。多实例环境应明确每个进程继承了哪些变量；系统代理并不保证被 OpenClaw 的 HTTP 客户端采用。

容器内访问宿主机代理不能使用 `127.0.0.1`，通常应使用 Docker 提供的宿主机名称或显式网关映射。代理是否监听局域网接口、防火墙是否放行以及 DNS 是否可解析，都需要从容器内部验证。

### Control UI 的安全上下文

远程 HTTP 页面可能无法使用设备身份所需的 WebCrypto secure context。临时禁用设备认证会显著降低安全性，只适合受控测试。长期远程访问应优先使用 Tailscale Serve、受认证的 HTTPS 反向代理，或其他能提供加密与访问控制的方案。公网暴露不能仅依赖可猜测 Token。

### macOS 连接耗尽

来源记录了一次代理与 Docker 连接产生大量 `TIME_WAIT`、`FIN_WAIT_1`，最终耗尽 macOS 临时端口的事故。症状是 Telegram 和模型请求同时失败。诊断应统计 TCP 状态、目标分布和代理监听情况；应急重启只能恢复资源，预防措施还包括移除死节点、降低健康检查频率和监控连接数。

该事故文档把未释放状态归因于 macOS TCP 栈问题，但这是单次生产观察，不应推广为所有类似故障的确定结论。

## 实践要点与决策依据

- 内网个人访问优先安全隧道；公网生产访问优先 HTTPS 反向代理、强认证和最小暴露面。
- `dangerouslyDisableDeviceAuth` 属于临时风险接受项，必须限制来源并设定撤销时间。
- 代理排障分别测试宿主机直连、宿主机经代理、容器到代理和容器经代理到目标 API。
- 多实例不要无意共享错误的全局代理；记录每个实例的出口、端口和 `NO_PROXY` 范围。
- Telegram 与模型同时失效时，优先检查共同的代理、DNS、端口和系统连接资源。
- 凭证型代理 URL、Control UI Token 和公网地址不进入共享日志或 wiki 示例。

## 相关主题

- [[summaries/openclaw-telegram-connectivity-troubleshooting|OpenClaw Telegram 连通性排障]]
- [[summaries/openclaw-installation-and-runtime-management|OpenClaw 安装与运行时管理]]
- [[summaries/openclaw-runtime-configuration|OpenClaw 运行时配置体系]]
