# 项目状态与续接说明（RESUME）

> 换电脑后从这里接手：所有需要的东西都在本仓库，不依赖原来那台机器。
> 最后更新：2026-11-14 18:46 GMT+8

## 一、当前状态：固件已构建完成并核对通过 ✅

| 项目 | 状态 |
|---|---|
| Fork 仓库 | `cpproutine/openwrt-ci-roc`（master） |
| 最终定制提交 | `03df396` — 瘦身版配置（关闭 sing-box GVISOR/TAILSCALE/ACME/WIREGUARD） |
| 成功构建 | Run 43574606314（28 分 56 秒）+ 首次 Run 43564682402（1h39m） |
| 源码 | `laipeng668/openwrt-6.x` 分支 `25.12-nss`，固定提交 `6c363ced06ee6dc0c49092a8da087e1dc45f17c9` |
| **固件下载** | https://github.com/cpproutine/openwrt-ci-roc/releases/tag/AX3600-Stock （公开仓库，无需登录） |
| 路由器 | 原厂 1.0.17，**SSH 已破解**（root / admin） |
| 路由器分区备份 | **尚未执行**（见第三节，开刷前必须做） |

## 二、固件核对与深度验证结果（全部通过 ✅）

**尺寸（v2 瘦身版）**：`factory.ubi` 32.38 MiB（分区 35.75 MiB，余量 +3.38 MiB 可刷）；`sysupgrade.bin` 31.09 MiB；
sing-box 35.6 MB（已关 GVISOR/TAILSCALE/ACME/WIREGUARD）；NSS 512M 档 + 固件 11.4 + 卸载开/Mesh 关；
包 298 个，方案外组件零命中。SHA256 见 `docs/SHA256SUMS.txt`。

**结构级深检（2026-11-14 18:50 追加）：**

| 检查项 | 结果 |
|---|---|
| UBI 结构 | 259/259 PEB 全部有效（EC/VID header 完整） |
| kernel 卷（PEB2–46） | FIT 结构完整；kernel-1（gzip/arm64/5.64MB）**CRC32+SHA1 双哈希匹配**；fdt-1 DTB magic 正确、双哈希匹配 |
| 内核可解压 | 14.16MB，`Linux version 6.12.116`，与 rootfs `/lib/modules/6.12.116`（149 个 .ko）**完全匹配** |
| DTB 设备 | `xiaomi,ax3600-stock` / 型号 "AX3600 (stock layout)" ✅ |
| squashfs | 完整解包 1843 文件、176 目录**零错误** |
| 二进制架构 | busybox / sing-box(35.6MB) / rtp2httpd / kmodloader 全部 **AArch64 ELF** ✅ |
| NSS | ecm.ko、qca-nss-drv.ko、qca-nss-dp.ko 等模块在位；`99-qca-nss-ecm`、`992_set-nss-load.sh`、`993_set-ecm-conntrack.sh` 首启脚本在位 |
| 首启 Wi-Fi | `99-wifi-firstboot` 在位 ✅ |
| sysupgrade 元数据 | `supported_devices:["xiaomi,ax3600-stock"]`、`revision: r0-6c363ce`（与固定提交一致）✅ |
| kernel 卷容量 | FIT 5,692,032 B vs 卷容量 5,713,920 B，余量 21 KB（紧但够） |

**结论：镜像结构、内核、rootfs、二进制、元数据自洽完整，具备真实刷入和启动条件。**
剩余无法离线验证的只有两点：无线校准加载（依赖本机 ART）和 NSS 加速实测——首次开机验收时确认。

## 三、换机后的行动清单（按顺序）

1. **下载固件**：Releases → Tag `AX3600-Stock` → 只需 `*-squashfs-factory.ubi` 这一个文件
   → `certutil -hashfile <文件> SHA256` 与 `docs/SHA256SUMS.txt` 比对
2. **连接路由器**：连上原厂 Wi-Fi（192.168.31.1），SSH 已通：
   ```
   ssh -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa root@192.168.31.1
   ```
   密码 `admin`（若重置过则按 `docs/AX3600_降级破解SSH步骤.md` 重走一遍）
