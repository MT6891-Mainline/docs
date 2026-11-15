# chopin · Linux 路线

主线内核（7.3.3）+ Debian 13 + denial 桌面。这条路线是**当前在用的**。

## 从哪开始读

| 想知道什么 | 读 |
| --- | --- |
| **移植是怎么一步步走到今天的** | [`TIMELINE.md`](TIMELINE.md) |
| 开机那一刻发生了什么 | [`01-boot.md`](01-boot.md) |
| 现在能干什么、不能干什么 | 下面的状态矩阵 |
| 某个功能的细节 | 对应 `NN-*.md`；更深的原始记录在 [`archive/`](archive/) |

## 功能状态矩阵

| 功能 | 状态 | 文档 |
| --- | --- | --- |
| 启动（LK → 内核 → initramfs → systemd → denial） | ✅ 可用，**不插 USB 关机/重启会卡死**（未解） | [`01-boot.md`](01-boot.md) |
| 显示（厂商 `mediatek_v2` DRM + k10a 面板） | ✅ 可用；首次 doze 有 ~100 ms 花屏 | [`02-display.md`](02-display.md) |
| GPU（Panfrost / Mali-G77） | ⚠️ 能跑 3D，但**锁 350 MHz**、无 DVFS | [`03-gpu.md`](03-gpu.md) |
| 电源：充电/电量 | ✅ 可用（慢充，无充电泵） | [`04-power.md`](04-power.md) |
| 电源：挂起 | ❌ **放弃**（会烧屏，禁止测试） | [`04-power.md`](04-power.md) |
| 音频（CS35L41 x2） | ✅ 外放可用 | [`05-audio.md`](05-audio.md) |
| Wi-Fi（connac2 / conninfra） | ✅ 可用 | [`06-wireless.md`](06-wireless.md) |
| 蓝牙 | ✅ 可用（A2DP 需额外条件、HID 需 `CONFIG_UHID`） | [`06-wireless.md`](06-wireless.md) |
| 传感器 / 自动旋转 | ⚠️ 加速度计在 SCP 后面，AP 侧没打通 | [`07-sensors.md`](07-sensors.md) |
| 桌面（denial） | ✅ 可用（PAM 解锁有缺陷） | [`08-desktop.md`](08-desktop.md) |
| Debian rootfs | ✅ 可用 | [`09-rootfs.md`](09-rootfs.md) |
| 视频硬解（VDEC） | ❌ 无 V4L2 解码器，软解 625% CPU | [`10-media.md`](10-media.md) |
| 指纹（Goodix GF3626ZS2） | ⚠️ 采集未闭环（TA 已在用户态跑起来） | [`11-fingerprint.md`](11-fingerprint.md) |
| USB OTG / 主机模式 | ❌ 端口使能失败 | [`12-usb.md`](12-usb.md) |
| KVM（EL2） | ⚠️ 只在 MIUI 12.5 的 preloader 下可用 | [`04-power.md`](04-power.md) |
| 构建（内核基线 / DTS） | ✅ | [`13-build.md`](13-build.md) |

## 目录约定

* `NN-功能.md` —— 每个功能一篇，结构固定：现状表 → 关键结论 → 已修/未修 → 细节链接；
* `archive/` —— 原始长文（按主题写的交接记录、历史评估），只做存档，不再增长；
* 时间线在 [`TIMELINE.md`](TIMELINE.md)，按"发现问题 → 定位 → 修复"的顺序讲。
