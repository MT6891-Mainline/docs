# chopin — Redmi Note 10 Pro（MT6891Z / MT6893）

把 Android 厂商内核换成主线 Linux 的记录。代号来自甜点：chopin = 肖邦。

```mermaid
graph LR
  A[主线内核 7.3.3] --> B[Debian 13 rootfs]
  B --> C[denial 桌面]
  C --> D[日常使用]
```

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
| 触摸（Novatek NT36672E） | ✅ 可用 | [`01-boot.md`](01-boot.md) 固件一节 |
| 桌面（denial） | ✅ 可用（PAM 解锁有缺陷） | [`08-desktop.md`](08-desktop.md) |
| Debian rootfs | ✅ 可用 | [`09-rootfs.md`](09-rootfs.md) |
| 视频硬解（VDEC） | ❌ 无 V4L2 解码器，软解 625% CPU | [`10-media.md`](10-media.md) |
| 指纹（Goodix GF3626ZS2） | ⚠️ 采集未闭环（TA 已在用户态跑起来） | [`11-fingerprint.md`](11-fingerprint.md) |
| USB OTG / 主机模式 | ❌ 端口使能失败 | [`12-usb.md`](12-usb.md) |
| KVM（EL2） | ⚠️ 只在 MIUI 12.5 的 preloader 下可用 | [`04-power.md`](04-power.md) |
| 构建（内核基线 / DTS） | ✅ | [`13-build.md`](13-build.md) |
| Windows on ARM | ❌ 支线，未成主线 | [`14-windows-on-arm.md`](14-windows-on-arm.md) |

## 硬件速查

| 部件 | 型号 | 备注 |
| --- | --- | --- |
| SoC | MT6891Z / MT6893 | 天玑 1100 |
| 存储 | UFS，`sdc` = LUN2，rootfs 在 `/dev/sdc75` | nvdata `/dev/sdc6`，super `/dev/sdc61` |
| 显示 | 10.5" AMOLED，DSI `k10a_36_02_0a_dsc_vdo` | 厂商 `mediatek_v2` DRM 栈 |
| 触摸 | Novatek NT36672E（SPI） | 固件在 `/lib/firmware` |
| 指纹 | Goodix GF3626ZS2（SPI） | 厂商 TA 在用户态跑 |
| 充电 | MT6360（i2c5）+ BQ27Z561 电量计（i2c0） | 电量计必须 100 kHz |
| 音频 | CS35L41 x2（i2c6 0x40/0x42）+ MT6359 codec | 无 3.5mm 口 |
| 无线 | connac2（conninfra） | Wi-Fi + BT 共用 |
| 传感器 | ICM40607（挂在 SCP 后面） | AP 总线上看不到 |

## 相关仓库

| 仓库 | 内容 |
| --- | --- |
| `git/chopin-linux-7.3` | 内核与设备树 |
| `git/chopin-support` | 固件、服务、配置 |
| `git/chopin-initramfs` | 启动 initramfs（`rootfs` / `recovery` 两个分支） |
