# 11 项验收与检测网站

[手册首页](README.md) · [Windows](WINDOWS.md) · [Android](ANDROID.md) · [共用分流与 DNS](ROUTING.md)

先确认当前订阅、**Rule** 和 TUN/VPN 均已生效，打开核心的**连接**页。从实际待测的浏览器或应用发起**新请求**，记下测试时间、目标、规则、出站和观测结果。旧连接可以保持旧路径；一个浏览器标签页也不能代表 Claude 进程。

## 逐项判断

| # | 怎么测 | 为什么这样测 | 本次观察与边界 |
| --- | --- | --- | --- |
| 1 · 公网出口 | 对同一路径用两个 HTTPS 回显服务采样；先在核心里确认探针命中目标组。 | 回显服务只看得到这一次 HTTPS 连接的公网来源。 | 短时样本一致。运营商和城市标签不能证明独享、信誉或真实物理位置。 |
| 2 · 出口稳定性 | 切换**普通组**，新建连接，复查专用组成员和出口。 | 若两组误用同一选择，切换普通节点也会改变 Claude 的路线。 | 专用组只有一个节点。节点名固定仍不能保证服务商长期分配同一 IP。 |
| 3 · TCP、UDP、WebRTC | 分别检查节点的 TCP/TLS 外层和浏览器的 ICE 候选，对国内、海外 STUN 各测一轮。 | 网页可另外开 UDP 套接字，HTTPS 的出口正确不能证明这条路径也正确。 | Windows 受测页面未返回候选；Android 仍有 VPN 内部 `host`。 |
| 4 · TLS | 核对节点证书校验已启用，对目标服务发起正常验证证书的 HTTPS 请求。 | 代理外层和应用到服务的内层各有一次握手；验证证书检查的是对端身份。 | 辅助探针通过证书校验；该结果不能验证 TLS 指纹与某浏览器相同。 |
| 5 · DNS | 用新随机域名对照权威记录与核心的 DoH 出站链，节点启动查询单列。 | 前者告诉你谁来查过，后者解释你的设备怎样到达它。 | 受测业务查询只见 Cloudflare；节点域名有加密直连例外。 |
| 6 · IPv6 | 对照 AAAA 答复、路由和字面 IPv6 地址的可达性。 | 应用知道一个 IPv6 地址时，可以跳过 DNS。 | Windows 保留系统 IPv6 栈；本次手机样本无 IPv6 路由。 |
| 7 · Claude 分流 | 在实际应用产生流量时检查 API、CDN 等连接的进程、匹配规则及最终节点。 | 域名规则按目的地匹配，包名规则按连接拥有者匹配。 | Windows 观察到域名规则；Android 观察到 Claude、GBox、microG 的包名规则。 |
| 8 · 规则优先级 | 在生成配置中找到专用规则和国内规则，再打开一个国内站点。 | 列表中更早的通用规则可能让后面的专用规则失去机会。 | 专用规则在前，受测国内 HTTPS 命中 `DIRECT`。 |
| 9 · 连接复用 | 用同一个测试客户端连续请求，插入一段空闲，统计新建 TLS 连接数。 | 多次请求共用一条连接才是复用；仅打开 TCP 保活开关不能证明这一点。 | 辅助探针 5 次请求共用 1 条 TLS 连接，中间空闲 10 秒。`401` 只表示未认证。 |
| 10 · 并发与限额 | 在正常获授权的工作负载中记录请求量、`429` 和 `retry-after`（如返回）。 | 服务端按账户限额处理请求；网络延迟和出口测试推不出并发阈值。 | 未做账户压力测试或付费模型调用，没有可据此给出的安全并发数。[官方限额](https://platform.claude.com/docs/en/api/rate-limits) |
| 11 · 路径与时延 | 重复采样，分别记代理链、响应耗时和边缘标记。 | 一次请求的总耗时混合了网络和服务处理时间；边缘标记只定位可见的接入点。 | 只有短时样本，不能还原完整路由或 Claude 后端位置。 |

这 11 项沿用[第三方清单](https://ip.net.coffee/claude/claudecode.html)作为目录。判断“出口”的一个实用对照：若 ipify 在连接页命中普通组，它测到的就只是普通组。需要单测专用组时，可在**测试副本**里临时把 `DOMAIN,api.ipify.org,Claude-Fixed` 放在通用规则前，发起新请求并检查链路，测完撤销。回到 Claude 时仍要核对它自己的真实连接。

## DNS：同时查解析器和出站链

1. 在待测设备的实际浏览器新开一轮 DNS 检测，让测试站生成**新的随机域名**。新名字绕开上一次的缓存。权威服务器记录的是前来查询的递归解析器；其出口 IP 可能和你配置的 DoH 入口 IP 不同。
2. 在同一台 Windows 上，对测试服务能在报告里识别的两个新域名执行下面的命令。第一条让系统按常规方式查询（通常先用 UDP），第二条强制 TCP。以核心记录确认实际传输；TXT 即使没有记录，也要看权威侧是否收到了这个名字。Android 应将浏览器发起的查询和从核心发起的查询分别记录。

```powershell
$udpName = Read-Host 'Fresh test domain 1'
Resolve-DnsName -Name $udpName -Type TXT -DnsOnly
$tcpName = Read-Host 'Fresh test domain 2'
Resolve-DnsName -Name $tcpName -Type TXT -DnsOnly -TcpOnly
```

3. 对照同一轮核心记录。业务 DoH 应经 `Claude-Fixed`；节点域名启动解析的 AliDNS 直连另记。若出现意外运营商解析器，先查 53 劫持、`direct-nameserver`、备用上游、系统 DNS 追加，再查浏览器自带 DoH 或 Android 私人 DNS。即使意外直连的是同一公共解析器，也要靠实际连接链才能识别。
4. 只有合成 A 地址或全部超时，记录为**证据不足**。要检查整台设备是否还从物理网卡发出意外 DNS 流量，可另做获授权的抓包，按进程、目的地和时间与测试对应；加密 DNS 的报文内容在网卡上通常不可读。

历史样本中，Windows 的浏览器随机域名和 6 次 UDP、6 次 TCP TXT 查询，以及 Android 浏览器的 12 个随机域名、核心的 12 次 TXT 查询，都只观察到 Cloudflare。两端均核对了核心 DoH 链。bash.ws 同时显示过 `DNS may be leaking.`；保留这个提示，按具体解析器和出站链判读。[Microsoft 命令说明](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname) · [DoH 协议](https://www.rfc-editor.org/rfc/rfc8484.html)

## WebRTC：读候选地址，而非只看“通过”

刷新页面，国内和海外目标分别收集，等到收集完成或记下超时。首次仅测 Google STUN 时曾漏掉国内直出。若站点无法选择目标，可用 [WebRTC 项目的 Trickle ICE 页面](https://webrtc.github.io/samples/src/content/peerconnection/trickle-ice/) 每轮只保留一个要测的 STUN 服务器。逐项记录类型，并在本机与预期代理出口、已知直连地址比较。

| 页面观察 | 如何解释 |
| --- | --- |
| `host` | 来自本地接口，可能是局域网、VPN、可公网路由的地址，也可能以 mDNS 名显示。看地址类别和接口，再判断暴露范围。 |
| `srflx` | STUN 返回它看到的映射地址。若与真实直出公网一致，这一轮存在泄露；若与代理出口一致，网页仍然得到了一个公网地址。 |
| `relay` | TURN 中继分配的地址。它可以和其他候选同时出现，应查看完整列表。 |
| 没有候选 | 分开记录 API 不可用、收集完成后为空、请求超时。超时也可能是目标不可达，不能单独作为通过依据。 |

若目标是“网页获取不到任何地址”，VPN 内部 `host` 也要记录为**仍可见**。Windows 受测主页面和 iframe 未给出候选，当轮未观察到核心后备拦截；Android 的 10 目标样本未见公网候选，保留了内部 `host`，并观察到核心拒绝 Chrome UDP。[ICE 候选的定义](https://www.rfc-editor.org/rfc/rfc8445.html)

## 时区：在实际使用的进程中读值

在普通网页的 DevTools Console 中运行以下只读代码。`Intl` 给出当前环境的时区名；取一月和七月分别算偏移，可区分全年固定的东八区和采用夏令时的洛杉矶。

```javascript
const year = new Date().getUTCFullYear();
({
  zone: Intl.DateTimeFormat().resolvedOptions().timeZone,
  january: new Date(Date.UTC(year, 0, 15, 12)).getTimezoneOffset(),
  july: new Date(Date.UTC(year, 6, 15, 12)).getTimezoneOffset()
})
```

`getTimezoneOffset()` 是 UTC 减当地时间的分钟数，因此东八区 `Asia/Shanghai` 为 `-480 / -480`，洛杉矶 `America/Los_Angeles` 通常为 `480 / 420`。网页控制台只代表选中的脚本环境。核对 CLI 请按 [Windows 的进程检查](WINDOWS.md)，手机则另查系统设置。[偏移量的符号和夏令时](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date/getTimezoneOffset)

## 检测网站与未测场景

| 网站 | 用途 |
| --- | --- |
| [ipify](https://api.ipify.org?format=json) · [Cloudflare trace](https://1.1.1.1/cdn-cgi/trace) | 查看这条 HTTPS 连接的公网来源；trace 还显示接入的 Cloudflare 边缘。 |
| [IPPure DNS](https://ippure.com/en/DNS-Leak-Detect.html) · [Net.Coffee DNS](https://ip.net.coffee/dns/) · [bash.ws](https://bash.ws/dnsleak) | 比对当轮出现的递归解析器，配合核心日志解释结果。 |
| [IPPure WebRTC](https://ippure.com/en/Browser-WebRTC-Leak-Detect.html) · [Net.Coffee WebRTC](https://ip.net.coffee/webrtc/) | 对照页面暴露的候选类型和地址类别。 |
| [哔哩哔哩](https://www.bilibili.com/) | 用真实国内 HTTPS 验证 `DIRECT`，由连接日志确认。 |

检测服务能看到发起测试的公网来源。分享时用“预期代理出口”“直出公网”“VPN 内部地址”等类别代替原值。此前未覆盖全机抓包、所有应用自带 DNS、重启、网络切换、长时间熄屏、VPN 意外退出、数小时模型流式会话或微信登录、语音和视频的完整流程。
