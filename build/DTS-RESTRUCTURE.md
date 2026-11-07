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

## 当前状态（2026-11-07）

- 计划完成，尚未开始迁移
- 工作基线：`out-scp-v187-rootfs`（可用）
- 相关记录：`media/VDEC-PORT.md`（VDEC 依赖 SCP，与 DTS 无关）
