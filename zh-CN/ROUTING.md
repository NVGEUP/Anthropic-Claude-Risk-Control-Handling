# 共用分流与 DNS

[手册首页](README.md) · [Windows](WINDOWS.md) · [Android](ANDROID.md)

这一步把 Claude 的连接绑定到单一出口，同时保留原来的国内规则和普通策略组选择。先在备份副本中操作。`Claude-Fixed` 和 `Claude-Fixed-VLESS` 是为本文预留的策略组和节点名，如与自己的既有配置冲突，需在所有相关位置统一换名。

## 1. 生成专用策略组

1. 在当前订阅找到计划使用的源节点。历史配置接受 `type: vless`、TLS 开启、TCP 传输且启用证书校验的节点。
2. 在持久覆写中复制这个节点，给副本命名为 `Claude-Fixed-VLESS`，设 `ip-version: ipv4`。Android 副本额外保留 `udp: true`，供应用流量转发。
3. 建 `select` 类型的 `Claude-Fixed` 组。源节点合格时，组内仅包含此副本。源节点消失或条件不符时，覆写把组成员设为 `REJECT`。
4. 保留原订阅其他组的成员、选择和顺序。切换普通节点后，再查看专用组是否仍只有一个成员。

以下是**代理订阅的配置示例**，可用来核对持久覆写的行为。先在本机替换节点名，Windows 保留 `android = false`，Android 改为 `true`。示例覆盖本文涉及的 DNS 策略，原配置如果依赖内网域名或分离解析，先在副本上核对并保留所需的内部 DNS 策略。

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

示例保留了历史实测的两条 API 入站 CIDR。整理手册时，官方清单还列有 `153.61.224.0/22`。部署时核对官方 **Inbound IP addresses**，按需加在原规则之前，再验证相应连接。GitHub、npm、Google Storage 等共享服务的域名规则对同一核心中的其他应用也生效。[Claude API 入站范围](https://platform.claude.com/docs/en/api/ip-addresses)

## 2. 检查匹配顺序

| 顺序 | 匹配 | 动作 |
| --- | --- | --- |
| 1 | 对应平台的 Chrome UDP | `REJECT` |
| 2，仅 Android | Claude、GBox、microG 三个包名 | `Claude-Fixed` |
| 3 | 上述域名、精确主机名和公开 API 入站范围 | `Claude-Fixed` |
| 4 | 原订阅的国内网站、普通策略组和兜底规则 | 沿用原规则 |

如果国内站点本来就有 `DIRECT` 规则，应能在前置规则之后找到它。在连接页用受测站点核实匹配结果。保持 Rule 模式，重启核心或切换回该订阅后再检查一次。[Mihomo 路由规则](https://wiki.metacubex.one/config/rules/)

## 3. 分开看 DNS 的两条路径

| 查询用途 | 路径 | 为什么这样设置 |
| --- | --- | --- |
| 普通业务域名，包括随后直连的国内网站 | Cloudflare DoH 经过 `Claude-Fixed` | 避免业务域名回到系统或本地运营商解析器 |
| 代理节点的主机名 | AliDNS DoH 加密直连 | 建立代理隧道前先获得节点地址，避免循环依赖 |

`#Claude-Fixed` 指定上游连接使用的代理组。`fake-ip` 的合成 A 记录只能验证本地接管。要确认查询真正经过指定上游，需要随机域名的权威结果和核心 DoH 连接链。若专用组变为 `REJECT`，依赖该组的普通 DNS 也可能失败，国内站点即便保有 `DIRECT` 规则仍可能打不开。[Mihomo DNS](https://wiki.metacubex.one/config/dns/)

## 4. 在副本上确认再应用

- 用本机 Mihomo 的配置检查功能检验合并后的配置，例如 `mihomo -t -f candidate.yaml`。如原订阅依赖规则集，使用该客户端的工作目录和已有资源。
- 在副本中用匹配与不匹配的源节点名各试一次。确认分别得到专用节点与 `REJECT`，并检查重复应用后规则只有一份。
- 在日常配置中恢复正确节点名，应用订阅，再依次检查普通节点选择、国内 HTTPS、Claude 连接和上游 DNS。Windows 同时核对 TUN，Android 同时核对 VPN 开关及应用级规则。
