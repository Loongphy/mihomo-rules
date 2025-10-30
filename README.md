# Mihomo 分流规则

自用，仅作为学习用途，针对常用网站和软件分流。

> [!CAUTION]
> Cursor 使用 Claude 模型需开启 TUN 模式，非 Windows 用户需要调整 PROCESS-NAME 规则。

## DNS

DNS 配置我更倾向于在客户端配置，请参考 [dns.yml](./dns.yaml) 自行配置。

目前使用的是：

- 桌面端 [Sparkle](https://github.com/xishang0128/sparkle)
- 安卓端 [FClash](https://github.com/chen08209/FlClash)

## 端口分流

- 目的：为特定应用指定线路
- 场景：你希望指纹浏览器走香港节点，同时又希望某个游戏客户端走日本节点以获得更低的游戏延迟。
- 实现：

1. 在指纹浏览器的代理设置中，将代理服务器设置为 SOCKS5 代理 127.0.0.1:42000。
2. 在游戏加速工具（如 Proxifier）或游戏客户端内置的代理设置中，将代理服务器设置为 127.0.0.1:42001。

效果：两个应用通过连接不同的端口，实现了自动分流，互不干扰，无需编写复杂的域名或进程名规则。

```yaml
listeners:
  - { name: hk-port, type: mixed, port: 42000, proxy: 🇭🇰 香港自动 }
  - { name: jp-port, type: mixed, port: 42001, proxy: 🇯🇵 日本自动 }
```

## 覆写规则

分为两类节点：**手动选择**和**自动**，手动选择更适合用于 AI 等需要特定节点的场景，自动更适合日常使用。

部分代理组会根据个人喜好调整顺序，强迫症请自行调整。

### 额外说明

- Gemini 与 Google 使用同一分组：🔍 Google 。因为 Gemini 不要求 IP 质量，但要求某些 Google 相关域名同 Gemini 请求 IP 一致，建议日用 🔍 Google 选择美国自动。
- Youtube 可单独选择分组，不必与 Google 一致。
- 非 Windows 用户需要调整 PROCESS-NAME 规则。
