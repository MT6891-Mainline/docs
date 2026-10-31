# chopin 主线 PMIC IRQ / 电源键笔记（2026-10-23）

## 现象

- 音量键正常（`mtk-kpd`，IRQ 263）。
- 电源键无任何 input 事件：
  - `mtk-pmic-keys` 的 4 个 IRQ 计数始终 0；
  - 父中断 `mt6358-irq`（GIC 254）始终 0；
  - 插拔充电器也不能让 GIC 254 计数增加。

## 已尝试

- `mtk-pmic-keys` 已 `device_init_wakeup()`，`power/wakeup=enabled`。
- 对照原厂把 `mt6358-irq.c` 的 MT6359 子中断使能改成
  **CON0+0x2 SET / CON0+0x4 CLR**（LDO 例外），仍无效。
- `mt6359-pmic` 的 `interrupts = <0xde ...>` 映射到 GIC 254，
  `mt6358-irq` 已 request 该 IRQ，但硬件从未拉高。

## 判断

不是 `mtk-pmic-keys` 或 `KEY_POWER` 映射问题，而是 **主线 pwrap 没有把
PMIC INT 线配置/路由到 GIC**。原厂 `pwrap` 会初始化 PMIC 中断极性/mask，
主线 `mtk-pmic-wrap.c` 只处理 starvation/exception IRQ，没有这套 PMIC INT
初始化。

## 结论

- 电源键暂不可用，用 `systemctl suspend` 或 GNOME 电源菜单。
- 继续修需要移植 vendor pwrap 的 PMIC INT 配置（极性、mask、GIC 路由），
  和 SPMI 一样是独立的一块工作。

## 相关提交

- `c9ad69e mfd: mt6358-irq: use MT6359 SET/CLR IRQ accessors (WIP)`
- `f0b281c input: mtk-pmic-keys: mark the device wakeup capable`
