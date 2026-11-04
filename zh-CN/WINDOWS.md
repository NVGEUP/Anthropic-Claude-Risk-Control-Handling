# Windows 操作步骤

[手册首页](README.md) · [共用分流与 DNS](ROUTING.md) · [11 项验收](VERIFICATION.md)

用 Clash Verge Rev 决定连接走哪个出口，用 Claude Code 的用户设置指定本机代理，用 Chrome 的 WebRTC 设置限制网页能获取的地址。这三处可以分别验收。

## 1. 留一份能还原的基线

1. 在 Clash Verge Rev 的**订阅**和**设置**中记下活动订阅、普通组当前选择、混合端口、模式和 TUN 开关。端口要沿用实际值；稍后 CLI 会连接它。
2. 从应用的配置目录入口找到活动订阅的覆写脚本，复制到本机备份目录。常见位置为 `%APPDATA%\io.github.clash-verge-rev.clash-verge-rev\profiles`。订阅更新会重新生成运行配置，所以要备份能再次生成它的脚本。
3. 备份 `~/.claude/settings.json`，记下已存在的 `env` 值、Chrome 的 WebRTC 选项，并运行 `Get-TimeZone`。这样可以分别还原 CLI、浏览器和代理，也能在最后比较系统时间。

## 2. 让流量进入核心，再按规则分流

在 Clash Verge Rev 选 **Rule**、启用 **TUN**。把以下字段合并到已有配置，保留原接口选项。应用后查看客户端实际运行的配置。

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

| 设置 | 为什么这样设 |
| --- | --- |
| `mode: rule` | 国内 HTTPS 可以匹配 `DIRECT`，Claude 可以匹配另一个组。两者共用同一个核心，却走不同出口。 |
| `auto-route`、`strict-route` | 前者让系统把相应流量交给虚拟网卡；后者在 Windows 收紧路由，并限制多网卡同时发起的 DNS 查询。TUN 停止后应重新判断实际路径。 |
| `any:53`、`tcp://any:53` | 第一条默认截获 UDP 53，第二条覆盖 TCP 53。浏览器自带 DoH 通常走 HTTPS，需另查它连接了哪个解析器。 |
| `find-process-mode: strict` | 核心在规则需要时识别进程，这样才能只匹配 `chrome.exe` 的 UDP。 |

[Mihomo TUN](https://wiki.metacubex.one/config/inbound/tun/) · [进程匹配模式](https://wiki.metacubex.one/config/general/)

## 3. 把专用组和 DNS 绑定到活动订阅

1. 在当前订阅的持久脚本中应用[共用配置](ROUTING.md)，Windows 保持 `android = false`。保存并重新应用订阅。打开生成配置，确认 Chrome UDP 规则在最前，专用组只有目标节点。这样才表示客户端已把编辑内容交给核心。[Clash Verge Rev 脚本入口](https://clash-verge-rev.github.io/guide/script.html)
2. 打开**连接**，分别触发一条新的 Claude 请求和一条哔哩哔哩 HTTPS 请求。前者应命中 `Claude-Fixed`，后者在原国内规则下应为 `DIRECT`。同时看“匹配规则”和“最终出站”两栏；网页能打开只说明有一条路能通。

## 4. 限制 Chrome 向网页提供 WebRTC 地址

1. 在 **WebRTC Control → 选项**中，将 D 设为 **Disable non proxied udp**。网页通话会尝试寻找双方可达的网络地址。这个选项限制 WebRTC 直接发送未代理的 UDP，减少 STUN 从另一出口发现公网地址的机会。
2. 勾选 A **Block WebRTC Support Detection** 和 C **Block Additional WebRTC Objects**，确认保护显示 **ON**。这两项限制页面探测或调用相关接口。刷新测试页，让设置作用于新文档；同时测主页面和 iframe，因为它们是独立的脚本环境。
3. 核对核心规则 `AND,((PROCESS-NAME,chrome.exe),(NETWORK,UDP)),REJECT`。它拦截核心识别到的 Chrome UDP，因此 QUIC、语音和视频也可能受影响。用国内和海外 STUN 目标分别采样：初次只测海外目标时，曾漏掉国内目标发现的直出地址。

记录中，收紧设置后受测 Windows 页面未给出候选地址；当轮未观察到核心 UDP 后备规则的命中。两项观察应分别记录。[WebRTC IP 地址处理原理](https://www.rfc-editor.org/rfc/rfc8828.html) · [候选类型与判断标准](VERIFICATION.md)

## 5. 设置 Claude Code 的代理和进程时区

将以下键**合并**到用户级 `~/.claude/settings.json` 的 `env`，保留已有配置。把两处 `7897` 换成第 1 步记录的混合端口。

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

- `HTTP_PROXY` 和 `HTTPS_PROXY` 让 CLI 把相应请求交给环回地址上的代理。值里的 `http://` 描述 CLI 如何连接本机代理；访问 HTTPS 目标时，CLI 可通过 `CONNECT` 建隧道，再跟目标建立 TLS。
- `NO_PROXY` 让本机服务直接互访。把远程业务域名也放进去，会让相应请求跳过 CLI 的显式代理。若日志没出现预期连接，检查已有的大小写代理变量及优先级。[Claude Code 网络配置](https://code.claude.com/docs/en/network-config)
- `TZ` 是进程时间的显示规则，`America/Los_Angeles` 包含夏令时。完全退出并重新启动 CLI，使新的进程读取设置；在独立 PowerShell 运行 `Get-TimeZone`，可核对 Windows 的东八区。改变显示时区不会改变网络出口。

**怎么核验。** 新会话中触发正常请求，查看本机代理的连接记录。如果已装 Node，可在 CLI 的本地命令模式输入 `! node -p "Intl.DateTimeFormat().resolvedOptions().timeZone"`，检查子进程读到的时区；普通 PowerShell 里的 Node 只代表该终端。历史测试另用隔离配置记录了 CLI 进程时区；一个只返回 `503` 的本机测试代理收到了 `CONNECT api.anthropic.com:443`，证实 CLI 找到了代理，但这次测试没有转发到 API。[CLI 本地命令模式](https://code.claude.com/docs/en/interactive-mode)

## 6. 验收和回退

按[验收清单](VERIFICATION.md)分别核对国内 HTTPS、Claude 出站、DNS 上游、IPv6 和浏览器候选。若需还原，先留存现状，再恢复原订阅脚本并重新应用，恢复 TUN、模式和普通组选择。把改动过的四个 `env` 键及 WebRTC 选项恢复为原值，重启 CLI、刷新网页，使用相同测试对照。
