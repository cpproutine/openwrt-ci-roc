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

## 二、最终固件核对结果（v2 瘦身版）

- `factory.ubi` = **33,947,648 B（32.38 MiB）**，stock 分区容量 35.75 MiB → **余量 +3.38 MiB，可刷入** ✅
- `sysupgrade.bin` = 32,604,439 B（31.09 MiB）✅
- sing-box 二进制 55.6 → 35.6 MB；rootfs 未压缩 110 → 92.6 MB；squashfs 25.66 MB
- buildinfo 确认：`CLASH_API/QUIC/UTLS`=y；`ACME/GVISOR/TAILSCALE/WIREGUARD`=not set
- NSS：512MB 内存档（MEDIUM）、固件 11.4、Wi-Fi 卸载开 / Mesh 关；无 flow offload / packet steering
- 包总数 298，方案外组件（Xray/SmartDNS/samba4/docker/openclash…）**零命中**
- `/etc/uci-defaults/99-wifi-firstboot` 已确认在镜像内（首启 SSID `AX3600-Setup`）
- SHA256 详见 `docs/SHA256SUMS.txt`（刷机前务必校验）

## 三、换机后的行动清单（按顺序）

1. **下载固件**：Releases → Tag `AX3600-Stock` → 只需 `*-squashfs-factory.ubi` 这一个文件
   → `certutil -hashfile <文件> SHA256` 与 `docs/SHA256SUMS.txt` 比对
2. **连接路由器**：连上原厂 Wi-Fi（192.168.31.1），SSH 已通：
   ```
   ssh -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa root@192.168.31.1
   ```
   密码 `admin`（若重置过则按 `docs/AX3600_降级破解SSH步骤.md` 重走一遍）
3. **⚠️ 先备份分区（必做，5 分钟）**：
   ```sh
   cat /proc/mtd
   mkdir -p /tmp/bak
   nanddump -f /tmp/bak/mtd8_ART.img /dev/mtd8
   nanddump -f /tmp/bak/mtd9_bdata.img /dev/mtd9
   ```
   拉回电脑（注意 `-O`，老 dropbear 不支持 SFTP）：
   ```
   scp -O -o HostKeyAlgorithms=+ssh-rsa root@192.168.31.1:/tmp/bak/*.img .
   ```
4. **上传固件**（在 PowerShell，电脑连路由器 Wi-Fi）：
   ```
   scp -O -o HostKeyAlgorithms=+ssh-rsa libwrt-...-factory.ubi root@192.168.31.1:/tmp/
   ```
5. **刷写（stock 布局专用，保留原厂回退，不用 wiki 的 sysupgrade -n）**：
   ```sh
   nvram get flag_boot_rootfs      # 0 → 刷对侧 mtd13；1 → 刷对侧 mtd12
   cat /proc/mtd                   # 人工确认 rootfs=12 / rootfs_1=13

   # 若返回 0：
   ubiformat /dev/mtd13 -y -f /tmp/libwrt-qualcommax-ipq807x-xiaomi_ax3600-stock-squashfs-factory.ubi -s 2048 -O 2048
   nvram set flag_boot_rootfs=1; nvram set flag_last_success=1; nvram set flag_boot_success=1; nvram commit

   # 若返回 1：mtd12 与 flag_boot_rootfs=0，命令同理
   reboot
   ```
6. **首启**：管理地址 **http://192.168.2.1**（不是 1.1），Wi-Fi SSID `AX3600-Setup` / 密码 `AX3600-Setup-9pQ7xM2k`（登录后立即改）
7. **验收**（对照 `docs/固件核对报告.md`）：
   - `ubus call system board` 确认版本
   - `luci` → NSS 状态页确认 NSS 加速生效（`/usr/sbin/nssinfo` 或 `echo "dump" > /proc/sys/kernel/qca-nss-drv/...`）
   - 网口测速对比原厂；Passwall 配 sing-box 节点测代理；rtp2httpd 组播转 HTTP（端口 5140）
   - 出问题回退：切回 flag_boot_rootfs 原值 reboot 即回到原厂

## 四、关键参数备忘

| 项 | 值 |
|---|---|
| 原厂 SSH | `192.168.31.1`，root / admin（需 ssh-rsa 兼容参数，见上） |
| 定制固件管理地址 | **`http://192.168.2.1`** |
| 首启 Wi-Fi | SSID `AX3600-Setup` / 密码 `AX3600-Setup-9pQ7xM2k` |
| overlay 剩余 | 约 3.3 MiB —— 改配置够用，**后期装大包装不下**，组件已编译期固化 |
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
