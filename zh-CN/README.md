# Anthropic/Claude Risk Control Handling

**简体中文使用手册** · [选择语言](../README.md)

这套配置让 Claude 的连接走指定代理，国内网站继续按原规则直连。先从自己的平台开始。进入共用配置一节时，再打开分流章节。

| 入口 | 读完可以做什么 |
| --- | --- |
| [Windows](WINDOWS.md) | 让 Clash Verge Rev 接管流量，检查 Claude Code 的代理与时区，限制 Chrome 交给网页的 WebRTC 地址。 |
| [Android](ANDROID.md) | 找出实际联网的软件包，让 FlClash 接管它，再检查 VPN 开启和关闭时的区别。 |
| [分流与 DNS](ROUTING.md) | 建立单节点策略组，安排规则优先级，分别指定业务域名和代理节点的解析路径。 |
| [11 项验收与检测网站](VERIFICATION.md) | 对照真实连接、DNS 记录和 WebRTC 候选，判断哪些结果有证据，哪些还需要复测。 |

### 用一次国内访问理解分流

打开哔哩哔哩时，设备先要知道域名对应哪个地址，再连接网站。在本配置中，**查询域名**由 Mihomo 通过专用代理向 Cloudflare 发出；**访问网站**这条 HTTPS 连接则按国内规则走 `DIRECT`。所以两条记录可以同时正确。网站能看到直连的公网来源；DNS 检测站观察到的是替你查询的递归解析器。

### 这次实际验证到了什么

| 项目 | Windows | Android |
| --- | --- | --- |
| 分流 | Claude Code 使用本机代理；受测 Claude 域名进入专用组，哔哩哔哩 HTTPS 命中 `DIRECT`。 | Claude、GBox、microG 的受测连接分别命中包名规则；哔哩哔哩 HTTPS 命中 `DIRECT`。 |
| DNS | 随机域名与 TXT 查询只观察到 Cloudflare；核心记录显示 DoH 经过专用组。 | 浏览器和核心样本得到相同结果。两端均保留代理节点域名的加密直连启动解析。 |
| WebRTC | 收紧设置后，受测页面未返回候选地址。 | 10 个 STUN 目标均未发现公网候选，仍可见 VPN 内部地址。 |
| 时区 | Windows 和 Chrome 为东八区，Claude Code 进程使用 `America/Los_Angeles`。 | 手机系统为东八区。 |
| 停止接管 | 未测试代理异常退出后的锁网。 | 手动关闭 VPN 后恢复普通联网；未测试意外断开。 |

实测使用 Clash Verge Rev 2.5.8、Mihomo 1.19.32 和 FlClash 0.8.99。换节点、换网络或更新客户端后，请重做与改动有关的检查。记录版本、匹配规则和结果即可；公网地址、订阅和完整日志留在本机。这些样本描述网络行为，无法判定服务端对账户的处理。
