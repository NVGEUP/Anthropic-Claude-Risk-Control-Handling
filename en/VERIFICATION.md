# 11 verification checks and test sites

[Guide home](README.md) · [Windows](WINDOWS.md) · [Android](ANDROID.md) · [Shared routing and DNS](ROUTING.md)

Confirm the active profile, **Rule** mode, and TUN/VPN capture. Open the core's **Connections** view. Make a **new request** from the browser or application you mean to test. Record the time, destination, matched rule, outbound route, and observation. An existing connection can retain its old path, and a browser tab is not a measurement of the Claude process.

## What to test and how to read it

| # | Test | Why it works | Recorded observation and boundary |
| --- | --- | --- | --- |
| 1 · Public egress | Query two HTTPS echo services on the same intended route. First confirm their matching group in the core. | An echo service sees the public source of that HTTPS connection. | Short samples agreed. ISP and city labels cannot establish exclusivity, reputation, or physical location. |
| 2 · Egress stability | Change the **ordinary group**, open a fresh connection, then recheck the dedicated group and egress. | If the two groups share the same selector, changing the ordinary node can move Claude too. | The dedicated group had one node. Its provider can still change the public IP behind it. |
| 3 · TCP, UDP, and WebRTC | Inspect the node's outer TCP/TLS transport and gather browser ICE candidates using domestic and overseas STUN targets separately. | A page can open UDP sockets independently of its HTTPS connection. | Sampled Windows pages returned no candidates. Android retained a VPN-internal `host` candidate. |
| 4 · TLS | Check that node certificate validation is enabled. Make a normal certificate-validated HTTPS request to the service. | The outer proxy and inner application each negotiate TLS. Certificate validation checks peer identity. | Auxiliary probes validated certificates. This cannot establish that the TLS fingerprint matches a browser's. |
| 5 · DNS | Use fresh random domains. Compare the authoritative report with the core's DoH outbound chain. Record bootstrap separately. | The report identifies who looked up the name. The chain shows how the device reached its resolver. | Application samples showed only Cloudflare. The node hostname used a direct encrypted exception. |
| 6 · IPv6 | Compare AAAA answers, routes, and reachability of a literal IPv6 address. | An app that already knows the address can skip DNS entirely. | Windows retained its IPv6 stack. The tested phone had no IPv6 route. |
| 7 · Dedicated Claude route | Trigger traffic in the actual app. For API and CDN connections, inspect the owner, matched rule, and final node. | A domain rule matches the destination. A package rule matches the connection owner. | Windows domain matches and the Android Claude, GBox, and microG package matches were observed. |
| 8 · Rule priority | Find the dedicated and domestic rules in the generated list, then open a domestic site. | An earlier catch-all or regional rule can claim the connection before a later rule sees it. | Dedicated rules came first. The sampled domestic HTTPS connection matched `DIRECT`. |
| 9 · Connection reuse | Send consecutive requests from the same test client, pause between some requests, and count new TLS connections. | Reuse means multiple requests shared a connection. A TCP keepalive setting alone cannot show this. | Five auxiliary requests used one TLS connection, including after a ten-second idle interval. `401` meant unauthenticated. |
| 10 · Concurrency and limits | During a normal authorized workload, record request volume, `429` responses, and `retry-after` when present. | Limits are applied by the service. Latency and public egress cannot determine an account's concurrency threshold. | No account stress test or paid model call was made. No safe concurrency figure follows. [Official rate limits](https://platform.claude.com/docs/en/api/rate-limits) |
| 11 · Path and latency | Take repeated samples. Log the proxy chain, elapsed time, and edge label separately. | Total time includes network and service work. An edge label identifies only the visible ingress point. | Short samples cannot reconstruct the complete route or locate the Claude backend. |

