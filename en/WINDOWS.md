# Windows setup

[Guide home](README.md) · [Shared routing and DNS](ROUTING.md) · [11 verification checks](VERIFICATION.md)

Clash Verge Rev chooses the route for a connection. Claude Code selects a local proxy, and Chrome controls which WebRTC addresses a page can collect. Check each at the point where it takes effect.

## 1. Record a baseline you can restore

1. In Clash Verge Rev's **Profiles** and **Settings**, note the active subscription, ordinary group selection, mixed port, mode, and TUN switch. Use the actual port later when pointing the CLI at the proxy.
2. Open the configuration directory from the application and back up the active subscription's override script locally. A common location is `%APPDATA%\io.github.clash-verge-rev.clash-verge-rev\profiles`. Subscription refreshes rebuild the runtime configuration, so keep the script that produces it.
3. Back up `~/.claude/settings.json`. Record its existing `env` values and Chrome's WebRTC options, and run `Get-TimeZone`. This gives you separate restore points for the CLI, browser, and proxy, and a reference for the system clock.

## 2. Capture traffic, then route it by rule

Select **Rule** and enable **TUN** in Clash Verge Rev. Merge the fields below into the current configuration, retaining existing interface options. Apply it and inspect the configuration the core actually loaded.

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

| Setting | What it does here |
| --- | --- |
| `mode: rule` | Domestic HTTPS can match `DIRECT` while Claude matches its own group. The same core handles both connections and chooses a different outbound route for each. |
| `auto-route` and `strict-route` | The first sends the relevant operating-system traffic into the virtual interface. On Windows, the second tightens routing and restricts DNS queries across multiple interfaces. Recheck the path whenever TUN stops. |
| `any:53` and `tcp://any:53` | The first intercepts UDP port 53 by default. The second covers TCP port 53. A browser running its own DoH usually uses HTTPS; inspect its resolver connection separately. |
| `find-process-mode: strict` | The core looks up the process when a rule needs it. That lets a rule target UDP from `chrome.exe` specifically. |

[Mihomo TUN](https://wiki.metacubex.one/en/config/inbound/tun/) · [Process matching](https://wiki.metacubex.one/en/config/general/)

## 3. Bind the dedicated route and DNS to the active subscription

1. Apply the [shared configuration](ROUTING.md) in the active subscription's persistent script. Leave `android = false` on Windows. Save and reapply the subscription. In the generated configuration, confirm that Chrome UDP comes first and the dedicated group contains only the intended node. This checks that the core received your edit. [Clash Verge Rev script guide](https://clash-verge-rev.github.io/guide/script.html)
2. Open **Connections** and trigger a fresh Claude request and a Bilibili HTTPS request. Claude should match `Claude-Fixed`. The domestic rule should send Bilibili to `DIRECT`. Check both the matching rule and the final outbound route. A page loading successfully only shows that *some* route worked.

## 4. Restrict the WebRTC addresses Chrome gives to pages

1. In **WebRTC Control → Options**, set D to **Disable non proxied udp**. To establish a call, WebRTC looks for reachable addresses. Restricting direct UDP reduces the chance that a STUN server discovers a public address through a different exit.
2. Enable A, **Block WebRTC Support Detection**, and C, **Block Additional WebRTC Objects**. Confirm protection is **ON**. These controls restrict how pages detect or use the relevant APIs. Reload the test page so the new document picks up the settings. Test both the page and an iframe, which has its own script environment.
3. Check the core rule `AND,((PROCESS-NAME,chrome.exe),(NETWORK,UDP)),REJECT`. It rejects UDP that the core attributes to Chrome, including traffic used by QUIC, voice, and video. Sample both domestic and overseas STUN targets. An initial test against overseas targets missed a direct address that a domestic target later found.

After the settings were tightened, the tested Windows pages returned no candidates. The core recorded no hit on the fallback UDP rule during that run. Record these two observations separately. [WebRTC IP address handling](https://www.rfc-editor.org/rfc/rfc8828.html) · [How to read candidates](VERIFICATION.md)

## 5. Set the Claude Code proxy and process time zone

**Merge** these keys into `env` in the user-level `~/.claude/settings.json`. Preserve other settings. Replace both instances of `7897` with the mixed port recorded in step 1.

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

- `HTTP_PROXY` and `HTTPS_PROXY` direct CLI requests to the proxy on loopback. The `http://` in the value describes the connection to that local proxy. For an HTTPS destination, the CLI can open a tunnel with `CONNECT` and establish TLS to the destination inside it.
- `NO_PROXY` lets local services communicate directly. Adding a remote service domain would let matching requests skip the explicit CLI proxy. If the expected connection is missing, check existing proxy variables, including lowercase variants and their precedence. [Claude Code network configuration](https://code.claude.com/docs/en/network-config)
- `TZ` controls how the process displays local time. `America/Los_Angeles` includes daylight saving rules. Fully quit and restart the CLI so the new process reads the settings. Run `Get-TimeZone` in a separate PowerShell to check the system's UTC+8 setting. Changing a displayed time zone does not move network egress.

**Verify it.** Trigger a normal request from a new CLI session and check the local proxy's connections. If Node is installed, enter `! node -p "Intl.DateTimeFormat().resolvedOptions().timeZone"` in the CLI's local command mode to check the time zone inherited by a child process. Running Node in a separate PowerShell only tests that terminal. The recorded test also checked the CLI process itself with an isolated configuration. A local proxy that always returned `503` received `CONNECT api.anthropic.com:443`. That established that the CLI selected the proxy; the test forwarded no request to the API. [CLI local command mode](https://code.claude.com/docs/en/interactive-mode)

## 6. Verify and roll back

Use the [verification checks](VERIFICATION.md) for domestic HTTPS, the Claude route, the DNS upstream, IPv6, and browser candidates. To roll back, save the current state, restore the original subscription script, and reapply it. Restore the TUN switch, mode, and ordinary group selection. Return the four `env` keys and WebRTC options to their previous values, restart the CLI, reload the page, and compare the same probes.
