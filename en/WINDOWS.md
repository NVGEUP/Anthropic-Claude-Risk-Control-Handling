# Windows setup

[Guide home](README.md) · [Shared routing and DNS](ROUTING.md) · [Verification checks](VERIFICATION.md)

These steps cover Clash Verge Rev, Claude Code, and Chrome. Check each layer separately. TUN intercepts system traffic. Claude Code reads its proxy and time-zone settings at startup. Chrome's WebRTC settings affect which ICE candidates a page can obtain.

## 1. Record the starting configuration

1. In Clash Verge, note the active subscription and the ordinary proxy group's current selection. Record the mixed proxy port, TUN setting, and routing mode.
2. Open the application's configuration directory and locate the persistent override associated with the active subscription. A typical directory is `%APPDATA%\io.github.clash-verge-rev.clash-verge-rev\profiles`. Back up the relevant override and settings elsewhere on the same computer.
3. Back up `~/.claude/settings.json`. Note any existing values in `env` so you can restore individual entries.
4. Run `Get-TimeZone` in PowerShell and record the Windows time zone.

## 2. Enable TUN in Rule mode

Select **Rule** in Clash Verge and turn on TUN. Check the generated configuration for the following fields. Merge the settings into the existing configuration and preserve the current interface options.

```yaml
mode: rule
find-process-mode: strict
tun:
  enable: true
  auto-route: true
  strict-route: true
  dns-hijack:
    - any:53
    - tcp://any:53
```

`auto-route` and `strict-route` control route capture. The two interception rules cover ordinary UDP and TCP queries on port 53. Check connection records separately for applications that perform their own DNS resolution over HTTPS.

## 3. Apply the dedicated route and DNS settings

Configure the node, group, domain rules, and DNS settings using the shared procedure. Put the Windows Chrome UDP rule first, followed by the Claude rules and the original subscription rules. Save the changes in the **active subscription's persistent override**, apply that subscription, and inspect the resulting configuration. [Shared procedure](ROUTING.md)

Open a Claude endpoint covered by the rules and a Bilibili page. In the core's connections view, the first connection should use `Claude-Fixed` and the second should match `DIRECT` if the original rules allow it. Check the group's actual membership again after a source node changes name.

## 4. Restrict WebRTC candidates in Chrome

1. Open **WebRTC Control → Options**. In the tested version, set D to **Disable non proxied udp**.
2. Enable A, **Block WebRTC Support Detection**, and C, **Block Additional WebRTC Objects**. Confirm that the extension displays **ON**.
3. Refresh the test page. Check the top-level page and an iframe, and use multiple STUN targets on both domestic and overseas networks.
4. Check Chrome UDP traffic in the connections view. The first fallback rule is `AND,((PROCESS-NAME,chrome.exe),(NETWORK,UDP)),REJECT`. It also affects QUIC, browser calls, and conferencing in Chrome.

An initial check using only overseas STUN targets missed a domestic public candidate. A broader test exposed it. After the stricter settings were applied, the tested pages returned an empty candidate list. The Windows sample recorded no live hit on the fallback UDP rule. An attempt to set a system-level Chrome WebRTC policy had failed, so the recorded result rests on the extension settings and the page tests.

## 5. Set the Claude Code proxy and process time zone

Merge these entries into `env` in the user-level `~/.claude/settings.json`. The example uses local HTTP mixed port `7897`. Use the port shown by your client and preserve existing permission, MCP, and application settings.

```json
{
  "env": {
    "HTTP_PROXY": "http://127.0.0.1:7897",
    "HTTPS_PROXY": "http://127.0.0.1:7897",
    "NO_PROXY": "localhost,127.0.0.1,::1",
    "TZ": "America/Los_Angeles"
  }
}
```

Exit all old Claude Code processes before starting a new one. If the route differs from the configured value, check for lowercase proxy variables that take precedence. The recorded startup test used an isolated CLI configuration, dummy credentials, and a loopback proxy that returned HTTP 503. It observed both the process time zone and an HTTPS CONNECT request. [Claude Code network configuration](https://code.claude.com/docs/en/network-config)

## Restore the Chrome new-tab page

The experimental time-zone override initially returned Los Angeles time in five browser contexts. Its replacement new-tab page subsequently failed to open. The following steps restored normal browsing.

1. Cancel the debugging notice from the time-zone extension at the top of Chrome.
2. Open `chrome://extensions` and remove the custom Los Angeles time-zone extension. **WebRTC Control is a separate protection**.
3. Close the affected tab, press `Ctrl+T`, enter a normal URL, and test navigation and refresh.
4. In PowerShell, `Get-TimeZone` should show `China Standard Time` for this UTC+8 setup. After recovery, the top-level page, cross-origin iframe, Web Worker, Shared Worker, and Service Worker all reported `Asia/Shanghai`. The January and July values of `getTimezoneOffset()` were both `-480`.

To restore the system setting, open **Settings → Time & language → Date & time** and select the UTC+08:00 Beijing time zone. The Claude Code `TZ` entry applies to its process. A stable Chrome-only Los Angeles time zone was not achieved in this deployment.

## 6. Verify and roll back

- **Routing and DNS.** Check domestic HTTPS, Claude traffic, and the DoH upstream. Windows retained its IPv6 stack and TUN capture. DNS suppressed AAAA answers, and the dedicated proxy connection used IPv4.
- **Browser.** Record candidate types, address classes, and timeouts across multiple STUN targets. If you need browser calls, test them separately with the strict UDP rule.
- **Proxy rollback.** Back up the current state, restore the previous persistent override, and reapply the subscription. Restore the prior TUN setting and ordinary group selection, then test domestic access.
- **CLI rollback.** Restore the four affected environment entries to their backed-up values. Exit and restart the CLI.

See the [11 verification checks](VERIFICATION.md) for the acceptance criteria and untested scenarios.
