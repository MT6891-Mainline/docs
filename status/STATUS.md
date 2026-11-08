# chopin (Redmi Note 10 Pro / MT6891) 主线 Linux 移植 —— 当前状态

> 最后更新：2026-11-08
> 镜像：`out-scp-v199-733-final/boot-stockdt.img`
> 内核：`7.3.3`，树 `git/chopin-linux-7.3`（18 个按子系统的提交）

## 1. 能用什么

| 功能 | 状态 | 备注 |
| --- | --- | --- |
| 启动 | ✅ | preloader → LK → GenieZone → Linux，无 U-Boot/UEFI |
| 显示 | ✅ | **主线 `mediatek-drm`**，1080x2400；厂商 V2 仅编译不绑定 |
| 触摸 | ✅ | |
| 桌面 | ✅ | **Denial**（v0.6.2，tty1 自动登录 `kily` 后 `denial-session`） |
| Droidloom（安卓应用） | ❌ | cell 能启动到 zygote，SurfaceFlinger 因 EGL/ANGLE 崩溃循环；见 [`desktop/DROIDLOOM.md`](../desktop/DROIDLOOM.md) |
| 硬件渲染 | ✅ | panfrost (Mali-G77)，**810 MHz** |
| 音频 | ✅ | `mt6893mt6359` 声卡 |
| WiFi / 蓝牙 | ✅ | |
| 摄像头 | ❌ | 完全未移植 |
| 加速度计 | ✅ | SCP RV33 + 自建 iio-sensor-proxy，自动旋转可用 |
| 自动旋转 | ✅ | |
| **电量显示** | ✅ | BQ27Z561 真库仑计（±1%） |
| **充电** | ✅ | **PD 12V / 3.75A，约 2%/分钟** |
| 熄屏（空闲 60s） | ✅ | 唤醒不花屏 |
| **电源键熄屏** | ✅ | 按一下切背光（`chopin-powerkey`），v166 修好 PMIC 中断后可用 |
| GPU 最高频率 | ✅ | 810 MHz（350→810，**+77%**） |
| 挂起 | ❌ | 见 §4，**永久放弃**（10+ 轮修复后仍有硬件 IRQ 中止） |

## 2. 性能实测

### GPU（glmark2-es2-wayland，1080x2400 全屏）

| 频率 | 分数 | 相对 350 |
| --- | --- | --- |
| 350 MHz | 968 | — |
| **810 MHz** | **1710** | **+77%** |

836 MHz 是 SoC 上限，早期测得分 1570，但那组数据用的是"连续扫频"协议
（每次改频后的首轮系统性偏慢），不可靠；810 已经接近饱和，且余量更好。
**推荐 810 MHz**（`/etc/chopin-gpu-freq`，开机由服务应用）。

### 充电

| 项 | 值 |
| --- | --- |
| PD 合约 | 12 V / 2.25 A（DTS 声明 5V/9V/12V） |
| 输入功率 | 23 W |
| 入电池 | 15 W（4.02 V × 3.73 A） |
| 速度 | **2% / 分钟**，19%→100% 约 1 小时 |
| 电池温度 | 40-42 °C（0.17 °C/分钟） |
| 电池信息 | 学习容量 4950 mAh，健康度 99%，循环 3 次 |

## 3. 设备上的关键配置

| 路径 | 作用 |
| --- | --- |
| `/etc/chopin-gpu-freq` | GPU 频率（当前 `810000000`） |
| `/etc/chopin-charge-limit` | 输入限流覆盖（可选） |
| `/etc/chopin-charge-current` | 充电电流覆盖（可选，调低更凉） |
| swap (`/swapfile`) | 4 GB，防 gnome-shell 泄漏触发 OOM |
| `chopin-gpu-freq.service` | 开机应用 GPU 频率 |
| `chopin-charge-tune.service` | 开机 + 插拔时应用充电限流 |
| `chopin-no-suspend.desktop` | 会话内挂起抑制锁 |
| `gdm.service.d/10-wait-for-dsi.conf` | 等 DSI 连接后再启动 gdm |

## 4. 已知限制

### 挂起：放弃（2026-10-31 最终结论）

10+ 轮修复分别解决了：mtu3 状态跟踪、显示 wakelock、WiFi 唤醒事件、conninfra
与蓝牙唤醒锁、蓝牙 IRQ 武装、**PMIC 中断接错**（电源键原本完全无事件）、
电源键 release 唤醒、WiFi 扫描撞窗、永久持有唤醒锁、timesync/充电器 hard 事件。

