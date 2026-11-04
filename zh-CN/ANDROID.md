# Android 操作步骤

[手册首页](README.md) · [共用分流与 DNS](ROUTING.md) · [11 项验收](VERIFICATION.md)

本例使用 FlClash 0.8.99。手机先决定哪些应用进入 VPN，Mihomo 才能决定这些连接怎样出站。能看到 VPN 图标，只说明隧道已建立。

## 1. 找出真正发请求的应用，保存原配置

1. 打开 Claude，在 FlClash 的**连接**页观察新连接的包名。本例从 GBox 下载的 Claude 有独立包名；如果连接显示来自容器，应按实际显示的包名匹配。下载来源和联网进程可以不同。
2. 如使用电脑核对，连接 USB、在手机上允许本次调试，运行 `adb devices`。状态为 `device` 后再执行下列查询。它们只确认包已安装；连接归属仍看第 1 步。`com.follow.clash` 是 FlClash 本身。[Android ADB](https://developer.android.com/tools/adb)

```powershell
adb shell pm list packages com.anthropic.claude
adb shell pm list packages com.gbox.android
adb shell pm list packages com.google.android.gms
adb shell pm list packages com.follow.clash
```

3. 记下活动订阅、普通节点、模式、VPN 开关及应用访问控制范围。在 **Tools → Backup and restore → Backup** 导出原 ZIP，只编辑副本。稍后使用的“全部恢复”会同时替换多个设置，单独记下一个节点名不足以还原。

## 2. 把脚本绑到正在使用的订阅

在活动订阅的 **Overwrite / 覆写** 中选 **Script / 脚本**，绑定[共用脚本](ROUTING.md)，设置 `android = true` 和源节点名，然后重新应用。脚本绑定属于某个订阅；编辑了另一份订阅时，VPN 可以正常开启却仍运行旧规则。[该版本的覆写界面](https://github.com/chen08209/FlClash/blob/v0.8.99/lib/views/profiles/overwrite/overwrite.dart)

若复现历史上的**离线备份编辑**，下面三处必须指向同一份配置。通过界面完成绑定的读者可直接核对生成的四条规则。

| 在 ZIP 副本中做什么 | 为什么做、怎么核对 |
| --- | --- |
| 从 `config.json` 读取 `currentProfileId`，找到 `profiles/<currentProfileId>.yaml`。 | 手机可以保存多个订阅。先核对 ID 和文件，避免恢复成功后仍激活另一份配置。 |
| 将脚本保存为 `scripts/<script_id>.js`。在 `database.sqlite` 的 `scripts` 表注册脚本，在同一事务里把对应 `profiles` 行的 `overwrite_type` 设为 `script`、`script_id` 设为同一个 ID。仅更新 `selected_map` 的专用组条目。 | 文件存代码，数据库决定哪个订阅调用它。事务让两项绑定一起成功或一起撤销；保留其余条目可保留普通节点选择。 |
| 用能保留大整数的工具处理 ID（如 Python 标准 `json`），提交事务后运行 `PRAGMA integrity_check` 并关闭数据库，再打包。保持 `config.json` 与 `database.sqlite` 在 ZIP 根目录。 | ID 被舍入会指向错误记录，多套一层目录会改变导入路径。`ok` 只说明数据库结构完整，规则是否生效还要用连接来验证。 |

[该版本的备份结构](https://github.com/chen08209/FlClash/blob/v0.8.99/lib/common/backup.dart) · [SQLite 完整性检查](https://www.sqlite.org/pragma.html#pragma_integrity_check)

查看生成配置的前四条，顺序应如下。第一条仅拒绝 Chrome 的 UDP；后面三条按发起连接的软件包选用专用组，节点本身保留 UDP 转发能力。把 GBox 或 `com.google.android.gms`（本例是 microG）放进组，也会影响它们为其他应用发出的连接。

```text
AND,((PROCESS-NAME,com.android.chrome),(NETWORK,UDP)),REJECT
PROCESS-NAME,com.anthropic.claude,Claude-Fixed
PROCESS-NAME,com.gbox.android,Claude-Fixed
PROCESS-NAME,com.google.android.gms,Claude-Fixed
```

## 3. 设置 VPN 的接管范围

| 设置与取值 | 原理及核对点 |
| --- | --- |
| `vpnProps.systemProxy: false`、`allowBypass: false` | 使用 Android VPN 虚拟接口，不额外公布 HTTP 系统代理，也不授权应用主动绕过 VPN。另查应用访问控制：若启用“只代理选中应用”，Claude、实际容器及待测浏览器都须在范围内。 |
| `patchClashConfig.mode: rule`、`find-process-mode: always` | 按规则分流，并为连接查找所属应用。该版本借助系统查询连接的 UID，再映射到包名；共享 UID 或容器使界面实测尤其重要。 |
| `vpnProps.dnsHijacking: true`；`patchClashConfig.tun.dns-hijack` 包含 `any:53`、`tcp://any:53` | 把经过 VPN 的常规 UDP/TCP 53 查询交给核心。Android 私人 DNS 或应用自带的加密 DNS 使用另一条连接，须单独验收。 |
| `overrideDns: false`、`networkProps.appendSystemDns: false` | 本配置由订阅脚本生成 DNS。关闭额外的界面覆写和系统解析器追加，避免最终上游列表被换掉或混入其他服务器；以运行配置为准。 |
| `vpnProps.ipv6: false`、`patchClashConfig.ipv6: false`、`dns.ipv6: false` | 三处分别控制 VPN 的 IPv6 地址与路由、核心的 IPv6 使用和 AAAA 答复。只关闭 AAAA 仍允许应用尝试已知 IPv6 地址，因此需单测字面地址。 |

在日常配置中关闭 **Allow LAN** 和 **External controller**，避免局域网设备接入本机代理或操作核心。字段和菜单按 0.8.99 编写；版本不同时先核对实现再导入。[FlClash VPN 实现](https://github.com/chen08209/FlClash/blob/v0.8.99/android/service/src/main/java/com/follow/clash/service/VpnService.kt) · [Android VPN 地址族与绕行](https://developer.android.com/reference/android/net/VpnService.Builder)

## 4. 开启 VPN，用真实连接验收

1. 离线编辑者通过 **Tools → Backup and restore → Restore → Restore all data** 导入副本。回到首页核对活动订阅、**Rule**、普通组和专用组，再打开 VPN。界面编辑者直接重新应用并开启 VPN。
2. 依次打开 Claude 和用到的关联服务，在**连接**页查看新连接的包名、目标、命中规则和出站链。再用浏览器打开哔哩哔哩，核对其 HTTPS 为 `DIRECT`。这样分别验证了“应走代理”和“应走直连”的流量。
3. 在手机浏览器做[随机域名 DNS 和多目标 WebRTC 检测](VERIFICATION.md)。记录中，10 个 STUN 目标未产生公网候选，仍有 VPN 内部 `host` 候选；核心也记录到 Chrome UDP 拒绝命中。
4. 在 VPN 开启时核对 IPv6 路由，并用字面地址绕开 DNS，例如 `adb shell ping -6 -c 1 2606:4700:4700::1111`。本例报告无路由，手机浏览器的 IPv6 HTTPS 也失败。单次 ping 超时还可能是对方不回 ICMP；ADB shell 的结果也不能代替目标应用。

## 5. 验证手动开关，清理并回退

- **手动关、再开 VPN。** 关闭后用普通网页验证联网，重新开启后再核对分流。这样保留了主动关闭时正常上网的选择；关闭期间这套规则停止工作。异常退出锁网、重启和网络切换未测。
- **检查后台状态。** 本例给 FlClash 保留了 `deviceidle` 休眠豁免。`adb shell dumpsys deviceidle whitelist` 可查看；名单里存在应用只说明有豁免，不能证明熄屏数小时后仍在运行。[Android 休眠机制](https://developer.android.com/training/monitoring-device-state/doze-standby)
- **撤销临时连接。** 用 `adb forward --list` 和 `adb reverse --list` 对照本次记录。仅删除自己创建的映射，例如 `adb forward --remove tcp:19090`、`adb forward --remove tcp:17890` 或 `adb reverse --remove tcp:18765`；随后关闭临时控制口。端口号须与自己的映射一致。
- **保存可还原版本。** 导出验收通过的 ZIP。需要撤销时，备份现状，用 **Restore all data** 恢复原 ZIP，再核对订阅、VPN 和国内访问。休眠豁免是系统设置；若是本次新增，可用 `adb shell dumpsys deviceidle whitelist -com.follow.clash` 单独撤销。