3. **⚠️ 全分区备份（必做，约 15 分钟，这是"官方救砖都救不回来"场景的唯一保险）**：
   原厂 SSH 里逐分区流式备份到电脑（`cmd /c` 保证二进制重定向安全，**不要在 PowerShell 原生 `>`**）：
   ```powershell
   cmd /c "ssh -o HostKeyAlgorithms=+ssh-rsa root@192.168.31.1 cat /proc/mtd"   # 先记录分区表
   cmd /c "ssh -o HostKeyAlgorithms=+ssh-rsa root@192.168.31.1 dd if=/dev/mtd8 bs=131072" > mtd8_ART.img
   cmd /c "ssh -o HostKeyAlgorithms=+ssh-rsa root@192.168.31.1 dd if=/dev/mtd9 bs=131072" > mtd9_bdata.img
   cmd /c "ssh -o HostKeyAlgorithms=+ssh-rsa root@192.168.31.1 dd if=/dev/mtd10 bs=131072" > mtd10_APPSBL.img
   # ……对 /proc/mtd 每个分区各来一条（尤其 rootfs / rootfs_1）；dd 只读零风险
   # 校验：电脑侧 certutil -hashfile <文件> SHA256 对比路由器侧 sha256sum /dev/mtdN
   ```
   备份存两块介质。**特别原因**：官方修复工具**不覆盖 `0:ART`（无线校准）和 `0:bdata`（SN/MAC）**，这两个分区损坏后官方救砖也无能为力，只有备份能救。
   恢复手段矩阵详见 `docs/刷机准备清单.md` 第 6.3 节（nvram 切回 / mtd write / 官方修复 / ART 恢复 / UART 五级）。
4. **上传固件**（在 PowerShell，电脑连路由器 Wi-Fi）：
   ```
   scp -O -o HostKeyAlgorithms=+ssh-rsa libwrt-...-factory.ubi root@192.168.31.1:/tmp/
   ```
5. **刷写（stock 布局专用，保留原厂回退，不用 wiki 的 sysupgrade -n）**：
   ```sh
   # 先记录原值（回退用，务必抄下来）
   nvram get flag_boot_rootfs; nvram get flag_last_success
   nvram get flag_boot_success; nvram get flag_try_sys1_failed; nvram get flag_try_sys2_failed

   # 确认分区索引（人工核对 rootfs=12 / rootfs_1=13，mtd 编号以实际输出为准）
   cat /proc/mtd

   # 若 flag_boot_rootfs=0 → 刷对侧 mtd13（全部 && 链接：ubiformat 失败则不会写任何启动标志）
   ubiformat /dev/mtd13 -y -f /tmp/libwrt-qualcommax-ipq807x-xiaomi_ax3600-stock-squashfs-factory.ubi -s 2048 -O 2048 \
     && nvram set flag_try_sys2_failed=0 \
     && nvram set flag_boot_rootfs=1 \
     && nvram set flag_last_success=1 \
     && nvram set flag_boot_success=0 \
     && nvram commit \
     && reboot

   # 若 flag_boot_rootfs=1 → 刷对侧 mtd12（flag_try_sys1_failed=0，其余标志置 0，同理）
   ubiformat /dev/mtd12 -y -f /tmp/libwrt-qualcommax-ipq807x-xiaomi_ax3600-stock-squashfs-factory.ubi -s 2048 -O 2048 \
     && nvram set flag_try_sys1_failed=0 \
     && nvram set flag_boot_rootfs=0 \
     && nvram set flag_last_success=0 \
     && nvram set flag_boot_success=0 \
     && nvram commit \
     && reboot
   ```
   > 标志设置与 LibWrt `platform.sh` 的 stock 变体逐项一致（该代码升级时也是重置 `flag_boot_success=0` + 重置 try_sys 失败标志）。
   > **不要预先把 flag_boot_success 设为 1**：固件内 `etc/init.d/bootcount` 会在新系统**真正启动成功后**自己 `fw_setenv flag_boot_success 1`——提前置 1 会伪造成功状态，削弱 U-Boot 对失败分区的回退重试。
