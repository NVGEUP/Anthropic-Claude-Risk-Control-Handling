# 11 verification checks and test sites

[Guide home](README.md) · [Windows](WINDOWS.md) · [Android](ANDROID.md) · [Shared routing and DNS](ROUTING.md)

Before collecting a sample, confirm that VPN or TUN capture is active and the proxy client is in Rule mode. For each check, record the **action, observation, and conclusion** separately. Compare actual addresses locally. In a shared report, use categories such as expected proxy egress, local public address, or private host candidate.

## Verification matrix

| # | Check | How to test | Recorded result and limits |
| --- | --- | --- | --- |
| 1 | Egress IP | Sample HTTPS egress from the target client and compare results across services. | Short samples agreed. A carrier label cannot establish exclusive use or prior reputation. |
| 2 | Egress consistency | Switch the ordinary group, inspect the dedicated group, and repeat the egress sample. | The dedicated group contained one node. Its provider can still change the actual egress IP. |
| 3 | TCP, UDP, and WebRTC | Check the outer proxy TCP/TLS transport, then collect ICE candidates using both domestic and overseas STUN targets. | After hardening, sampled Windows pages returned no candidates. Android retained a VPN-internal host candidate. |
| 4 | TLS | Check certificate validation settings and make a certificate-validated HTTPS request to the service. | Auxiliary probes validated certificates. The outer proxy and the application establish separate TLS handshakes. |
| 5 | DNS | Generate fresh random subdomains for browser and UDP/TCP TXT probes. Inspect upstream connections. | Sampled application queries used the expected DoH route. Proxy hostname bootstrap used a direct exception. |
| 6 | IPv6 | Inspect DNS policy, VPN address families, and reachability of a literal IPv6 destination. | Windows retained system IPv6. The tested phone had no IPv6 route. |
| 7 | Dedicated Claude routing | Check rules and final routes for API, CDN, and related process connections. | Observations supported Windows domain matching and the three Android application rules. |
| 8 | Rule priority | Inspect the merged order and open a domestic site. | Dedicated rules came first. Sampled domestic HTTPS still matched DIRECT. |
| 9 | Connection reuse | Send consecutive unauthenticated requests, count TLS connections, and include a short idle interval. | Five auxiliary requests shared one TLS connection, including after a ten-second idle period. HTTP 401 indicated an unauthenticated request. |
| 10 | Concurrency and limits | An authorized real workload and service-side limit records are needed for this check. | No account stress test or paid model call was made. The tests establish no safe concurrency limit. |
| 11 | Path and latency | Record the proxy chain, response latency, and visible edge-site labels together. | Only short samples were collected. An edge location does not identify the Claude backend or the complete route. |

These checks use a third-party checklist as their organizing scheme. The probes do not verify account risk controls, device recognition, or long-term model stability. [Original checklist](https://ip.net.coffee/claude/claudecode.html)

## Verify DNS

1. In the actual browser on the device being tested, start a random-domain DNS test. Use a fresh test session every time. Inspect the **recursive resolvers** observed by the authoritative service.
2. Test ordinary DNS on the same device. On Windows, use `Resolve-DnsName <fresh-test-domain> -Type TXT -DnsOnly`. Use a different fresh name with `-TcpOnly` for TCP port 53. Check the core's transport records for the UDP path. Historical Android checks issued queries separately from the phone's browser and its core.
3. In the core's connections view, locate the Cloudflare DoH connections and check that their outbound chains include the dedicated group. Record proxy-hostname bootstrap separately.
4. If the report shows an ISP resolver or application queries using an unexpected direct upstream, inspect TUN port 53 interception, `direct-nameserver`, fallback servers, appended system DNS, and application-managed DoH/DoT.
5. If you only have synthetic A records from `fake-ip`, or if all requests time out, record the result as **inconclusive**. An authorized physical-interface capture can provide further evidence about traffic leaving directly.

On Windows, random browser queries plus six UDP and six TCP TXT probes showed only Cloudflare resolvers. On Android, twelve random browser domains and twelve core TXT queries also showed only Cloudflare. Both systems' core DoH routes were inspected. The bash.ws report also displayed `DNS may be leaking.` Retain that message and interpret it alongside the actual resolver and routing records.

## Verify WebRTC

| Observation | What to record |
| --- | --- |
| `host` | Note a local interface, VPN-internal address, or mDNS name. Distinguish the actual LAN from the VPN interface. |
| `srflx` | STUN discovered a mapped address. Locally compare it with the direct public address and the expected proxy egress. |
| `relay` | Record the address allocated by a TURN relay separately. |
| Empty candidates, API unavailable, or timeout | Record these as distinct outcomes. Compare several STUN targets and the core's rule hits. |

Refresh the multi-target test page. The initial Google-only STUN check missed a domestic direct route. After hardening, the sampled Windows top-level page, iframe, and compatibility API returned no candidates. The ten-target Android sample had no public candidate and retained a VPN-internal host address. The core recorded Chrome UDP rejections on Android. Repeat relevant checks after browser updates and network changes.

## Verify the time zone

Run the following read-only snippet in DevTools Console on an ordinary page. The output describes the JavaScript context where it ran.

```javascript
const year = new Date().getUTCFullYear();
({
  zone: Intl.DateTimeFormat().resolvedOptions().timeZone,
  january: new Date(Date.UTC(year, 0, 15, 12)).getTimezoneOffset(),
  july: new Date(Date.UTC(year, 6, 15, 12)).getTimezoneOffset()
})
```

For this UTC+8 setting, the expected values are `Asia/Shanghai` and `-480 / -480`. For `America/Los_Angeles` they are typically `480 / 420` in January and July. The recorded verification also covered a cross-origin iframe, Web Worker, Shared Worker, and Service Worker. After the time-zone extension was removed, all five contexts reported UTC+8. Long-term browser stability requires separate testing.

## Test sites

Opening an external test sends a request from the test connection and exposes its public source IP to the service. Keep raw results locally and share redacted observations.

| Site | Use in this procedure |
| --- | --- |
| [IPPure WebRTC](https://ippure.com/en/Browser-WebRTC-Leak-Detect.html) · [Net.Coffee WebRTC](https://ip.net.coffee/webrtc/) | Cross-check multiple STUN targets. |
| [IPPure DNS](https://ippure.com/en/DNS-Leak-Detect.html) · [Net.Coffee DNS](https://ip.net.coffee/dns/) · [bash.ws](https://bash.ws/dnsleak) | Compare recursive resolvers with authoritative records for fresh domains. |
| [ipify](https://api.ipify.org?format=json) · [Cloudflare trace](https://1.1.1.1/cdn-cgi/trace) | Sample HTTPS egress and Cloudflare edge information. |
| [Bilibili](https://www.bilibili.com/) | Verify a direct domestic HTTPS connection against the connection log. |

## Scenarios still to test

Coverage did not include device-wide physical-interface capture, all application-managed DNS, reboots, network handovers, long screen-off periods, unexpected VPN exit, hours-long model streaming, or full login, voice, and video workflows in apps such as WeChat. On Android, manually stopping the VPN deliberately restores ordinary connectivity. A domestic website reached through DIRECT can see the source of that connection.
