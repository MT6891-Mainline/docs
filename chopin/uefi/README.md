# chopin · UEFI / Windows on ARM

**探路性质，未跑通。** 文档只有两篇，都在 [`archive/`](archive/)。

## 状态

| 项 | 内容 |
| --- | --- |
| 引导方案 | Mu-Silicium UEFI（参考 `MT6789Pkg`） |
| 固件基线 | r81 |
| 结果 | ❌ 未跑通 |
| 与 Linux 路线的关系 | 完全独立：UEFI 需要另建一套固件工程（ACPI/SMBIOS/DT 转换），两条链的驱动不共用 |

## 为什么搁置

Android 侧每修一个驱动（显示、电源、USB），在 UEFI 路线上都要再实现一遍；
投入产出比不划算。主线仍然是 Debian + denial 桌面。

## 文档

* [`archive/UEFI-PORT.md`](archive/UEFI-PORT.md) — UEFI 移植记录（Mu-Silicium，参考 MT6789Pkg）
* [`archive/CHOPIN-WOA-HANDOFF.md`](archive/CHOPIN-WOA-HANDOFF.md) — WoA 交接（固件基线 r81）

> 早期这些文档放在 `archive/` 顶层、和 Linux 的文档混在一起；现在按路线分开。
