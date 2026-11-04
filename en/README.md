# Anthropic/Claude Risk Control Handling

**English guide** · [Choose a language](../README.md)

This setup sends Claude connections through a chosen proxy while domestic sites keep their existing direct routes. Start with your platform. It links to the shared configuration at the point where you need it.

| Start here | What you will be able to do |
| --- | --- |
| [Windows](WINDOWS.md) | Capture traffic with Clash Verge Rev, check the Claude Code proxy and process time zone, and restrict the WebRTC addresses that Chrome exposes to pages. |
| [Android](ANDROID.md) | Identify which package owns a connection, route it through FlClash, and check what happens when the VPN stops. |
| [Routing and DNS](ROUTING.md) | Build a single-node group, order the rules, and choose distinct DNS paths for application traffic and the proxy hostname. |
| [11 checks and test sites](VERIFICATION.md) | Use live connections, DNS observations, and WebRTC candidates to decide what a test actually establishes. |

### Follow one domestic request

Opening Bilibili involves two connections. First the device needs an address for the site. In this setup, Mihomo sends that **DNS lookup** to Cloudflare through the dedicated proxy. The subsequent **HTTPS connection to the site** matches a domestic `DIRECT` rule. Both log entries are correct. Bilibili sees the public source of the direct connection. A DNS test sees the recursive resolver that looked up its test name.

### What the recorded tests showed

| Check | Windows | Android |
| --- | --- | --- |
| Routing | Claude Code used the local proxy. Sampled Claude domains matched the dedicated group, and Bilibili HTTPS matched `DIRECT`. | Sampled Claude, GBox, and microG connections matched their package rules. Bilibili HTTPS matched `DIRECT`. |
| DNS | Fresh domains and TXT probes showed only Cloudflare. The core recorded DoH through the dedicated group. | Browser and core samples agreed. Both platforms kept a direct encrypted lookup for the proxy hostname. |
| WebRTC | The tested pages returned no candidates after the stricter settings were applied. | Ten STUN targets yielded no public candidate. A VPN-internal address was still visible. |
| Time zone | Windows and Chrome used UTC+8. The Claude Code process used `America/Los_Angeles`. | The phone used its UTC+8 system time zone. |
| Capture stopped | Blocking traffic after an unexpected proxy exit was not tested. | Manually stopping the VPN restored ordinary connectivity. Unexpected disconnection was not tested. |

The recorded versions were Clash Verge Rev 2.5.8, Mihomo 1.19.32, and FlClash 0.8.99. Repeat the relevant checks after changing a node, network, or client version. Record the version, matching rule, and result; keep public addresses, subscriptions, and full logs on the device. These tests describe network behavior. They cannot determine how a service evaluates an account.
