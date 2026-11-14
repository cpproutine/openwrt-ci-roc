# AX3600 云编译使用说明（GitHub Actions）

按 `AX3600_LibWrt_NSS_完整方案.md` 生成的一套云编译定制文件。本机无 Linux/WSL/Docker，
因此采用 LibWrt 作者推荐的云编译仓库 `laipeng668/openwrt-ci-roc` 方案构建。

## 需要上传到 Fork 的文件对照表

⚠️ 不能把整个 `openwrt-ci/` 目录直接放入仓库，必须按下列路径放置：

| 本地文件 | 上传到 Fork 的位置 | 说明 |
|---|---|---|
| `configs/AX3600-Stock.config` | `configs/AX3600-Stock.config` | 设备/NSS 配置（新增文件） |
| `configs/General.config` | `configs/General.config` | **覆盖替换**原文件（精简组件、去掉 Xray/SmartDNS） |
| `workflows/AX3600-Stock.yml` | `.github/workflows/AX3600-Stock.yml` | 新增工作流（重命名为 `AX3600-Stock.yml`） |
| `workflows/Build-OpenWrt.yml` | `.github/workflows/Build-OpenWrt.yml` | **覆盖替换**原文件（新增 repo_commit 固定提交参数，与上游原版仅 18 行差异，已 diff 核验） |
| `Roc-script.sh-追加片段.txt` 内容 | 追加到 `scripts/Roc-script.sh` 末尾 | 首启自动开 Wi-Fi + chmod 0755（方案第十二节） |

## 操作步骤

1. 登录 GitHub，Fork `https://github.com/laipeng668/openwrt-ci-roc` 到自己账号。
2. 在 Fork 中按上表上传/替换 5 处文件（网页端 Add file → Upload files 即可）。
   - 编辑 `scripts/Roc-script.sh`：点铅笔图标，把追加片段粘贴到文件最末尾，Commit。
3. 进入 Fork 的 **Actions** 页，若提示工作流被禁用，点 **Enable workflows**。
4. 左侧选择 **AX3600-Stock** → **Run workflow** → 确认运行。
5. 编译约 1–2 小时。完成后：
   - 产物在仓库 **Releases → Tag `AX3600-Stock`** 下载；
   - Release 说明里记录了本次源码 commit hash（已固定为 `6c363ced`，与方案基线一致，构建严格可复现）。
6. 下载后核对（方案第八节）：
   - 固件文件名含 `xiaomi_ax3600-stock`（原厂分区布局）；
   - `.config` 与 manifest 一并保存归档；
   - 核对 SHA256 再刷机。

## 首启管理地址说明

- 云编译脚本（Roc-script.sh）会把默认 IP 从 `192.168.1.1` 改为 `192.168.2.1`，
  因此**首启应访问 http://192.168.2.1**（默认密码 none）。
- 方案文档中写的是 OpenWrt 默认的 `192.168.1.1`，该差异由 CI 脚本导致，属已知且符合预期；
  如需改回 `192.168.1.1`，可在 Fork 中删除 Roc-script.sh 里 `sed 's/192.168.1.1/192.168.2.1/g'`
  那一行（不建议：作者配套的说明与固件版本页署名均按 192.168.2.1 预期）。

## 与方案的对应关系

- **底座**：`laipeng668/openwrt-6.x`（LibWrt 同作者的镜像仓库）`25.12-nss` 分支，
  与 `LiBwrt/LibWrt` 内容一致；**构建固定在提交 `6c363ced`**（方案钉住的基线）。
- **设备**：仅 `xiaomi_ax3600-stock`，并显式排除 `xiaomi_ax3600`（自定义 U-Boot 布局）防误刷。
- **NSS 内存档**：`CONFIG_NSS_MEM_PROFILE_MEDIUM=y`（覆盖 ipq807x 默认的 1GB HIGH 档，
  这是方案第三节强调的关键点）。
- **ath11k**：512M 内存档；Wi-Fi NSS 卸载默认开启；NSS Mesh 显式关闭。
- **NSS 固件**：11.4（`CONFIG_NSS_FIRMWARE_VERSION_11_4=y`）。
- **Flow Offloading / Packet Steering**：显式不编译 `kmod-nft-flow-offload`、
  `kmod-ipt-flowoffload`、`packet-steering`，卸载统一走 NSS ECM。
- **代理**：Passwall(nftables) + 仅 sing-box；Xray、Hysteria、SS/SSR、Trojan-Plus、
  tuic、SmartDNS 等其余核心/插件全部显式关闭（SmartDNS/Haproxy/SS-rust/SSR/Simple-Obfs/
  V2ray_Plugin 在 aarch64 上为 Passwall 包内默认 y，必须显式关闭——已逐项核对 Makefile 默认值）；
  Passwall2/OpenClash 不编译。
- **IPTV**：`rtp2httpd` + `luci-app-rtp2httpd`（默认端口 5140）。
- **大件裁剪**：samba4、frp、ddns、upnp、wol、diskman、hd-idle、3cat、watchcat、
  vlmcsd、acme、wireguard、coremark、zram 等全部不编译（原版 General.config 均有）。
- 注：autocore/luci-app-cpufreq/automount/e2fsprogs 等是 qualcommax 目标 DEFAULT_PACKAGES，
  底座强制自带，`.config` 无法排除。

## 与方案的一处偏差（已确认原因）

1. **sing-box-tiny 不存在**：ImmortalWrt `openwrt-25.12` feeds 和 Passwall 的
   `openwrt-passwall-packages` 中均无 `sing-box-tiny` 包（已验证 404），
   故使用标准 `sing-box`。功能裁剪方案第四节的建议在云编译体系下不可行，
   如确需 tiny 版只能本地编译自行裁剪。

## 刷机提醒（方案第八、十二节）

- 固件镜像不是原厂网页升级包；首次刷入走 OpenWrt AX3600 ToH 的
  SSH + initramfs + UBI 流程。
- 首启 Wi-Fi SSID `AX3600-Setup` / 密码 `AX3600-Setup-9pQ7xM2k` 仅用于过渡，
  登录后立即改正式 SSID 与强密码。
- 刷写开始后 SSH 断开属正常，切勿断电或重复执行刷写命令。
- 救砖请准备 USB 转千兆网卡或串口。
