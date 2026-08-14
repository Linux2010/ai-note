# OpenClaw Control UI HTTP 远程访问配置

**文档类型**: 技术解决方案
**适用场景**: 通过公网 HTTP 访问 OpenClaw Control UI
**最后更新**: 2026-04-05

---

## 问题描述

当通过公网 HTTP（非 HTTPS）访问 OpenClaw Control UI 时，会遇到以下错误：

```
control ui requires device identity (use HTTPS or localhost secure context)
```

**根本原因**：
- Control UI 需要 **secure context**（HTTPS 或 localhost）才能使用 WebCrypto
- WebCrypto 用于生成 device identity（设备身份验证）
- HTTP 远程访问 → 浏览器阻止 WebCrypto → 无法完成设备认证 → 报错

---

## 解决方案

### 方案 1: 临时测试 - 禁用设备认证 ⚠️

**警告：这是严重安全降级，仅用于测试环境！**

配置修改：
```json5
{
  gateway: {
    mode: "remote",
    bind: "lan",  // 或 "0.0.0.0"
    controlUi: {
      allowedOrigins: ["*"],
      allowInsecureAuth: true,
      dangerouslyDisableDeviceAuth: true  // 关键配置
    },
    auth: {
      mode: "token",
      token: "your-secure-token"
    }
  }
}
```

**注意事项**：
- `allowInsecureAuth: true` **只对 localhost 有效**，不对远程生效
- `dangerouslyDisableDeviceAuth: true` 禁用所有设备身份检查
- 任何拥有 token 的人都能访问，无设备绑定
- 生产环境 **绝对不要使用此配置**

---

### 方案 2: Tailscale Serve（推荐）

使用 Tailscale 提供 HTTPS，最安全的方式：

```json5
{
  gateway: {
    bind: "loopback",
    tailscale: { mode: "serve" }
  }
}
```

访问地址：`https://<magicdns>/`

---

### 方案 3: Tailscale Funnel（公网 HTTPS）

将服务暴露到公网，自动提供 HTTPS：

```json5
{
  gateway: {
    bind: "loopback",
    tailscale: { mode: "funnel" },
    auth: { mode: "password" }  // Funnel 必须用密码认证
  }
}
```

---

### 方案 4: frp + IP 白名单（内网穿透方案）

使用 frp 进行端口转发，配合安全组/IP 白名单实现访问控制：

```ini
# frpc.ini
[openclaw-control-ui]
type = tcp
local_ip = 127.0.0.1
local_port = 18789
remote_port = 18789
```

**安全组配置**：
- 只允许固定 IP 访问远程端口
- 云服务商安全组 / 防火墙规则

**优点**：
- 无需 HTTPS 证书配置
- IP 白名单提供访问控制
- 适合固定 IP 的团队/个人使用

**缺点**：
- HTTP 无加密，流量可被监听
- IP 变化时需要更新白名单

---

### 方案 5: Nginx 反向代理 + HTTPS（生产环境推荐）

使用 Nginx/Caddy 等反向代理配置 HTTPS 证书：

```nginx
server {
    listen 443 ssl;
    server_name your-domain.com;
    
    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;
    
    location / {
        proxy_pass http://127.0.0.1:18789;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

推荐使用 Let's Encrypt 免费 SSL 证书。

---

## 配置优先级说明

| 配置项 | 作用范围 | 安全等级 |
|--------|----------|----------|
| `allowInsecureAuth` | 仅 localhost | 低风险 |
| `dangerouslyDisableDeviceAuth` | 所有来源 | **高风险**（需配合 IP 白名单）|
| frp + IP 白名单 | 公网（限 IP）| 中安全（HTTP 无加密）|
| Tailscale Serve | Tailnet 内 | 高安全 |
| Tailscale Funnel | 公网 | 中安全（需密码）|
| Nginx + HTTPS | 公网 | 高安全 |

---

## 实际配置示例

用户访问地址：`http://39.105.196.245:18789/chat?session=main`

完整配置：
```json5
{
  gateway: {
    port: 18789,
    mode: "remote",
    bind: "lan",
    controlUi: {
      allowedOrigins: ["*"],           // 允许任何来源
      allowInsecureAuth: true,          // localhost 兼容
      dangerouslyDisableDeviceAuth: true  // 禁用设备认证（测试用）
    },
    auth: {
      mode: "token",
      token: "your-secure-token-here"
    }
  }
}
```

---

## 相关文档

- [OpenClaw Web 文档](https://docs.openclaw.ai/web)
- [Control UI 文档](https://docs.openclaw.ai/web/control-ui)
- [Gateway Remote Access](https://docs.openclaw.ai/gateway/remote)
- [Tailscale 配置](https://docs.openclaw.ai/gateway/tailscale)

---

## 总结

| 场景 | 推荐方案 |
|------|----------|
| 本地开发测试 | `dangerouslyDisableDeviceAuth` + Token |
| 内网穿透 + 固定 IP | frp + IP 白名单 + Token |
| 内网团队使用 | Tailscale Serve |
| 公网生产环境 | Nginx + HTTPS + Token |
| 公网快速部署 | Tailscale Funnel + Password |

**记住**：HTTP 远程访问 = 降级安全。生产环境必须用 HTTPS。