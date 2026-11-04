# Windows 操作步骤

[手册首页](README.md) · [共用分流与 DNS](ROUTING.md) · [验收清单](VERIFICATION.md)

适用于 Clash Verge Rev、Claude Code 和 Chrome。三个设置的作用域需要分别核对。TUN 接管系统流量，Claude Code 在启动时读取代理和进程时区，Chrome 自身的防护决定网页能否取得 WebRTC 候选。

## 1. 记录现状并备份

1. 在 Clash Verge 的订阅页记录当前订阅和普通策略组选择。在设置页记录混合代理端口、TUN 开关和当前模式。
2. 从应用的配置目录入口找到当前订阅关联的持久覆写。常见位置为 `%APPDATA%\io.github.clash-verge-rev.clash-verge-rev\profiles`。在其他本地目录备份相关脚本和设置。
3. 备份 `~/.claude/settings.json`。记下原有 `env` 中已存在的键，便于逐项撤销。
4. 在 PowerShell 运行 `Get-TimeZone`，记录系统时区。

## 2. 选择规则模式并启用 TUN

在 Clash Verge 选 **Rule**，启用 TUN。核对生成配置中包含以下字段。把片段合并到当前配置，保留原来的接口选项。

```yaml
mode: rule
find-process-mode: strict
tun:
  enable: true
  auto-route: true
  strict-route: true
  dns-hijack:
    - any:53
    - tcp://any:53
```

`auto-route` 和 `strict-route` 用于接管路由。两条劫持规则覆盖常见 UDP 和 TCP 的 53 端口查询。网页或应用自带的 HTTPS 解析还要通过连接记录检查。

## 3. 应用专用出口和 DNS

按共用步骤配置节点、策略组、域名规则和 DNS。将 Windows 的 Chrome UDP 规则置于最前，再接 Claude 规则和原订阅规则。保存到**当前订阅的持久覆写**，应用订阅后查看最终生成配置。[共用配置步骤](ROUTING.md)

分别打开 Claude 的受测域名与哔哩哔哩。检查核心连接页，前者应进入 `Claude-Fixed`，后者在原规则允许时命中 `DIRECT`。节点名变化后，也要检查专用组的实际成员。

## 4. 限制 Chrome 的 WebRTC 地址获取

1. 打开现有 **WebRTC Control → 选项**。在本次使用的版本中，D 选 **Disable non proxied udp**。
2. 勾选 A **Block WebRTC Support Detection** 和 C **Block Additional WebRTC Objects**。确认扩展保护显示 **ON**。
3. 刷新测试页，分别检查普通页面和 iframe，并使用含国内、海外目标的多 STUN 测试。
4. 在连接页检查 Chrome 的 UDP。首条后备规则为 `AND,((PROCESS-NAME,chrome.exe),(NETWORK,UDP)),REJECT`。它也会影响 Chrome 的 QUIC、网页通话和会议功能。

最初只测海外 STUN 时漏掉了国内公网候选。扩大目标范围后发现直出，再加固并复测，受测页面的候选列表为空。Windows 这轮未观察到后备规则的动态命中。系统级 Chrome WebRTC 策略曾写入失败，最终验证依据来自扩展设置和页面实测。

## 5. 设置 Claude Code 的代理与时区

在用户级 `~/.claude/settings.json` 的 `env` 中合并下列键。示例使用本机 HTTP 混合端口 `7897`，请按界面实际端口填写。已有的权限、MCP 和其他应用设置应保留。

```json
{
  "env": {
    "HTTP_PROXY": "http://127.0.0.1:7897",
    "HTTPS_PROXY": "http://127.0.0.1:7897",
    "NO_PROXY": "localhost,127.0.0.1,::1",
    "TZ": "America/Los_Angeles"
  }
}
```

完全退出旧 Claude Code 进程，再启动新进程。若代理路径不符，还要检查小写代理环境变量的优先级。历史验证使用隔离的 CLI 配置、无效测试凭据和只返回 503 的环回代理，观测到了进程内时区变化及 HTTPS CONNECT 请求。[Claude Code 网络配置](https://code.claude.com/docs/en/network-config)

## Chrome 新标签页恢复

独立时区实验曾让五种网页环境返回洛杉矶时区，随后接管的新标签页报错。实际恢复过程如下。

1. 在 Chrome 顶部取消时区扩展的调试提示。
2. 打开 `chrome://extensions`，移除自制的洛杉矶时区扩展。**WebRTC Control 是另一项独立防护**。
3. 关闭报错的旧标签页，按 `Ctrl+T` 新建一页，再分别测试输入网址、刷新及普通网站导航。
4. PowerShell 的 `Get-TimeZone` 应显示 `China Standard Time`。恢复后普通页面、跨源 iframe、Web Worker、Shared Worker、Service Worker 均报告 `Asia/Shanghai`。一月和七月的 `getTimezoneOffset()` 均为 `-480`。

若还需把系统调回东八区，可在 **设置 → 时间和语言 → 日期和时间** 选择 UTC+08:00 的北京时区。Claude Code 的 `TZ` 只作用于其进程。Chrome 独立洛杉矶时区这一目标尚未实现。

## 6. 验收与回退

- **分流与解析**　核对国内、Claude 和 DoH 三条路径。Windows 保留系统 IPv6 栈和 TUN 接管，DNS 抑制 AAAA，专用节点优先使用 IPv4。
- **浏览器**　记录多目标 STUN 的候选类型、地址类别、超时情况。如需网页会议，单独评估严格 UDP 策略的影响。
- **撤销网络配置**　备份当前状态，恢复修改前的持久覆写并重新应用订阅。恢复 TUN 原值和普通组选择，再验国内访问。
- **撤销 CLI 配置**　把本次涉及的四个环境键恢复为备份值，完全退出并重启 CLI。

完整的通过条件与未测项见 [11 项验收](VERIFICATION.md)。