The eleven topics follow a [third-party checklist](https://ip.net.coffee/claude/claudecode.html). A practical control for egress tests: if ipify matches the ordinary group, that is the only group it measured. To isolate the dedicated group in a **test copy**, temporarily place `DOMAIN,api.ipify.org,Claude-Fixed` ahead of the general rules. Make a fresh request, check its chain, then remove the test rule. Check Claude's own connections separately.

## DNS: check both the resolver and the outbound connection

1. Start a new DNS test in the actual device browser. Let the service generate **fresh random domains** so an earlier cache entry cannot answer the query. The authoritative service sees the recursive resolver that asked for the name. Its outward-facing IP may differ from the DoH endpoint IP you configured.
2. On the same Windows device, run the commands below with two new test names the service can identify in its report. The first uses ordinary resolution, usually starting with UDP; the second forces TCP. Check the core for the actual transport. Even if no TXT record exists, check whether the authoritative service saw the query. On Android, distinguish browser-originated and core-originated probes.

```powershell
$udpName = Read-Host 'Fresh test domain 1'
Resolve-DnsName -Name $udpName -Type TXT -DnsOnly
$tcpName = Read-Host 'Fresh test domain 2'
Resolve-DnsName -Name $tcpName -Type TXT -DnsOnly -TcpOnly
```

3. Correlate the same test period with the core's connections. Application DoH should use `Claude-Fixed`. Record the direct AliDNS proxy-hostname lookup separately. If an unexpected ISP resolver appears, check port 53 interception, `direct-nameserver`, alternate upstreams, appended system DNS, then browser DoH or Android Private DNS. Even an unexpected direct connection to the *same* public resolver requires the outbound chain to distinguish it.
4. If the only evidence is a synthetic A record or every test timed out, record **inconclusive**. To check for additional direct DNS traffic from the device, an authorized physical-interface capture can correlate time and endpoints with processes and test activity. Encrypted query contents are normally unreadable on the physical interface.

Only Cloudflare appeared in the recorded Windows browser queries and the six UDP and six TCP TXT probes. The same held for twelve Android browser domains and twelve core TXT queries. Both platforms' core DoH routes were checked. bash.ws also displayed `DNS may be leaking.` Keep the warning with the actual resolvers and routes when interpreting it. [Microsoft command reference](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname) · [DoH protocol](https://www.rfc-editor.org/rfc/rfc8484.html)

## WebRTC: inspect the candidates a page can see

Reload the page and gather candidates separately for domestic and overseas targets. Wait for gathering to finish, or record the timeout. An early Google-only test missed a domestic direct route. If the test site does not let you select targets, use the [WebRTC project's Trickle ICE page](https://webrtc.github.io/samples/src/content/peerconnection/trickle-ice/) with one chosen STUN server per run. Compare each address privately with the expected proxy egress and known direct address.

| Page observation | How to interpret it |
| --- | --- |
| `host` | Comes from a local interface. It may be a LAN address, a VPN address, a publicly routable address, or an mDNS name. Check the address class and interface. |
| `srflx` | STUN reported a mapped address. A match with the direct public IP is a leak in this test. A match with proxy egress still means the page obtained a public address. |
| `relay` | TURN allocated a relay address. It can appear alongside other candidates, so read the complete list. |
| No candidates | Distinguish an unavailable API, completed gathering with an empty list, and a timeout. An unreachable test server can also cause a timeout. |

If the requirement is *no address visible to the page*, record even a VPN-internal `host` as **still visible**. The sampled Windows page and iframe yielded no candidates; the core's fallback rule recorded no hits in that run. Android's ten-target sample yielded no public candidate, retained an internal `host` address, and showed Chrome UDP rejections in the core. [ICE candidate definitions](https://www.rfc-editor.org/rfc/rfc8445.html)

## Time zone: read it in the relevant process

Run this read-only snippet in DevTools Console on an ordinary page. `Intl` reports the zone used by the current environment. January and July offsets distinguish a fixed UTC+8 setting from a zone such as Los Angeles that observes daylight saving time.

```javascript
const year = new Date().getUTCFullYear();
({
  zone: Intl.DateTimeFormat().resolvedOptions().timeZone,
  january: new Date(Date.UTC(year, 0, 15, 12)).getTimezoneOffset(),
  july: new Date(Date.UTC(year, 6, 15, 12)).getTimezoneOffset()
})
```

`getTimezoneOffset()` returns UTC minus local time in minutes. `Asia/Shanghai` therefore gives `-480 / -480`. `America/Los_Angeles` normally gives `480 / 420`. The console represents the selected script context. For Claude Code use the [Windows process check](WINDOWS.md); for the phone check the system setting separately. [Offset sign and daylight saving time](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date/getTimezoneOffset)

## Test sites and remaining gaps

| Site | Use |
| --- | --- |
| [ipify](https://api.ipify.org?format=json) · [Cloudflare trace](https://1.1.1.1/cdn-cgi/trace) | Read the public source of this HTTPS connection. The trace also reports the Cloudflare edge that received it. |
| [IPPure DNS](https://ippure.com/en/DNS-Leak-Detect.html) · [Net.Coffee DNS](https://ip.net.coffee/dns/) · [bash.ws](https://bash.ws/dnsleak) | Compare the recursive resolvers seen in this run with the core's connection records. |
| [IPPure WebRTC](https://ippure.com/en/Browser-WebRTC-Leak-Detect.html) · [Net.Coffee WebRTC](https://ip.net.coffee/webrtc/) | Cross-check exposed candidate types and address classes. |
| [Bilibili](https://www.bilibili.com/) | Exercise a real domestic HTTPS route and confirm `DIRECT` in the log. |

Each test service sees the public source of the request it receives. Share categories such as expected proxy egress, direct public address, or internal VPN address in place of raw values. The recorded tests did not cover device-wide physical-interface capture, every app-managed resolver, reboots, network handovers, long screen-off periods, unexpected VPN exit, hours-long model streams, or full WeChat login, voice, and video workflows.
