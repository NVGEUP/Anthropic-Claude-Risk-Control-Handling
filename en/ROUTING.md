# Shared routing and DNS

[Guide home](README.md) · [Windows](WINDOWS.md) · [Android](ANDROID.md) · [Verification](VERIFICATION.md)

Prepare the script at the end of this page in a copy of the active subscription. The core makes two decisions: **where to send an application connection** and **where to send the DNS lookup it needs first**. Configure the routes, then the resolvers, and follow both connections during a single visit.

## 1. Give Claude an outbound route you can verify independently

1. Set `sourceName` to the **exact name** of the source node in your subscription. The example searches `config.proxies` for VLESS over TCP with TLS and certificate verification enabled. A subscription that exposes nodes only through `proxy-providers` needs a different lookup. Keep node credentials in your local subscription.
2. The script copies a qualifying node to `Claude-Fixed-VLESS` and sets `ip-version: ipv4`. The copy leaves the ordinary groups free to use their original node. This field selects the address family for dialing the **proxy server**. On Android set `android = true` to retain UDP forwarding for applications; the transport to the node can still be TCP/TLS.
3. Create a `select` group called `Claude-Fixed` with exactly one member. Changing an ordinary selection or running a latency test then cannot switch Claude's group to another node. If no source passes the checks, the script gives the group only `REJECT` so matching connections fail explicitly. A provider can still change the public egress behind a stable node name.
4. Check that no unrelated node or group uses these two reserved names. If one does, rename the example consistently. The script rebuilds its entries by name each time the subscription refreshes. [Mihomo proxy fields](https://wiki.metacubex.one/en/config/proxies/)

## 2. Match exceptions before the original rules

Mihomo works down the list until it finds a rule that can handle the connection. A broad `MATCH` or domestic direct rule near the top could claim traffic before the dedicated rule sees it. Put rows 1–3 ahead of the original list.

| Order | Match and action | Reason and scope |
| --- | --- | --- |
| 1 | Chrome process **and** UDP → `REJECT` | Both conditions must match. Domestic STUN is covered before regional rules run. Ordinary TCP web traffic continues down the list. |
| 2, Android only | Claude and the GBox / microG packages actually used → dedicated group | Matching the connection owner covers new domains, as well as traffic the package generates for other purposes. The node must support the required UDP traffic, or UDP can continue to later rules. |
| 3 | `DOMAIN-SUFFIX`, `DOMAIN`, and API inbound `IP-CIDR` / `IP-CIDR6` → dedicated group | A suffix includes subdomains; an exact rule covers one host. IP rules cover known destination addresses. `no-resolve` avoids a DNS lookup solely to test a network range. Rules for shared services such as GitHub and npm affect other apps too. |
| 4 | Original rules, including domestic `DIRECT` and the final catch-all | Preserve the original order and ordinary selections. Check the matched rule and outbound route for a known domestic site to confirm direct access. |

The example retains the two API ranges in the recorded configuration. Check the official **Inbound IP addresses** at deployment time. The list checked for this edition also includes `153.61.224.0/22`. To cover it, insert `IP-CIDR,153.61.224.0/22,Claude-Fixed,no-resolve` before the original rules. The official outbound list describes connections initiated by the service for a different purpose. [Mihomo routing rules](https://wiki.metacubex.one/en/config/rules/) · [Claude API addresses](https://platform.claude.com/docs/en/api/ip-addresses)

## 3. Choose the DNS resolver and the route to it

DoH encrypts a query between the device and the resolver using HTTPS. **Which resolver learns the name** and **which exit carries the connection to that resolver** are separate choices. This configuration makes both explicit.

| What needs resolving | Configuration | Reason |
| --- | --- | --- |
| Application domains, including sites later reached directly | `nameserver`, the service `nameserver-policy`, and `direct-nameserver` use the two Cloudflare DoH endpoints with `#Claude-Fixed` appended. | The group suffix selects a route to the resolver. HTTPS alone selects encryption. Setting the resolver for direct destinations lets domestic HTTPS stay direct while application DNS goes through the proxy. |
| The proxy server itself | `proxy-server-nameserver` points directly to AliDNS DoH at a literal IP in this example. | If the node uses a hostname, its IP is needed before the tunnel exists. Sending that lookup through the same unbuilt tunnel creates a loop. AliDNS sees this bootstrap query. |
| DNS server names and alternative resolvers | `default-nameserver` resolves resolver hostnames; the listed upstreams already use literal IPs. Clear `fallback` and, on Android, do not append system DNS. | An alternate resolver could change the route for ordinary names. If the original configuration needs split DNS or internal domains, preserve those policies in a copy before applying this example. |

With `fake-ip` the core hands the app a synthetic address so it can associate a later connection with the requested name. Seeing that address proves local mapping only. Check the authoritative DNS test and the core's outbound DoH chain together. If the group is unavailable and no cached answer exists, even a domestic site with a `DIRECT` HTTPS rule can fail at name resolution. [Mihomo DNS](https://wiki.metacubex.one/en/config/dns/) · [Cloudflare DoH](https://developers.cloudflare.com/1.1.1.1/encryption/dns-over-https/)

## 4. Run three checks on a copy

1. Validate the **merged** configuration, for example with `mihomo -t -f candidate.yaml`. Use the client's resource directory if the subscription needs rule sets. A successful parse shows the core can load the file; live connections still need testing.
2. Run the script with the correct source name, then with a missing source in the copy. Expect a single node in the first case and only `REJECT` in the second. Apply it again and check that each rule still appears once. This exercises node renaming and repeated subscription refreshes.
3. Restore the correct source name and apply. Trigger a new Claude request and a domestic HTTPS request, then inspect a DoH upstream connection. Expect the dedicated group, `DIRECT`, and a resolver connection through the dedicated group, respectively. Recheck after changing the ordinary group; an already-open connection can retain its previous path.

### Subscription script

Set `sourceName` and `android` before use. The final `disable-keep-alive: false` setting concerns TCP keepalive. Whether an application reuses an HTTP/TLS connection needs a separate measurement.

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
