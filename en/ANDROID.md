# Android setup

[Guide home](README.md) · [Shared routing and DNS](ROUTING.md) · [Verification checks](VERIFICATION.md)

This deployment used FlClash 0.8.99. Although Claude was downloaded through GBox, Android exposed it as a separate package. Identify the process making each connection before deciding which application rules to apply.

## 1. Identify the packages and export a backup

1. Unlock the phone and connect it to the computer over USB. If using ADB, approve USB debugging on the phone and run `adb devices`. The device status should be `device`.
2. Check for the package names below. If the system shows only a container, use FlClash's connections view to identify the process that actually sends the traffic.
3. Record the ordinary proxy group selection, mode, and VPN state. In FlClash, use **Tools → Backup and restore → Backup** to export a fresh full backup. Keep the original safe locally and work on a copy.

```powershell
adb shell pm list packages com.anthropic.claude
adb shell pm list packages com.gbox.android
adb shell pm list packages com.google.android.gms
adb shell pm list packages com.follow.clash
```

## 2. Bind the persistent override to the active subscription

Prepare the single-node group, DNS settings, and original subscription rules using the shared procedure. On Android, route by package name. The first rule rejects Chrome UDP. The dedicated node retains UDP forwarding for application traffic such as NTP. [Rule order and DNS](ROUTING.md)

The recorded setup changed three parts of the backup. If you edit the settings in FlClash, export another backup and use the same relationships to check the result.

| Backup component | Change | Verification |
| --- | --- | --- |
| `config.json` | Read `currentProfileId`. Update the VPN and core fields in the next section. | The profile ID retains its original numeric value and unrelated settings are preserved. |
| `scripts/<script_id>.js` | Store the Mihomo override used by this subscription. | The first four rules in the generated configuration match the list below. |
| `database.sqlite` | Register the local script in `scripts`. For the active `profiles` row, set `overwrite_type` to `script` and set `script_id` to the corresponding ID. Update only the dedicated group entry in `selected_map`. | The ordinary node selection is preserved and `PRAGMA integrity_check` returns `ok`. |

For offline backup editing, first check that `profiles/<currentProfileId>.yaml` exists in the copy. Insert the script record and update the active profile within one database transaction. Some JSON and SQLite identifiers exceed JavaScript's safe integer range. Use a parser that preserves integers, such as Python's standard `json` module. When repacking, keep `config.json` and `database.sqlite` at the ZIP root. Check the matching version's implementation before working with a different schema. [FlClash 0.8.99 source](https://github.com/chen08209/FlClash/tree/v0.8.99)

```text
AND,((PROCESS-NAME,com.android.chrome),(NETWORK,UDP)),REJECT
PROCESS-NAME,com.anthropic.claude,Claude-Fixed
PROCESS-NAME,com.gbox.android,Claude-Fixed
PROCESS-NAME,com.google.android.gms,Claude-Fixed
```

The GBox and microG rules also affect other connections from those processes. Recorded connections showed Claude API/CDN traffic, GBox service connections, and microG connections to Google matching their respective rules.

## 3. Configure VPN capture, DNS, and IPv6

| Setting | Recorded value | Purpose |
| --- | --- | --- |
| `vpnProps.systemProxy` | `false` | Use Android VPN capture. |
| `vpnProps.allowBypass` | `false` | Restrict applications from explicitly bypassing the VPN. |
| `vpnProps.dnsHijacking` | `true` | Intercept ordinary DNS. |
| `vpnProps.ipv6` | `false` | Leave the IPv6 address family unconfigured on this VPN. |
| `patchClashConfig.mode` | `rule` | Retain domestic and other existing routes. |
| `patchClashConfig.find-process-mode` | `always` | Obtain Android process information for package matching. |
| `patchClashConfig.ipv6` and `dns.ipv6` | `false` | Disable IPv6 in the core and suppress AAAA answers. |
| `patchClashConfig.tun.dns-hijack` | `any:53` and `tcp://any:53` | Intercept UDP and TCP port 53. |
| `overrideDns` and `networkProps.appendSystemDns` | `false` | Keep the override's DNS settings and avoid appending system resolvers. |

In FlClash, confirm that **Allow LAN** and the **External controller** are off for everyday use. On the phone, set `find-process-mode` to `always` and check live rule matches to confirm that the core recognized each process. [Android VpnService.Builder](https://developer.android.com/reference/android/net/VpnService.Builder)

## 4. Import the candidate and check live connections

1. If you edited a backup copy, choose its ZIP under **Tools → Backup and restore → Restore → Restore all data**. Have the original backup available before importing.
2. Return to the dashboard. Check **Rule** mode, the ordinary node selection, and the dedicated single-node group. Then start the VPN.
3. Open Claude and the associated services normally. In the connections view, inspect the package, destination, matching rule, and final outbound route. Check the owning process for both ordinary service domains and dynamic telemetry domains.
4. Open a domestic HTTPS site on the phone and verify that the connection matches `DIRECT`. Then test the public egress address, DNS, and multiple STUN targets. [Test sites and acceptance criteria](VERIFICATION.md)

## 5. Check the VPN boundary and clean up

- **IPv6.** With the VPN running, check address families and routes. Supplement that check with an IPv6 literal such as `adb shell ping -6 -c 1 2606:4700:4700::1111`. In the recorded sample, the device reported no route and the browser's IPv6 HTTPS request also failed.
- **WebRTC.** Ten STUN targets yielded no public candidate, but the browser still returned a VPN-internal `host` address. The core also recorded hits on the Chrome UDP reject rule.
- **Manual switching.** Manually turning off the VPN restored ordinary connectivity. The routing and DNS protections stop with it. Crash-time network blocking, reboots, prolonged screen-off periods, and network handovers were not tested.
- **Background operation.** The deployment retained FlClash's `deviceidle` exemption. Inspect it locally with `adb shell dumpsys deviceidle whitelist`. Vendor-specific power management requires separate verification.
- **End of test.** Close any temporary controller. Remove only the ADB forwards you created. The recorded session used `adb forward --remove tcp:19090`, `adb forward --remove tcp:17890`, and `adb reverse --remove tcp:18765`. Skip this step if no forwards were created.

## 6. Save the final backup and roll back if needed

Export the final state. Check the script binding, ordinary group selection, rule order, DNS/VPN settings, and closed control endpoint. To roll back, back up the current state first. Use **Restore all data** to restore the pre-change ZIP, then test domestic browsing and the original applications. The system idle exemption is separate from the backup. To remove that exemption, use `adb shell dumpsys deviceidle whitelist -com.follow.clash`.

The phone kept its UTC+8 system time zone. The checks covered live network connections and short HTTP connection reuse. Account concurrency, long-running workloads, and in-app location signals were outside the test scope.
