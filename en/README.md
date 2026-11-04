# Anthropic/Claude Risk Control Handling

**English edition** · [Language selection](../README.md)

This guide describes how to give Claude a dedicated proxy route on Windows and Android while keeping selected domestic websites on their usual direct route. It covers the configuration, the checks used in one deployment, the observed results, and how to roll back.

| Start here | What you will find |
| --- | --- |
| [Windows setup](WINDOWS.md) | Clash Verge Rev, Claude Code, Chrome WebRTC settings, and new-tab recovery |
| [Android setup](ANDROID.md) | FlClash, Claude installed through GBox, VPN coverage, and rollback |
| [Shared routing and DNS](ROUTING.md) | A dedicated proxy group, rule order, encrypted DNS, and the bootstrap exception |
| [11 verification checks and test sites](VERIFICATION.md) | How to run each check, interpret its result, and identify what still needs testing |

## Observed results

| Check | Windows | Android |
| --- | --- | --- |
| Domestic websites | The tested Bilibili HTTPS connection matched `DIRECT` | The HTTPS probe to the same site matched `DIRECT` |
| Claude traffic | Claude Code used the local proxy settings. Matching domains used the dedicated group. | Sampled Claude, GBox, and microG connections matched their respective process rules. |
| Ordinary DNS | Random-domain and TXT probes showed only the expected recursive resolver operator. Core connections showed DoH through the proxy. | Browser and core probes gave the same result. |
| DNS exception | Resolving the proxy server's hostname used a direct, encrypted bootstrap query. | The same exception applied. |
| WebRTC | After the strict settings were applied, the tested pages returned no ICE candidates. | Ten STUN targets yielded no public candidate. A VPN-internal host candidate was still visible. |
| Time zone | Windows and Chrome used UTC+8. The Claude Code process used Los Angeles time. | The phone kept its UTC+8 system time zone. |
| VPN stopped | Blocking traffic after an unexpected failure was not tested. | Manually turning off the VPN restored ordinary connectivity. |

The Chrome-only Los Angeles time-zone experiment was withdrawn after it broke the new-tab page. Removing the extension restored that page, and all five tested browser contexts again reported the system time zone. [Chrome recovery steps](WINDOWS.md#restore-the-chrome-new-tab-page)

## Order of operations

1. **Back up.** Record the active subscription, the ordinary group's selected node, and the routing mode. Back up the proxy configuration and application settings separately.
2. **Configure the platform.** Create a group with one eligible node, put the dedicated rules ahead of the general rules, then configure DNS interception. Process matching differs between Windows and Android.
3. **Verify real connections.** Check domestic HTTPS, Claude traffic, the DNS upstream route, and WebRTC candidates independently. Keep reproducible observations in a local record.
4. **Save a rollback point.** Close temporary control endpoints and remove test port forwarding before exporting the final configuration.

The recorded deployment used Clash Verge Rev 2.5.8, Mihomo 1.19.32, and FlClash 0.8.99. Repeat the relevant checks after a version, node, or network change. In a shared report, record versions, matched rules, candidate types, and conclusions. Keep raw material containing node credentials, device identifiers, or public IP addresses on your own device.
