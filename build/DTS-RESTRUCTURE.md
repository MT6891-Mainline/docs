# DTS 重整计划（参考 MT6895 主线移植）

目标结构（对照 `port/mt6895-linux`）：

```
mt6893.dtsi         ~4076 行    SoC：CPU/GIC/timer/时钟/电源/外设控制器
mt6893-xiaomi-chopin.dts  ~1449 行  板级：引脚、器件、面板、传感器、WiFi、USB、启动参数
```

现状：`mt6893.dtsi` 1270 行（已是主线风格），但板级外设散在 **25 个**
`mt6893-xiaomi-chopin-*.dtsi` 里，共 24,268 行。

## 文件分类

| 文件 | 行数 | 性质 | 目标 |
| --- | --- | --- | --- |
| `-storage-timers.dtsi` | 1779 | SoC（mmc/ufs/phy） | → `mt6893.dtsi` |
| `-i2c-spi.dtsi` | 1203 | **混合**（控制器 SoC + 器件板级） | 拆分 |
| `-board-devices.dtsi` | 1013 | 板级（传感器/触摸/稳压器） | → 板级 dts |
| `-iommu-gce.dtsi` | 992 | SoC（iommu/larb/gce） | → `mt6893.dtsi` |
| `-pmic-misc.dtsi` | 904 | **混合**（PMIC SoC + 板级稳压器） | 拆分 |
| `-display.dtsi` | 738 | SoC（mmsys/ovl/dsi） | → `mt6893.dtsi` |
| `-gpu.dtsi` | 643 | SoC（mali/opp） | → `mt6893.dtsi` |
| `-scp-dbg.dtsi` | 635 | **混合**（SCP SoC + 共享内存板级） | 拆分 |
| `-camera.dtsi` | 630 | SoC（seninf/isp） | → `mt6893.dtsi` |
| `-m4u-nodes.dtsi` | 569 | SoC（iommu 端口） | → `mt6893.dtsi` |
| `-infra-pinctrl.dtsi` | 510 | **混合**（pinctrl 控制器 SoC + 引脚板级） | 拆分 |
| `-lpm-memory.dtsi` | 452 | SoC（低功耗内存） | → `mt6893.dtsi` |
| `-multimedia.dtsi` | 351 | SoC（venc/vdec/jpeg） | → `mt6893.dtsi` |
| `-boot.dtsi` | 284 | 板级（启动参数/别名） | → 板级 dts |
| `-audio.dtsi` | 231 | SoC（audsys/AFE） | → `mt6893.dtsi` |
| `-ovl-*.dtsi`（14 个） | ~1400 | 板级叠加（面板/WiFi/USB/显示） | → 板级 dts |
| `-androidbase.dts` | ~10000 | 厂商原始板级（参考用，不参与编译） | 归档 |

## 迁移原则

1. **一个文件一次**，每步都能编译、能启动（回滚点）
2. 纯 SoC 文件：整体移入 `mt6893.dtsi` 末尾（节点定义不变，phandle 会重排，
   功能等价）
3. 混合文件：先拆出板级部分（引脚/器件/地址），控制器留 dtsi
4. 每步之后跑：
   `make dtbs` → 对比 DTB 大小/节点数 → 刷 boot 验证启动和关键外设
5. 全部完成后，板级 dts 应接近 mt6895-xiaomi-xaga.dts 的形态（单文件、
   按外设分节、无 ovl 叠加）

## 顺序（先易后难）

1. `-multimedia.dtsi`（351 行，纯 SoC）
2. `-audio.dtsi`（231 行，纯 SoC）
3. `-lpm-memory.dtsi`（452 行，纯 SoC）
4. `-m4u-nodes.dtsi` + `-iommu-gce.dtsi`（SoC）
5. `-storage-timers.dtsi`（SoC）
6. `-camera.dtsi`、`-gpu.dtsi`、`-display.dtsi`（SoC）
7. `-scp-dbg.dtsi`、`-pmic-misc.dtsi`、`-infra-pinctrl.dtsi`（拆分）
8. `-i2c-spi.dtsi`（拆分）
9. `-ovl-*`、`-board-devices.dtsi`、`-boot.dtsi`（合并进板级 dts）


## 实际情况（2026-11-07 盘点后修正）

把 30 个切片按 include 顺序合并后是 **12,096 行、约 700 个根节点**，而且：

- 切片**不是自包含的**：例如 `-camera.dtsi` 以未闭合的 `mfb@15812000 {` 结尾，
  内容延续到 `-multimedia.dtsi`。它们是反编译厂商 DTS 的任意切片。
- 节点用**数字 phandle**（`phandle = <0xad>`、`clocks = <0xad 0x02>`），
  不是主线风格的 label。
- 大量厂商 binding（`mediatek,mfb`、`mediatek,venc_gcon`、`mediatek,smi_larb7`…），
  主线没有对应驱动。

**结论：机械搬移不可行，必须按外设逐个重写**（参考 mt6895 的主线节点，
但用 MT6893 的地址/中断/时钟）。这是一个**多周**工程，不是一次整理。

## 可行的分阶段路线

优先级按「主线已有驱动 + 收益大」排序：

| 阶段 | 外设 | 理由 |
| --- | --- | --- |
| 1 | **mmc / ufs（存储）** | 主线 mtk-sd/ufs 驱动成熟，节点数少、地址独立 |
| 2 | **i2c / spi 控制器** | 主线 i2c-mt65xx/spi-mt65xx，板级器件留在 dts |
| 3 | **pinctrl** | 主线 pinctrl-mt6893，pin 定义搬板级 |
| 4 | **PMIC (mt6359) + regulators** | 主线 mt6397/mt6359 驱动 |
| 5 | **display（mmsys/dsi）** | 主线 mtk-drm，节点复杂 |
| 6 | **GPU（mali/panfrost）** | 主线 panfrost，已有工作 |
| 7 | **audio / AFE** | 主线 mt8192-afe 已验证可用 |
| 8 | **camera / vcodec / mfb 等** | 厂商 IP，需驱动移植 |

每阶段：重写节点（label 化）→ 编译 → 刷 boot 验证 → 提交。

## 立即可做的第一小步

把合并后的完整 DTS 保存为参考基线（`work/chopin-full.dtsi`，12,096 行），
下次从 **mmc 节点**开始重写。

## 当前状态（2026-11-07）

- 计划完成；已确认切片问题，路线改为「按外设重写」
- 合并基线：`work/chopin-full.dtsi`（12,096 行，717 节点）
- 工作基线：`out-scp-v187-rootfs`（可用）
- 相关记录：`media/VDEC-PORT.md`（VDEC 依赖 SCP，与 DTS 无关）
