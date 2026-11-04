# Android setup

[Guide home](README.md) · [Shared routing and DNS](ROUTING.md) · [11 verification checks](VERIFICATION.md)

This example uses FlClash 0.8.99. Android first decides which apps enter the VPN. Only then can Mihomo decide how to route their connections. The VPN icon confirms that a tunnel exists.

## 1. Identify the app that opens the connection and back up the profile

1. Open Claude and look for a new connection in FlClash's **Connections** view. In the recorded setup, Claude downloaded through GBox had its own Android package. If the container owns the connection on your phone, match the container you observe. The app store and the process opening a socket can be different.
2. To check installed packages from a computer, connect over USB, approve the debugging session on the phone, and run `adb devices`. When it shows `device`, run the queries below. They establish which packages are installed; use step 1 to establish which one connected. `com.follow.clash` is FlClash itself. [Android ADB](https://developer.android.com/tools/adb)

```powershell
adb shell pm list packages com.anthropic.claude
adb shell pm list packages com.gbox.android
adb shell pm list packages com.google.android.gms
adb shell pm list packages com.follow.clash
```

3. Record the active subscription, ordinary node, mode, VPN switch, and app access list. Export the original ZIP through **Tools → Backup and restore → Backup**. Edit only a copy. The full restore used later replaces several settings together, so a note of the node name alone cannot undo it.

## 2. Bind the script to the active subscription

In the active profile's **Overwrite** settings, choose **Script**, bind the [shared script](ROUTING.md), set `android = true` and the source node name, and reapply the profile. The binding belongs to a particular subscription. Editing a different one can leave the VPN running with the old rules. [Override screen in this release](https://github.com/chen08209/FlClash/blob/v0.8.99/lib/views/profiles/overwrite/overwrite.dart)

To reproduce the recorded **offline backup edit**, keep the three pieces below consistent. If you used the app's editor, skip ahead to checking the four generated rules.

| In the backup copy | Why it matters and what to check |
| --- | --- |
| Read `currentProfileId` in `config.json` and locate `profiles/<currentProfileId>.yaml`. | A phone can store several subscriptions. Matching the ID and file prevents a successful restore from activating a different configuration. |
| Save the script as `scripts/<script_id>.js`. In `database.sqlite` register it in `scripts`. In the same transaction set the corresponding `profiles` row's `overwrite_type` to `script` and its `script_id` to that ID. Update only the dedicated group's entry in `selected_map`. | The file holds the code. The database tells a profile to use it. A transaction commits or rolls back the binding together; preserving the other map entries keeps ordinary node selections. |
| Handle IDs with an integer-preserving parser such as Python's standard `json`. Commit, run `PRAGMA integrity_check`, close the database, and repack with `config.json` and `database.sqlite` at the ZIP root. | A rounded ID can point at the wrong record. An extra directory changes the import paths. An `ok` result checks database structure; live connections still need testing. |

[Backup layout in this release](https://github.com/chen08209/FlClash/blob/v0.8.99/lib/common/backup.dart) · [SQLite integrity check](https://www.sqlite.org/pragma.html#pragma_integrity_check)

Check the first four rules in the generated configuration. The first rejects only Chrome UDP. The next three select the dedicated group by the package that opened the connection. The node retains UDP forwarding for other application traffic. Routing GBox or `com.google.android.gms` (microG in this setup) by package also affects connections they make for other apps.

```text
AND,((PROCESS-NAME,com.android.chrome),(NETWORK,UDP)),REJECT
PROCESS-NAME,com.anthropic.claude,Claude-Fixed
PROCESS-NAME,com.gbox.android,Claude-Fixed
PROCESS-NAME,com.google.android.gms,Claude-Fixed
```

## 3. Set the VPN's capture scope

| Setting and value | How it works and what to check |
| --- | --- |
| `vpnProps.systemProxy: false` and `allowBypass: false` | Use the Android VPN interface without advertising an extra HTTP system proxy or permitting apps to opt out of the VPN. Check app access control separately. In an allowlist, include Claude, its actual container if relevant, and the browser being tested. |
| `patchClashConfig.mode: rule` and `find-process-mode: always` | Route by rule and look up connection owners. This release asks Android for the connection's UID and maps it to a package. Check the live name, especially for containers or packages sharing a UID. |
| `vpnProps.dnsHijacking: true` and `patchClashConfig.tun.dns-hijack` containing `any:53` and `tcp://any:53` | Hand ordinary UDP/TCP port 53 traffic captured by the VPN to the core. Android Private DNS and app-managed encrypted DNS use separate connections and need their own checks. |
| `overrideDns: false` and `networkProps.appendSystemDns: false` | The subscription script supplies the DNS settings here. Turning off the separate UI override and system DNS appending avoids replacing or expanding the intended resolver list. Inspect the running configuration. |
| `vpnProps.ipv6: false`, `patchClashConfig.ipv6: false`, and `dns.ipv6: false` | These control the VPN's IPv6 addresses and routes, IPv6 use by the core, and AAAA answers respectively. Suppressing AAAA alone still lets an app try a known IPv6 address, so test a literal address too. |

Turn off **Allow LAN** and **External controller** for normal use so other LAN devices cannot use these listeners to reach the local proxy or control the core. The fields and menus above refer to 0.8.99. Check the implementation before importing a backup into a different version. [FlClash VPN implementation](https://github.com/chen08209/FlClash/blob/v0.8.99/android/service/src/main/java/com/follow/clash/service/VpnService.kt) · [Android VPN address families and bypass](https://developer.android.com/reference/android/net/VpnService.Builder)

## 4. Start the VPN and follow real connections

1. For an offline edit, import the copy through **Tools → Backup and restore → Restore → Restore all data**. Check the active subscription, **Rule** mode, ordinary group, and dedicated group on the dashboard, then start the VPN. If you edited in the app, reapply the profile and start the VPN.
2. Open Claude and the associated services you use. For each new connection, read the package, destination, matched rule, and final outbound chain. Then open Bilibili in the browser and check that its HTTPS connection used `DIRECT`. This tests both the intended proxy path and an intended direct path.
3. Run the [fresh-domain DNS and multi-target WebRTC checks](VERIFICATION.md) in the phone's browser. The recorded ten-target STUN test yielded no public candidate and retained a VPN-internal `host` candidate. The core also recorded rejected Chrome UDP connections.
4. With the VPN active, inspect IPv6 routes and try a literal address to bypass DNS, for example `adb shell ping -6 -c 1 2606:4700:4700::1111`. The recorded phone reported no route, and IPv6 HTTPS also failed in its browser. A ping timeout alone can mean the target ignored ICMP. A test from the ADB shell cannot stand in for the target app.

## 5. Test the manual switch, clean up, and preserve a way back

- **Stop and restart the VPN.** Open an ordinary page while it is off, then recheck routing after restarting it. This preserves normal networking when you deliberately stop the VPN. Its routing rules stop with it. Crash-time blocking, reboots, and network handovers were not tested.
- **Check background behavior.** This setup retained a `deviceidle` exemption for FlClash. Inspect it with `adb shell dumpsys deviceidle whitelist`. A listing confirms the exemption, not that the VPN will survive hours with the screen off. [Android Doze](https://developer.android.com/training/monitoring-device-state/doze-standby)
- **Remove temporary connections.** Compare `adb forward --list` and `adb reverse --list` with your notes. Remove only mappings you created, for example `adb forward --remove tcp:19090`, `adb forward --remove tcp:17890`, or `adb reverse --remove tcp:18765`. Use your own mapping's port. Close the temporary controller when finished.
- **Keep a restorable version.** Export the verified ZIP. To undo the changes, back up the current state, use **Restore all data** with the original ZIP, then recheck the subscription, VPN, and domestic access. An idle exemption is an OS setting. If you added it for this setup, remove it separately with `adb shell dumpsys deviceidle whitelist -com.follow.clash`.
