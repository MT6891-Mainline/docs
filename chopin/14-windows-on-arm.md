# 14 · Windows on ARM（支线）

探路性质，**未成为主线**（主线仍是 Debian + denial）。

| 项 | 内容 |
| --- | --- |
| 引导方案 | Mu-Silicium UEFI（参考 `MT6789Pkg`） |
| 固件基线 | r81 |
| 状态 | ❌ 未跑通 |

文档：

* [`archive/UEFI-PORT.md`](archive/UEFI-PORT.md) — UEFI 移植记录
* [`archive/CHOPIN-WOA-HANDOFF.md`](archive/CHOPIN-WOA-HANDOFF.md) — WoA 交接

## 为什么先搁置

UEFI 需要一整套与 Android boot 链并存的固件工程（ACPI、SMBIOS、DT 转换），
而 Android 侧每修一个驱动（显示、电源、USB）都要在两条引导路径上各维护一遍。
投入产出比不划算。
