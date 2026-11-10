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

## 微软分流（ms-proxy）

`ms-proxy.yaml` 是远程规则集（rule-provider），把"IP 区域敏感"的微软域单独捞出来走代理，其余微软域全部直连。覆写中引用方式：

```yaml
rule-providers:
  ms-proxy:
    type: http
    behavior: classical
    format: yaml
    url: "https://cdn.jsdelivr.net/gh/Loongphy/mihomo-rules@main/ms-proxy.yaml"
    path: ./rules/ms-proxy.yaml
    interval: 86400

rules:
  # 遥测/埋点直接拒绝（纯上报通道，不影响功能）
  - GEOSITE,microsoft@telemetry,REJECT      # data./telemetry./applicationinsights/aria.ms/vortex 等
  - DOMAIN-SUFFIX,aria.microsoft.com,REJECT # ARIA 管道 browser.pipe.aria
  - DOMAIN-SUFFIX,clarity.microsoft.com,REJECT
  - DOMAIN-SUFFIX,bat.bing.com,REJECT       # 广告点击跟踪（顺手省代理流量）
  - DOMAIN-SUFFIX,arc.msn.com,REJECT        # MSN 广告上报

  # github 必须在 microsoft 兜底前（geosite:microsoft 包含 github）
  - GEOSITE,github,🚀 默认

  # IP 区域敏感业务 → 代理（组内手动锁定目标节点，别用自动组）
  - RULE-SET,ms-proxy,Ⓜ️ Microsoft

  # Azure OpenAI（须在 GEOSITE,microsoft 直连兜底之前）
  - DOMAIN-SUFFIX,openai.azure.com,💬 OpenAI

  # 其余微软全部直连（下载/更新/OneDrive 客户端同步/Xbox/SharePoint/CDN）
  - GEOSITE,microsoft,DIRECT
```

> [!NOTE]
> `GEOSITE,microsoft` 包含 github/onedrive/xbox 等子集，顺序必须保证 github 规则在前。

### 覆盖的产品

| 分组 | 域名 | 说明 |
|---|---|---|
| Bing 搜索 + Microsoft Rewards | `bing.com` `bing.com.cn` `bing.net` `bingapis` `bingstatic` `bingvisualsearch` `dictate.ms` `engkoo.com` `microsoftapp.net` `copilot.*` | 搜索、`rewards.bing.com`、`/rewards/panelflyout`、`/msrewards/api` 全部同源覆盖；Rewards 计分要求与登录态同 IP 区域 |
| MSA/AAD 登录态 | `login./account./signup./hip.live.com` `microsoftonline.com` `account.microsoft.com` `msauth*` `msftauth*` `msftidentity` `passport.net` `autologon.microsoftazuread-sso.com` | Rewards 与 M365 共用；刻意不含 `live.com` 后缀（`storage.live.com` 是下载域） |
| OneDrive 网页端 | `onedrive.live.com` | 网页外壳在大陆被 DNS 投毒，必须代理；客户端同步/下载域（`storage.live.com` `api.onedrive.com` `*.files.1drv.com` `1drv.ms`）直连可用，不代理 |
| M365 门户 | `cloud.microsoft`（m365/word/excel/powerpoint/outlook/teams/admin…）`office.com` `office365.com` `microsoft365.com` `officeapps.live.com` `graph./substrate.*` | Copilot 区域判定走 auth+API，已全部覆盖 |
| Office 在线应用/内容 | `microsoftpersonalcontent.com` `officeapps.live.com` `onenote.com` `sway.com` `microsoftstream.com` `whiteboard.microsoft.com` | OneDrive 个人内容、OneNote、Sway、Stream、白板 |
| 协作 | `teams.microsoft.com` `microsoftteams.com` `trouter.skype.com` `yammer.com` `yammerusercontent.com` | Teams 网页版及消息通道、Viva Engage |
| 邮箱 | `outlook.com` `outlook.live.com` | Outlook 网页版/API |
| Power Platform | `powerbi.com` `powerapps.com` `powerautomate.com` | 官方 M365 端点列表收录 |

### 刻意走直连的

- **下载/更新/CDN**：`*.mp.microsoft.com`、`windowsupdate/update.microsoft.com`、`officecdn*`、`cdn.office.net`、`msocdn`/`msecnd`/`azureedge`、`sharepoint.com`、`storage.live.com`、`api.onedrive.com`、`*.files.1drv.com`、`1drv.ms`、`xboxlive`、`blob.core.windows.net` —— 由 `GEOSITE,microsoft→DIRECT` 兜底
- **遥测/广告**：`*.events.data.*`、`aria.microsoft.com`、`clarity.*`、`arc.msn.com`、`bat.bing.com` —— 覆写里 `REJECT`（纯上报，不影响功能）
- **CDN/静态**：`res.cdn.office.net`、`shell.cdn.office.net`、`uhf/wcpstatic.microsoft.com` —— 直连更快，不参与区域判定

### Edge Copilot 图标显隐

`msedge.exe` 每隔几分钟拉取 `edge.microsoft.com` 与 `config.edge.skype.com`（Edge 服务配置 / ECS 功能开关），**按请求 IP 下发地区功能开关**：

- **直连（本仓库现状）**：CN IP → `variations_country=CN` → Copilot 侧栏图标隐藏 ✅ 想隐藏就保持直连
- **走代理**：非 CN IP → Copilot 图标恢复

如需恢复图标，把这三条加进 `ms-proxy.yaml`（或覆写 rules 中 `GEOSITE,microsoft` 之前）：

```yaml
- DOMAIN-SUFFIX,edge.microsoft.com
- DOMAIN,config.edge.skype.com
- DOMAIN-SUFFIX,api.cdp.microsoft.com
```

已写入 `Local State` 的 `variations_country` 可能需重启 Edge 后随下次配置下发刷新。Copilot 聊天本体域（`copilot.microsoft.com`、`copilot.com`、`sydney.bing.com`）一直在 `ms-proxy` 内，不受影响。

## 额外说明

- Gemini 与 Google 使用同一分组：🔍 Google 。因为 Gemini 不要求 IP 质量，但要求某些 Google 相关域名同 Gemini 请求 IP 一致，建议日用 🔍 Google 选择美国自动。
- Youtube 可单独选择分组，不必与 Google 一致。
- 非 Windows 用户需要调整 PROCESS-NAME 规则。
