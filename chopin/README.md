# chopin — Redmi Note 10 Pro（MT6891Z / MT6893）

代号来自甜点：chopin = 肖邦。这台机器上跑过两条完全不同的路，分开放：

| 路线 | 目录 | 状态 |
| --- | --- | --- |
| **Linux**（主线内核 + Debian + denial 桌面） | [`linux/`](linux/README.md) | ✅ 日常可用（见功能状态矩阵） |
| **UEFI / Windows on ARM** | [`uefi/`](uefi/README.md) | ❌ 探路，未跑通 |

两条路线共用同一套硬件和分区，但引导链、驱动和文档几乎不重叠——所以从
`chopin/` 往下第一刀就切在这里。

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
| `git/chopin-linux-7.3` | 内核与设备树（Linux 路线） |
| `git/chopin-support` | 固件、服务、配置 |
| `git/chopin-initramfs` | 启动 initramfs（`rootfs` / `recovery` 分支） |
| `git/docs` | 本仓库 |
