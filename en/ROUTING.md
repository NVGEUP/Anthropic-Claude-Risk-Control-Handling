# Shared routing and DNS

[Guide home](README.md) · [Windows](WINDOWS.md) · [Android](ANDROID.md)

Bind matching Claude traffic to a single proxy node while preserving the original domestic rules and the ordinary proxy group's selection. Work on a backup copy first. This example reserves the names `Claude-Fixed` and `Claude-Fixed-VLESS`. If your configuration already uses either name, choose distinct names and update every reference consistently.

## 1. Build the dedicated group

1. Find the intended source node in the active subscription. The recorded configuration accepted a `type: vless` node using TCP, with TLS enabled and certificate verification active.
2. In the persistent override, clone that node as `Claude-Fixed-VLESS` and set `ip-version: ipv4` on the clone. The Android clone also sets `udp: true` to forward application UDP traffic.
3. Create a `select` group named `Claude-Fixed`. When the source is eligible, its clone is the group's only member. When the source disappears or fails validation, the override sets the group's only member to `REJECT`.
4. Preserve the other groups, their selections, and their order. After switching the ordinary group's node, check that the dedicated group still has a single member.

The following **proxy subscription override** shows the configuration logic. Substitute your source node name locally. Leave `android = false` for Windows and change it to `true` for Android. The example replaces the DNS policies it sets. If you rely on internal hostnames or split DNS, test in a copy and retain the internal policies you need.

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

The example reproduces the two API inbound CIDRs used in the recorded deployment. When the manual was prepared, the official list also included `153.61.224.0/22`. At deployment time, check **Inbound IP addresses**, add applicable rules ahead of the original rules, and verify the resulting traffic. Domain rules for shared services such as GitHub, npm, and Google Storage affect other applications using the same core. [Claude API inbound ranges](https://platform.claude.com/docs/en/api/ip-addresses)

## 2. Check the rule order

| Order | Match | Action |
| --- | --- | --- |
| 1 | Chrome UDP on the relevant platform | `REJECT` |
| 2, Android only | Claude, GBox, and microG package names | `Claude-Fixed` |
| 3 | Listed domain suffixes, exact hostnames, and public API inbound ranges | `Claude-Fixed` |
| 4 | Original domestic rules, ordinary proxy groups, and final catch-all | Retain each original action. |

Any existing `DIRECT` rule for a domestic site should still appear after the prepended rules. Open the site and inspect the connection that actually matched. Keep Rule mode selected and repeat the check after restarting the core or reactivating the subscription. [Mihomo routing rules](https://wiki.metacubex.one/config/rules/)

## 3. Distinguish the two DNS paths

| Query purpose | Route | Reason |
| --- | --- | --- |
| Ordinary application domains, including domestic sites whose HTTPS traffic goes direct | Cloudflare DoH through `Claude-Fixed` | Keep business-domain queries off the system or local ISP resolver. |
| The proxy server's hostname | Direct, encrypted AliDNS DoH | Resolve the node before the proxy tunnel exists, avoiding a circular dependency. |

`#Claude-Fixed` pins the DNS upstream connection to the proxy group. A synthetic A record in `fake-ip` mode shows local interception. To verify the upstream route, compare an authoritative random-domain report with the core's DoH connection chain. If the dedicated group resolves to `REJECT`, ordinary DNS through that group can also fail, leaving a domestic site unreachable even though its HTTPS rule remains `DIRECT`. [Mihomo DNS configuration](https://wiki.metacubex.one/config/dns/)

## 4. Validate a copy before applying it

- Check the merged configuration with the local Mihomo validator, for example `mihomo -t -f candidate.yaml`. If the subscription uses rule sets, use the client's working directory and its existing resources.
- Try a matching and a missing source-node name in the copy. Confirm that they produce the dedicated clone and `REJECT` respectively. Applying the override a second time should leave one copy of each rule.
- Restore the correct source-node name in the everyday configuration and apply the subscription. Check the ordinary node selection, domestic HTTPS, Claude traffic, and the DNS upstream in turn. On Windows, also check TUN. On Android, check the VPN switch and the package-level rules.
