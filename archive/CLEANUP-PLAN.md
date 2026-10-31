# 历史整理计划

> **状态（已执行一部分）**：内核仓库 `git/chopin-linux-7.2.8` 的**历史整理已完成** ——
> 121 个迭代提交压成 10 个逻辑提交，`git diff` 为空、tree 哈希不变。
> 回退点 tag `chopin-v2-working`。详见 `V2-HANDOVER.md` §0.12。
>
> **尚未执行**（刻意推迟）：
> - **删插桩**：`V2-HANDOVER.md` §0.11 的 120 Hz 问题未解决，`CHOPIN-*` 日志还要用。
>   删它会改变 tree，届时需要新的对照 tag，不能再靠 tree 哈希验证。
> - **port 仓库整理**（本文档 §5 那一段）。

## 现状

- 内核仓库 `git/chopin-linux-7.2.8`：数十个迭代提交（`ringfix1..88` 的修补、探索性改动、
  v2 导入过程中的机械化改动混在一起）；
- port 仓库：过程日志类提交较多。

## 整理目标（一次到位，不再反复）

按**功能单元**重组为少量提交，每个提交都是可编译、可讲述的：

| 目标提交 | 内容 |
| --- | --- |
| 1. `arm64: dts: mediatek: add chopin board tree` | `arch/arm64/kernel/chopin-board.dts`（含 disp 节点覆盖、panel、mipi_tx）|
| 2. `soc: mediatek: mmsys/mutex support for MT6893` | `mt6893-mmsys.h` + `mtk-mmsys.c`/`mtk-mutex.c` 数据（若 v2 路径不再需要，则从提交中剔除）|
| 3. `clk/pinctrl/power: MT6893 display dependencies` | 时钟/pinctrl/scpsys bringup 等 |
| 4. `drm/mediatek: MT6893 display support (DSC/DSI/DDP)` | **v2 栈本体**（MT6895-Mainline 移植 + 我们的 SoC 数据）|
| 5. `drm/panel: k10a 36 02 0a (DSC)` | 面板驱动 |
| 6. `arm64: configs: chopin_defconfig` | 最终配置 |
| 7. `port: tools/docs` | 构建、验证、诊断工具与文档（audit/ 只保留结论性文件）|

## 执行方式

1. ~~先冻结一个"v2 可显示"的 tag：`git tag chopin-v2-working`~~ **已做**（`21d1ba4`）；
2. ~~新建分支 `chopin-clean` ... 按上表逐个提交~~ **已做**，但基线取的是 DRM 工作的起点
   `99df977^`（`5baaae9`）而不是上游基线 —— 因为 clk/pinctrl/PMIC 那些在 `99df977` 之前
   就已经整理好了，这次要归类的正好是整个 DRM 工作；最终分成 10 组（比原表多，因为
   v2 栈 / MML / mainline DRM / SMI / cmdq / panel 各自独立了）；
3. 用 `git diff chopin-v2-working..chopin-clean --stat` 确认**最终树完全一致**
   （历史不同、内容相同）；
4. `git tag -d` 掉中间态 tag，删除临时分支，`chopin-clean` 取代旧历史；
5. port 仓库同样整理：保留 `DELIVERY.md`/`NEXT-DRM.md`/`DSC-REFERENCE.md`/`V2-IMPORT.md`/`SUMMARY.md`
   与 `tools/`，过程日志（`audit/*.log` 等）改成一个"结论文档 + 原始日志归档"的单提交。

## 注意

- 现在**不要**动历史：v2 还在调试，需要每次编译/对比都能回到任意一步；
- 达到"v2 真正驱动显示"后再执行上面步骤，且**先打 tag**，确保可回退。
