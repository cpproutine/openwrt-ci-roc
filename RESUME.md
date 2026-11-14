# 项目状态与续接说明（RESUME）

> 换电脑后从这里接手：所有需要的东西都在本仓库，不依赖原来那台机器。

## 当前状态（2026-11-14 15:22 GMT+8）

| 项目 | 状态 |
|---|---|
| Fork 仓库 | `cpproutine/openwrt-ci-roc`（master） |
| 定制提交 | `d3d5149` — 5 处文件已就位 |
| 云编译 | **Run 43564682402 进行中**（14:06 触发，约 1.5–2 小时） |
| 构建日志 | https://github.com/cpproutine/openwrt-ci-roc/actions/runs/43564682402 |
| 产物位置 | Releases → Tag **`AX3600-Stock`**（构建成功后才出现） |
| 路由器 | 原厂 1.0.17，SSH 已通（root / admin，需 `-o HostKeyAlgorithms=+ssh-rsa`） |

## 换电脑后要做什么

1. **看构建结果**：打开上面 Actions 链接；成功后到 Releases 页下载 `AX3600-Stock` 的 assets；
2. **核对固件**：
   - 文件名含 `xiaomi_ax3600-stock`（stock 原厂分区布局）；
   - `sha256sum` 校验；Windows 用 `certutil -hashfile <文件> SHA256`；
   - 打开 `*.manifest` 确认**没有** Xray、SmartDNS、samba4、docker、openclash 等方案外包；
3. **按文档执行**：
   - 刷机前准备 → `docs/刷机准备清单.md`
   - 降级/SSH（已完成可跳过）→ `docs/AX3600_降级破解SSH步骤.md`
   - 构建与上传流程 → `docs/README-使用说明.md`

## 重新触发构建（如需）

本仓库 `.github/workflows/AX3600-Stock.yml` 是唯一入口，参数已固定：

- 源码：`laipeng668/openwrt-6.x` 分支 `25.12-nss`
- 固定提交：`6c363ced06ee6dc0c49092a8da087e1dc45f17c9`（严格可复现）
- 设备配置：`configs/AX3600-Stock.config`（仅 `xiaomi_ax3600-stock`）
- 包配置：`configs/General.config`（Passwall 仅 sing-box + rtp2httpd）

在 Actions 页面点 "Run workflow" 即可，无需本机环境。

## 关键参数备忘

| 项 | 值 |
|---|---|
| 原厂 SSH | `192.168.31.1`，root / admin |
| 定制固件管理地址 | **`http://192.168.2.1`**（CI 脚本改写默认 IP） |
| 首启 Wi-Fi | SSID `AX3600-Setup` / 密码 `AX3600-Setup-9pQ7xM2k`（登录后立即改） |
| 官方 1.0.17 固件 | `http://cdn.cnbj1.fds.api.mi-img.com/xiaoqiang/rom/r3600/miwifi_r3600_firmware_5da25_1.0.17.bin`（sha256 `dfbe347339903f2ae348be2e4b6434cc77dfd2247d970d7be7b3eac76731dd40`） |
| NSS 配置 | 512MB 内存档（MEDIUM）、固件 11.4、Wi-Fi 卸载开 / Mesh 关 |

## 换机不带来的东西（需要重新准备）

- 本机的 `libwrt/` 源码克隆、`ci-fork/` 克隆（都可从 GitHub 重新获取）；
- 已下载的 1.0.17 固件（按上表链接重下，几秒钟）；
- GitHub 连接器授权（新机器需重新授权）；
- PAT 令牌（建议重新生成，别跨机器复制）。
