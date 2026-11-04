# Android 操作步骤

[手册首页](README.md) · [共用分流与 DNS](ROUTING.md) · [验收清单](VERIFICATION.md)

本次使用 FlClash 0.8.99。Claude 从 GBox 下载，但系统中存在独立软件包。先确认连接的实际归属，再决定应用规则的范围。

## 1. 核对软件包并导出备份

1. 解锁手机并通过 USB 连接电脑。若使用 ADB，在手机上确认 USB 调试授权，运行 `adb devices`，设备状态应为 `device`。
2. 检查以下包名是否存在。若系统只显示容器，应在 FlClash 连接页先确认真正发出请求的进程。
3. 记录普通策略组选择、模式与 VPN 状态。在 FlClash 打开 **Tools → Backup and restore → Backup**，新导出一份完整备份。把原件留在本机安全位置，修改副本。

```powershell
adb shell pm list packages com.anthropic.claude
adb shell pm list packages com.gbox.android
adb shell pm list packages com.google.android.gms
adb shell pm list packages com.follow.clash
```

## 2. 绑定当前订阅的持久覆写

先按共用说明准备单节点专用组、DNS 和原订阅规则。手机启用应用级分流，只有 Chrome 的 UDP 在首条被拒绝。专用节点保留 UDP 转发，以便应用自身的 NTP 等请求进入该组。[规则顺序与 DNS](ROUTING.md)

历史操作修改了备份的这三处。若通过 FlClash 界面编辑，重新导出的备份也可按同样的对应关系核对。

| 备份位置 | 操作 | 验收 |
| --- | --- | --- |
| `config.json` | 读取 `currentProfileId`，按下一节修改 VPN 和核心字段 | 数值与原备份一致，普通设置保留 |
| `scripts/<script_id>.js` | 写入当前订阅使用的 Mihomo 覆写逻辑 | 生成配置的前四条规则与下方一致 |
| `database.sqlite` | 在 `scripts` 注册本地脚本，将当前 `profiles` 行的 `overwrite_type` 设为 `script`、`script_id` 设为对应值；在 `selected_map` 中仅更新专用组 | 原普通节点选择保留，`PRAGMA integrity_check` 返回 `ok` |

使用离线备份方式时，先在副本内确认 `profiles/<currentProfileId>.yaml` 存在。在同一数据库事务内写入脚本记录并更新当前订阅。JSON 与 SQLite 的 ID 可能超过 JavaScript Number 的精度，读写时使用能保留整数的解析器，如 Python 的标准 `json`。重新打包时让 `config.json` 和 `database.sqlite` 保持在 ZIP 根目录。遇到不同版本的字段结构，应先核对版本对应的实现。[FlClash 0.8.99 源码](https://github.com/chen08209/FlClash/tree/v0.8.99)

```text
AND,((PROCESS-NAME,com.android.chrome),(NETWORK,UDP)),REJECT
PROCESS-NAME,com.anthropic.claude,Claude-Fixed
PROCESS-NAME,com.gbox.android,Claude-Fixed
PROCESS-NAME,com.google.android.gms,Claude-Fixed
```

GBox 和 microG 规则会影响这两个进程的其他连接。历史连接记录分别观察到 Claude 的 API/CDN、GBox 的服务连接和 microG 的 Google 连接命中专用组。

## 3. 配置 VPN、DNS 和 IPv6

| 位置 | 本次取值 | 作用 |
| --- | --- | --- |
| `vpnProps.systemProxy` | `false` | 依赖系统 VPN 接管 |
| `vpnProps.allowBypass` | `false` | 限制应用主动绕过 VPN |
| `vpnProps.dnsHijacking` | `true` | 接管常规 DNS |
| `vpnProps.ipv6` | `false` | 不为此 VPN 配置 IPv6 地址族 |
| `patchClashConfig.mode` | `rule` | 保留国内和其他既有规则 |
| `patchClashConfig.find-process-mode` | `always` | 获取手机进程信息以匹配包名 |
| `patchClashConfig.ipv6`、`dns.ipv6` | `false` | 关闭核心 IPv6 和 AAAA 答复 |
| `patchClashConfig.tun.dns-hijack` | `any:53`、`tcp://any:53` | 接管 UDP/TCP 53 |
| `overrideDns`、`networkProps.appendSystemDns` | `false` | 使用覆写的 DNS，避免追加系统解析器 |

同时在 FlClash 中确认 **Allow LAN** 和日常使用的 **External controller** 关闭。手机上的 `find-process-mode` 使用 `always`，其生效与否看运行中的规则命中。[Android VpnService.Builder](https://developer.android.com/reference/android/net/VpnService.Builder)

## 4. 导入并检查实际连接

1. 若修改了备份副本，在 **Tools → Backup and restore → Restore → Restore all data** 选择候选 ZIP。先确认手边保有可恢复的原备份。
2. 返回 Dashboard，核对 **Rule**、普通节点选择和单节点专用组，再打开 VPN。
3. 正常打开 Claude 及关联服务。在 FlClash 的连接页核对软件包、目标、匹配规则和最终出站。普通域名、动态遥测域名都应按所属进程判断。
4. 用手机浏览器访问一个国内普通 HTTPS 页面。在连接页核对 `DIRECT`，再执行公网出口、DNS 与多 STUN 验收。[检测网站与判断标准](VERIFICATION.md)

## 5. 检查 VPN 边界并清理

- **IPv6**　在 VPN 开启时核对地址族与路由，辅以字面 IPv6 目的地址测试，例如 `adb shell ping -6 -c 1 2606:4700:4700::1111`。历史样本返回无路由，浏览器 IPv6 HTTPS 同时失败。
- **WebRTC**　历史 10 个 STUN 目标没有产生公网候选，仍返回 VPN 内部 `host` 地址。手机核心同时记录了 Chrome UDP 拒绝命中。
- **手动开关**　本次保留手动关闭 VPN 后恢复正常联网的行为。关闭 VPN 时这套分流与 DNS 保护随之停止。未进行 VPN 崩溃锁网、重启、长期熄屏或网络切换试验。
- **后台**　本次保留 FlClash 的 `deviceidle` 休眠豁免，可用 `adb shell dumpsys deviceidle whitelist` 在本机检查。厂商电池管理仍需按设备验证。
- **结束测试**　关闭临时控制接口，只移除本次创建的 ADB 转发。历史现场对应 `adb forward --remove tcp:19090`、`adb forward --remove tcp:17890` 和 `adb reverse --remove tcp:18765`。未创建转发时跳过。

## 6. 保存最终备份与回退

重新导出最终备份，检查脚本绑定、原普通选择、规则顺序、DNS/VPN 和已关闭的控制口。若需撤销，先备份现状，再用 **Restore all data** 恢复修改前的 ZIP，复查国内网页与原应用。系统休眠豁免独立于备份，如需撤销该项，使用 `adb shell dumpsys deviceidle whitelist -com.follow.clash`。

手机系统时区仍为东八区。测试覆盖了实际网络连接及短时 HTTP 复用，账户并发、长期运行和应用内定位信号不在本次验收范围内。
