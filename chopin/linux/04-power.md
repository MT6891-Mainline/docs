# 04 · 电源：充电、电池、挂起、KVM

## 充电与电量

| 项 | 状态 | 说明 |
| --- | --- | --- |
| 充电 | ✅ | MT6360（i2c5），SDP 默认 500 mA，DCP/CDP 可到 2000 mA |
| 电量计 | ✅ | BQ27Z561 在 **i2c0，必须 100 kHz**（频率不对读数全是 `65535`） |
| 充电泵（33W/67W） | ❌ | 需要 BQ25970 + 约 2 万行厂商框架，不移植 |

细节：[`archive/CHARGING.md`](archive/CHARGING.md)、[`archive/CHARGING-STATUS.md`](archive/CHARGING-STATUS.md)。

## 挂起 —— ⛔ 已放弃

> **禁止在这台设备上继续测试挂起。** 反复触发 doze 会**烧屏**（AMOLED 残留）。

原因：`mediatek_v2` 的 doze/suspend 路径会在面板已关闭时释放 fence，导致
`DDP_COMPONENT_DSI0: buffer underrun` 和花屏。修复需要重写厂商 DRM 的电源时序，
代价远大于收益（设备定位是"能用的桌面 Linux"，不是省电）。

完整 10+ 轮记录：[`archive/SUSPEND-ANALYSIS.md`](archive/SUSPEND-ANALYSIS.md)、
[`archive/SUSPEND-NOTES.md`](archive/SUSPEND-NOTES.md)。

落地措施：`logind.conf.d/10-chopin-no-suspend.conf` 设 `IdleAction=ignore`、
`HandleSuspendKey/HibernateKey/LidSwitch/PowerKey=ignore`；**不要**用 mask
`sleep.target` 的方式（会让桌面会话立刻自杀）。

## KVM（EL2）

能用，但**取决于 preloader**：[`archive/KVM-EL2.md`](archive/KVM-EL2.md)。

| preloader | 内核启动级别 | KVM |
| --- | --- | --- |
| MIUI 12.5 | EL2（VHE） | ✅ |
| MIUI 14 / OS2.0 | EL1 | ❌ `HYP mode not available` |

## PMIC / 电源键

音量键走 `mtk-kpd`（IRQ 263）正常；电源键上报 `KEY_POWER` 正常
（`event0` 能抓到完整的按下/松开）。笔记：[`archive/PMIC-IRQ-NOTES.md`](archive/PMIC-IRQ-NOTES.md)。