6. **首启**：管理地址 **http://192.168.2.1**（不是 1.1），Wi-Fi SSID `AX3600-Setup` / 密码 `AX3600-Setup-9pQ7xM2k`（登录后立即改）
7. **验收**（对照 `docs/固件核对报告.md`）：
   - `ubus call system board` 确认版本、`cat /proc/mtd` 记录实际分区表
   - **overlay 实测**（不要用估算值）：`df -h /overlay` + `ubinfo -a` + `mount | grep overlay`
   - **NSS 加速确认**（固件内实际存在的工具，镜像已验证）：`lsmod | grep -E 'qca-nss-drv|ecm'`（两模块必须 loaded）+ `nss_stats`（输出 NSS 统计）+ LuCI NSS 状态页
   - 网口测速对比原厂；Passwall 配 sing-box 节点测代理；rtp2httpd 组播转 HTTP（端口 5140）
   - 回退（按所在系统选命令）：
     - 仍在**原厂**系统：`nvram set flag_boot_rootfs=<记录的原值>; nvram commit; reboot`
     - 已进入 **OpenWrt**（LibWrt 内）：`fw_setenv flag_boot_rootfs <原值>; fw_setenv flag_last_success <原值>; fw_setenv flag_boot_success 1; reboot`
     - 都进不去：官方修复工具（网线）或 UART——见 `docs/刷机准备清单.md` 第 6.3 节五级矩阵

## 四、关键参数备忘

| 项 | 值 |
|---|---|
| 原厂 SSH | `192.168.31.1`，root / admin（需 ssh-rsa 兼容参数，见上） |
| 定制固件管理地址 | **`http://192.168.2.1`** |
| 首启 Wi-Fi | SSID `AX3600-Setup` / 密码 `AX3600-Setup-9pQ7xM2k` |
| overlay 剩余 | **估算约 3 MiB**（分区 35.75 − 镜像 32.38 − 坏块预留）；**首启后以 `df -h /overlay` 和 `ubinfo -a` 实测为准** —— 改配置够用，**后期装大包装不下**，组件已编译期固化 |
| rtp2httpd | 端口 5140，`/status` `/playlist.m3u` `/player` |
| 官方 1.0.17 固件 | `http://cdn.cnbj1.fds.api.mi-img.com/xiaoqiang/rom/r3600/miwifi_r3600_firmware_5da25_1.0.17.bin`（sha256 `dfbe3473…`） |
| 救砖 | 小米官方修复工具（只认官方签名固件）+ 网线 → 这正是你要换网线机器的原因 |

## 五、重新触发构建（如需再改配置）

Actions 页面 → `AX3600-Stock` workflow → Run workflow。参数已全部固定在 `.github/workflows/AX3600-Stock.yml`。
改包配置只动 `configs/General.config`；缓存键包含配置 hash，但会自动降级匹配旧工具链（改几个包约 30–50 分钟）。
注意：不要改 `AX3600-Stock.config` 文件名（缓存 scope 一部分）；Release 会覆盖旧 Tag 的 assets。

## 六、换机不带来的东西（需重新准备）

- 本机 `libwrt/`、`ci-fork/` 克隆（可从 GitHub 重取，非必需）
- GitHub 连接器授权（新机器重新连）
- PAT 令牌 `~/.ghtoken_ci`：**建议先在旧机撤销，新机器重新生成**（新任务只需 Actions/Contents 读写）
- 1.0.17 官方固件本地副本（按上表直链重下）

## 七、文档索引

| 文档 | 内容 |
|---|---|
| `docs/固件核对报告.md` | 最终核对结论、全量配置开关、SHA256 |
| `docs/刷机准备清单.md` | 硬件/软件清单、Wi-Fi-only 风险 |
| `docs/AX3600_降级破解SSH步骤.md` | 降级+漏洞开 SSH（已完成，仅回退/重置后参考） |
| `docs/README-使用说明.md` | 构建流程与文件路径映射 |
| `docs/AX3600-Stock.config` | 设备配置留档 |
