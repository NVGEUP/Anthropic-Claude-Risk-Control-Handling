# 共用分流与 DNS

[手册首页](README.md) · [Windows](WINDOWS.md) · [Android](ANDROID.md) · [验收](VERIFICATION.md)

在当前订阅的副本中准备页末脚本。核心作两个决定：**业务连接从哪出去**，以及**为了拿到地址，DNS 查询从哪出去**。先安排业务规则，再安排 DNS；最后用同一次访问的两条连接核对结果。

## 1. 给 Claude 一个可以单独验证的出口

1. 把 `sourceName` 改成订阅中源节点的**完整名称**。示例从 `config.proxies` 查找一个 VLESS、TCP、开启 TLS 且未跳过证书校验的节点；订阅若只通过 `proxy-providers` 提供节点，就不能直接套用这个查找方式。不要把含凭据的节点粘到公开文档。
2. 脚本复制合格节点，命名为 `Claude-Fixed-VLESS`，并设 `ip-version: ipv4`。复制可保留原节点供普通组使用；这个字段约束核心连接**代理服务器**时使用的地址族。Android 设 `android = true`，让该副本支持应用的 UDP 转发；节点外层仍可用 TCP/TLS 传输。
3. 建立 `select` 组 `Claude-Fixed`，只给它一个成员。这样切换普通节点或进行延迟选优不会改到 Claude 这组。若脚本找不到符合条件的源节点，组里只放 `REJECT`，让匹配连接明确失败。即使同一个节点名始终有效，服务商也可能改变它背后的公网出口。
4. 确认预留的组名、节点名没有被其他用途占用；如有冲突，统一换名。脚本会按这两个名字重建自己的条目，才能在订阅每次刷新时重复执行。[Mihomo 节点通用字段](https://wiki.metacubex.one/config/proxies/)

## 2. 先匹配例外，最后交给原订阅

Mihomo 从上往下查找可以处理这条连接的规则。若通用 `MATCH` 或地区直连规则排在前面，专用规则可能根本不会被检查。因此把下表的 1–3 排在原规则之前。

| 顺序 | 规则 | 原理与范围 |
| --- | --- | --- |
| 1 | Chrome 进程 **并且** UDP → `REJECT` | 同时满足两个条件才拦截。国内 STUN 也会先遇到它；普通 TCP 网页可以继续向下匹配。 |
| 2，仅 Android | Claude、实际使用的 GBox / microG 包名 → 专用组 | 按连接拥有者匹配，能覆盖新出现的域名，也会包含该包为其他用途发出的连接。目标节点须能承载所需的 UDP，否则 UDP 可能继续匹配后续规则。 |
| 3 | `DOMAIN-SUFFIX`、`DOMAIN`、API 入站 `IP-CIDR` / `IP-CIDR6` → 专用组 | 后缀包括子域，精确规则只匹配指定主机。IP 规则处理已知目的地址；`no-resolve` 避免仅为了检查网段再发起 DNS。GitHub、npm 等共享域名在这里会影响其他应用。 |
| 4 | 原订阅规则，含国内 `DIRECT` 和最终兜底 | 保留原顺序及普通组选择。用一个已知国内站点做对照，核对规则和出站，才能知道直连仍然生效。 |

示例保留实测时的两条 API 网段。部署时对照官方 **Inbound IP addresses**；本次核对的清单还列有 `153.61.224.0/22`。若需覆盖该段，在原规则前补 `IP-CIDR,153.61.224.0/22,Claude-Fixed,no-resolve`。官方的 outbound 表描述由服务端发起的连接，用途不同。[Mihomo 路由规则](https://wiki.metacubex.one/config/rules/) · [Claude API 地址清单](https://platform.claude.com/docs/en/api/ip-addresses)

## 3. 为 DNS 指定上游，也指定到上游的路

DoH 用 HTTPS 加密设备到解析器之间的查询。**哪个解析器收到域名**和**连接解析器时使用哪个出口**是两件事。在本例里由以下字段一起决定。

| 谁需要地址 | 怎么配置 | 原因 |
| --- | --- | --- |
| 普通业务，包括随后直连的网站 | `nameserver`、业务的 `nameserver-policy` 和 `direct-nameserver` 都使用两个 Cloudflare DoH 地址，地址末尾带 `#Claude-Fixed`。 | `#组名` 指定上游连接走哪组；单写 HTTPS 只指定加密方式。单独设置直连目标的解析器，让国内网站可直连而业务 DNS 仍经代理。 |
| 代理节点自身 | `proxy-server-nameserver` 指向以 IP 写出的 AliDNS DoH，按本例直接连接。 | 节点若用域名，建隧道前必须先知道其 IP。要求这个查询也走尚未建好的同一隧道会形成循环。AliDNS 能看到这次节点查询。 |
| DNS 服务器名称及其他退路 | `default-nameserver` 用于解析上游服务器的主机名；此处列出的上游均为 IP。清空 `fallback`，手机也不追加系统 DNS。 | 这样不会因一个备用解析器而改变普通域名的预期路径。若原配置有内网或分域 DNS，先在副本里保留所需策略，再应用示例。 |

`fake-ip` 先给应用一个合成地址，使核心能把后续连接对应回域名。看到该地址只能说明本机进行了映射。验收要同时看权威 DNS 测试记录和核心中 DoH 的出站链。如果专用组不可用且没有缓存，连原本走 `DIRECT` 的国内站点也可能因无法解析而打不开。[Mihomo DNS](https://wiki.metacubex.one/config/dns/) · [Cloudflare DoH](https://developers.cloudflare.com/1.1.1.1/encryption/dns-over-https/)

## 4. 在副本上做三次检查

1. 验证**合并后**的配置，例如 `mihomo -t -f candidate.yaml`。订阅使用规则集时，指定客户端已有资源的工作目录。语法通过说明核心能读取配置，不能代替运行中的连接记录。
2. 在副本中分别用正确节点名、一个不存在的节点名运行脚本：专用组应分别为一个节点、仅 `REJECT`。再重复应用，确认同一条规则没有增加第二份。这模拟了订阅更新后节点改名及覆写重复执行。
3. 恢复正确源节点并重新应用。各发起一条新的 Claude 请求和国内 HTTPS 请求，再看一条 DoH 上游记录。依次核对专用组、`DIRECT`、经专用组连接解析器。切换普通组后重试，已有的长连接可能仍使用旧路径。

### 可复制的订阅脚本

先改 `sourceName` 和 `android`。脚本末尾的 `disable-keep-alive: false` 针对 TCP 保活；应用能否复用同一个 HTTP/TLS 连接还要独立测量。

```javascript
function main(config) {
  const android = false;
  const sourceName = "REPLACE_WITH_YOUR_NODE_NAME";
  const node = "Claude-Fixed-VLESS";
  const group = "Claude-Fixed";
  const proxies = config.proxies || [];
  const source = proxies.find(p => p.name === sourceName);
  const usable = source && source.type === "vless" && source.tls === true
    && (!source.network || source.network === "tcp")
    && source["skip-cert-verify"] !== true;

  config.proxies = proxies.filter(p => p.name !== node);
  if (usable) {
    const copy = JSON.parse(JSON.stringify(source));
    copy.name = node;
    copy["ip-version"] = "ipv4";
    if (android) copy.udp = true;
    config.proxies.push(copy);
  }
  config["proxy-groups"] = [
    {name: group, type: "select", proxies: usable ? [node] : ["REJECT"]},
    ...(config["proxy-groups"] || []).filter(g => g.name !== group)
  ];

  const suffixes = [
    "anthropic.com", "claude.ai", "claude.com", "claudeusercontent.com",
    "clau.de", "claudemcpclient.com", "claudemcpcontent.com", "statsigapi.net",
    "statsigcdn.com", "sentry.io", "sentry.dev", "intercom.io", "intercomcdn.com",
    "github.com", "githubusercontent.com", "googlesource.com"
  ];
  const exact = [
    "anthropic.com.cdn.cloudflare.net", "servd-anthropic-website.b-cdn.net",
    "anthropic.auth0.com", "anthropic-com.ghost.io", "http-intake.logs.us5.datadoghq.com",
    "browser-intake-us5-datadoghq.com", "cdn.usefathom.com", "storage.googleapis.com",
    "registry.npmjs.org"
  ];
  const browser = android ? "com.android.chrome" : "chrome.exe";
  const browserUdp = "AND,((PROCESS-NAME," + browser + "),(NETWORK,UDP)),REJECT";
  const apps = android ? ["com.anthropic.claude", "com.gbox.android", "com.google.android.gms"] : [];
  const ownRules = [
    browserUdp,
    ...apps.map(p => "PROCESS-NAME," + p + "," + group),
    ...suffixes.map(d => "DOMAIN-SUFFIX," + d + "," + group),
    ...exact.map(d => "DOMAIN," + d + "," + group),
    "IP-CIDR,160.79.104.0/23," + group + ",no-resolve",
    "IP-CIDR6,2607:6bc0::/48," + group + ",no-resolve"
  ];
  const original = (config.rules || []).filter(r =>
    r !== browserUdp && !String(r).split(",").some(part => part.trim() === group)
  );
  config.rules = [...ownRules, ...original];

  const upstream = [
    "https://1.1.1.1/dns-query#" + group,
    "https://1.0.0.1/dns-query#" + group
  ];
  const policy = {};
  for (const domain of suffixes) policy["+." + domain] = upstream;
  for (const domain of exact) policy[domain] = upstream;
  config.dns = {
    ...(config.dns || {}),
    enable: true, ipv6: false, "enhanced-mode": "fake-ip",
    "prefer-h3": false, "respect-rules": false,
    "default-nameserver": ["https://223.5.5.5/dns-query"],
    "proxy-server-nameserver": ["https://223.5.5.5/dns-query"],
    "proxy-server-nameserver-policy": {},
    nameserver: upstream, "direct-nameserver": upstream,
    "direct-nameserver-follow-policy": false,
    "nameserver-policy": policy, fallback: []
  };
  config["disable-keep-alive"] = false;
  return config;
}
```