**代价**：过程中 3 次卡死、3 次花屏；收益只有相比熄屏多的 0.3–0.5 W。

**决定**：用熄屏。细节与"如何继续"见 `chopin-docs/power/SUSPEND-ANALYSIS.md` §23。

⛔ **禁止**：`echo mem > /sys/power/state`、`rtcwake -m mem -s N`、`systemctl suspend`

### 其他

* **无 SoC 温度传感器**（主线缺 thermal 驱动），只能读电池温度
* **充电泵未移植**（33W+ 需要 BQ25970 + 约 2 万行厂商框架）
* 电量计必须用 **100 kHz** I2C（400 kHz 下 i2c0 读全 0xFF）
* 相机、指纹、GNSS 未移植

## 5. 深挖文档

| 文档 | 内容 |
| --- | --- |
| `port/GPU-PERF.md` | GPU 三个缺口的全部实验与结论 |
| `port/CHARGING.md` | 电量计与 PD 快充（含两个上游 bug） |
| `port/SUSPEND-ANALYSIS.md` | 挂起为何不可行 |
| `port/V2-DRM-AUDIT.md` | 厂商 DRM 审计 |
| `port/DRM-CLOCK-GAP.md` | 显示时钟链路 |
| `port/DEBIAN-ROOTFS.md` | rootfs 构建与 GNOME 配置 |
| `port/patches/pd-upstream/` | **两个上游补丁**（tcpci_mt6360） |
| `port/GPU-HANDOFF.md` / `port/GPU-PORT.md` | GPU 移交笔记 |

## 6. 最近的关键修复（时间序）

| 提交 | 内容 |
| --- | --- |
| `351d82f70` | MTCMOS provider 扩展到 40 个（修显示时钟） |
| `593a8d3b3` | mmsys 子设备重复绑定 |
| `1a6e01b2a` | v2 DRM ioctl/underrun 加固 |
| `8fa91bb0e` | 挂起时保留 fence 会话 |
| `c8a715ca9` | **panfrost mfg_sel 安全重锁（解锁 810MHz）** |
| `869e32df1`→revert | GPU OPP 表尝试（挂死，已回退） |
| `56ed28e5e` | DSI underrun 恢复改为重试 |
| `0f93fae6a`→revert | DVFSRC 顶档钉住（无收益，已回退） |
| `c92ef7a44` | **DTS 声明 9V/12V PD PDO** |
| `ff801f20a` | tcpci_mt6360 补 POWER_CTRL + LOOK4CONNECTION |
| `ee59e0e0e` | **tcpci_mt6360 unmask alerts（PD 真因）** |
| v183 | 内置 `ANDROID_BINDER_IPC/BINDERFS`、`NF_TABLES`（Droidloom 前置） |
| v184 | **panfrost `DRIVER_SYNCOBJ_TIMELINE`**（Denial 显式同步 / Droidloom syncobj 绑定失败的真因） |
| v185 | 内置 `DMABUF_HEAPS_SYSTEM/CMA`（`/dev/dma_heap/system`） |
| v186 | 内置 `VETH` + 全套 nftables（私有 cell 网络） |
| v187 | 内置 `EROFS_FS(+ZIP)`、`FUSE_FS`、`OVERLAY_FS`（Android 镜像格式） |

## 7. 镜像清单

| 镜像 | 说明 |
| --- | --- |
| **`out-scp-v187-rootfs`** | ✅ **当前推荐**（Denial + Droidloom 内核前置：binderfs/nftables/veth/DMA-BUF 堆/EROFS/syncobj） |
| `out-scp-v184`…`v186` | ✅ 中间版本（依次修 syncobj / DMA-BUF 堆 / veth+nftables） |
| `out-scp-v154-rootfs` | ✅ PD 快充 + 全部修复（旧基线） |
| `out-scp-v151-rootfs` | ✅ 真库仑计可用 |
| `out-scp-v143-rootfs` | ✅ 熄屏恢复不放弃 |
| `out-scp-v139-rootfs` | ✅ GPU 810MHz 基线（无后续修复） |
| `out-scp-v130-rootfs` | ✅ 传感器稳定版 |
| `out-scp-v135/136/140/141/142` | ⛔ 挂死或花屏，勿用 |
