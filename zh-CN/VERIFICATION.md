# 11 项验收与检测网站

[手册首页](README.md) · [Windows](WINDOWS.md) · [Android](ANDROID.md) · [共用分流与 DNS](ROUTING.md)

先确认 VPN 或 TUN 正在运行、配置为 Rule 模式，再采样。每项分别记录**操作、观察、结论**。复测时在本机比较具体地址，分享记录时使用“预期出口”“本地公网”“私网 host”这些类别。

## 原清单逐项核对

| # | 检查项 | 如何检验 | 历史结果与边界 |
| --- | --- | --- | --- |
| 1 | 出口 IP | 在目标客户端做 HTTPS 出口采样，比较多个服务的结果 | 短时样本一致，运营商标签不能证明 IP 独享或历史信誉 |
| 2 | 出口持续性 | 切换普通组，检查专用组成员，再重复出口采样 | 专用组只有一个节点。上游服务商仍可更换出口 |
| 3 | TCP、UDP、WebRTC | 核对代理外层 TCP/TLS，分别在国内与海外 STUN 目标收集 ICE 候选 | Windows 加固后受测页面为空，Android 保留 VPN 内部 host |
| 4 | TLS | 检查证书验证开关，向受测服务发起校验证书的 HTTPS 请求 | 辅助探针通过证书校验。代理外层与应用内层有各自的握手 |
| 5 | DNS | 对随机子域名做浏览器和 UDP/TCP TXT 查询，交叉查看上游连接 | 受测业务解析进入预期 DoH，节点启动解析有直连例外 |
| 6 | IPv6 | 查 DNS 策略、VPN 地址族和字面 IPv6 地址可达性 | Windows 保留系统 IPv6，手机样本无 IPv6 路由 |
| 7 | Claude 专用分流 | 查看 API、CDN 及关联进程连接的规则与最终出站 | Windows 域名规则和手机三条应用规则均有相应证据 |
| 8 | 通用规则优先级 | 查看合并后的顺序，实际打开国内站点 | 专用规则在前，受测国内 HTTPS 仍命中 DIRECT |
| 9 | 连接复用 | 多次连续发送无凭据请求，核对底层 TLS 连接数并插入短时空闲 | 辅助探针 5 次请求复用同一 TLS 通道，含 10 秒空闲。401 表示未认证 |
| 10 | 并发与限额 | 需要有授权的真实负载与服务侧限额记录 | 未做账户压力测试或付费模型调用，无法据此给出安全并发数 |
| 11 | 路径与时延 | 同时记代理链、响应时延与可见的边缘标记 | 仅有短时采样。边缘位置无法确定 Claude 后端位置或完整路由 |

这 11 项沿用原第三方清单作为检查目录。服务的账户风控、设备识别和模型长期稳定性不属于这些探针可验证的结果。[原始清单](https://ip.net.coffee/claude/claudecode.html)

## DNS 如何验收

1. 在待测设备的实际浏览器启动一次随机域名 DNS 检测。每一轮使用新的测试编号。读取权威服务观察到的**递归解析器**。
2. 在同一设备上补测常规 DNS。Windows 可用 `Resolve-DnsName <新测试域名> -Type TXT -DnsOnly`，用另一随机名称加 `-TcpOnly` 检查 TCP 53。UDP 路径要结合核心记录核对传输方式。Android 历史验证分别从手机浏览器和手机核心发出查询。
3. 打开核心连接记录，找到 Cloudflare 的 DoH 会话，检查其上游链是否经过专用组。节点启动解析另行记录。
4. 若看到运营商解析器或业务查询命中意外的 DIRECT 上游，应检查 TUN 的 53 接管、`direct-nameserver`、fallback、系统 DNS 附加和应用自带的 DoH/DoT。
5. 若只有 `fake-ip` 的 A 记录，或全部测试请求超时，记录为“证据不足”。可在获授权的物理接口抓包中进一步识别直接外发的流量。

历史上，Windows 的浏览器随机查询及分别 6 次 UDP、6 次 TCP 的 TXT 样本只见 Cloudflare。Android 浏览器的 12 个随机域名及手机核心的 12 个 TXT 样本也只见 Cloudflare。两端还核对了核心 DoH 链。bash.ws 同时给过 `DNS may be leaking.` 的通用提示，所以保留该提示并结合具体记录解释。

## WebRTC 如何验收

| 观察 | 记录与判断 |
| --- | --- |
| `host` | 记录为本地接口、VPN 私网或 mDNS 候选，区分真实局域网与 VPN 内部地址 |
| `srflx` | 这是 STUN 获得的映射地址。在本机比较其是否为真实直出公网或预期代理出口 |
| `relay` | 表示 TURN 中继分配的地址，单独记录 |
| 候选为空、API 不可用或超时 | 分别记录。检查多个 STUN 目标与核心命中情况 |

打开多目标 WebRTC 页面并重新测试。首次只测 Google STUN 时曾漏掉国内目标的直出。加固后 Windows 的普通页面、iframe 和兼容接口样本均为空。Android 的 10 目标样本没有公网候选，仍有 VPN 内部 host，且核心确实记录到 Chrome UDP 拒绝。两端都应在浏览器更新或网络切换后重新测试。

## 时区如何验收

在普通网页的 DevTools Console 中运行以下只读代码。主页面的结果只代表当前执行环境。

```javascript
const year = new Date().getUTCFullYear();
({
  zone: Intl.DateTimeFormat().resolvedOptions().timeZone,
  january: new Date(Date.UTC(year, 0, 15, 12)).getTimezoneOffset(),
  july: new Date(Date.UTC(year, 6, 15, 12)).getTimezoneOffset()
})
```

东八区结果为 `Asia/Shanghai`、`-480 / -480`。洛杉矶为 `America/Los_Angeles`，通常是 `480 / 420`。历史复测还覆盖了跨源 iframe、Web Worker、Shared Worker 和 Service Worker。移除时区扩展后五项均已回到东八区。Chrome 的长期稳定性需要另测。

## 检测入口

打开外部检测站会向该服务发送测试请求，其服务器可以看到连接的公网地址。原始结果应在本机查看，复测记录只保留脱敏结论。

| 网站 | 本次用途 |
| --- | --- |
| [IPPure WebRTC](https://ippure.com/en/Browser-WebRTC-Leak-Detect.html) · [Net.Coffee WebRTC](https://ip.net.coffee/webrtc/) | 多 STUN 交叉验证 |
| [IPPure DNS](https://ippure.com/en/DNS-Leak-Detect.html) · [Net.Coffee DNS](https://ip.net.coffee/dns/) · [bash.ws](https://bash.ws/dnsleak) | 对照递归解析器与随机域名的权威记录 |
| [ipify](https://api.ipify.org?format=json) · [Cloudflare trace](https://1.1.1.1/cdn-cgi/trace) | HTTPS 出口样本与 Cloudflare 边缘信息 |
| [哔哩哔哩](https://www.bilibili.com/) | 配合连接日志验证国内普通 HTTPS 直连 |

## 哪些情况还要另测

未完成全机物理接口抓包、所有应用自带 DNS、重启、跨网络切换、长时间熄屏、VPN 异常退出、数小时模型流式会话，以及微信等应用的完整登录、语音、视频流程。Android 手动关闭 VPN 后恢复普通联网是既定设置。国内 DIRECT 网站能看到其直接连接的来源。
