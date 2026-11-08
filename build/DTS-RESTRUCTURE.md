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


## 数据驱动的清理路径（2026-11-07 实测）

用当前能启动的 DTB 统计：

```
DTB 里 mediatek compatible：618 个
  ├─ 有驱动：106 个   ← 真正的 DTS 内容
  └─ 无驱动：512 个   ← 厂商死节点（对应驱动根本没编）
```

样本（无驱动）：`mediatek,mfb`、`mediatek,venc_gcon`、`mediatek,smi_larb7`、
`mediatek,ap_ccif0..5`、`mediatek,adsp_*`、`mediatek,aee_debug_kinfo`、
`mediatek,apcldmain*`、`mediatek,ap_md_*` …

**结论：DTS 重整的正确顺序是先删死节点，再重写活节点。**

1. **删死节点**（512 个）：没有驱动绑定它们，删除不改变行为；唯一要小心的是
   它们被活节点用 phandle 引用的情况（如 smi/larb、clock controller），需要
   先做引用图分析，保留被引用的。
2. **重写活节点**（106 个）：这才是 mt6895 风格的目标 DTS（SoC 部分 + 板级
   部分），按外设分节、用 label。

删完之后 DTS 预计从 12,096 行降到 2,000–3,000 行，和 mt6895 的规模相当。

### 工具产物

- `work/compatibles.txt`：DTB 里全部 686 个 compatible
- `work/driver-compat.txt`：树里 1,503 个 mediatek 驱动 compatible
- `work/dead-compat.txt`：512 个无驱动 compatible
- `work/board.dts`：当前 DTB 的反编译（节点/引用图分析用）
- `work/chopin-full.dtsi`：30 个切片合并后的 12,096 行全文

### 下一步（删死节点）

1. 从 `work/board.dts` 提取死节点的完整路径与 phandle
2. 构建引用图：哪些死节点的 phandle 被活节点引用
3. 未被引用的死节点 → 从对应切片文件里删除（按节点边界）
4. 被引用的 → 保留节点但可简化
5. 编译 + 刷 boot 验证


## 剪枝试验结果（2026-11-07）

用「死节点 + 引用图」自动剪枝（脚本 `work/prune-dts.py`）：

```
原始（合并后 DTB 反编译）  12,096 行
剪枝后                      8,740 行   （删除 313 个死节点）
DTB                        276K → 222K
dtc 编译                    通过（只剩 reg/unit_address 警告）
```

产物：

- `work/chopin-pruned.dts` —— 剪枝后的完整 DTS（独立可编译）
- `work/chopin-pruned.dtb` —— 编译验证
- `work/prune-dts.py` —— 剪枝脚本（保留被活节点引用的死节点及其祖先）
- `work/board.dts` —— 原始 DTB 反编译（12,096 行的来源）

**注意**：这是**反编译产物**（数字 phandle、无 label），还不是目标形态。
它的价值是验证了「死节点可以安全删除」这一点，并给出了可用的中间基线。

### 下一步

1. 用 `__symbols__` 把数字 phandle 转成 label（脚本已有基础：`work/clean-dts.py`）
2. 按 SoC / 板级分类节点（参考 mt6895 的节点归属）
3. 生成 `mt6893.dtsi`（SoC）+ `mt6893-xiaomi-chopin.dts`（板级）两个文件
4. 编译 → 刷 boot 验证


## 运行时设备树印证（2026-11-07）

从设备（10.223.197.115）直接读取**运行中的 DTB**：

```sh
cp /sys/firmware/fdt /tmp/live.dtb      # 282,313 字节
dtc -I dtb -O dts /tmp/live.dtb -o live.dts
```

### 对比结果

运行中 DTB 与构建产物 **只差 4 行**：

| 差异 | 说明 |
| --- | --- |
| `linux,initrd-end` | bootloader 补的 initrd 地址（正常） |
| `core_0 = "disable"` vs `"enable"` | 构建目录曾是 v189 残留；已重新 `make dtbs` 同步 |

### status 不能作为"能用"的依据

live DTB 里带 status 的节点 118 个：83 个 okay/ok、34 个 disabled。
但大量厂商节点（`msdc@11230000`、`adsp_*`、`mcu*`）status 是 okay，
驱动却根本没编进内核——**status 只反映 DTS 的意愿，不反映绑定**。

### 真正的依据：已绑定驱动的设备

```
/sys/bus/platform/drivers/*/  下能找到 driver 链接的设备 = 92 个
```

示例：`clk-mt6893-*`、`pinctrl-mt6893`、`mtk-sd`(mmc0)、`mtk-ufs`、`mtk-mbox`、
`mtk-pmic-wrap`、`pwm-mtk-disp`、`mediatek-drm`、`panfrost`、`mtk-iommu` 等。

**分类优先级**：驱动绑定（92）> 树里有驱动（106）> status okay（83，不可靠）。

`work/live-bound.txt` 保存了完整列表（驱动名 + 设备名）。


## SoC/板级拆分结果（2026-11-07）

用剪枝后的 DTS 按「有地址在 SoC 区间 / compatible 是 mediatek,mt6893-* 系列」
拆分：

| 文件 | 节点 | 行数 |
| --- | --- | --- |
| `work/mt6893-soc-test.dtsi` | 605 | 4,150 |
| `work/mt6893-board-test.dts` | 360 | 4,060 |
| 合计 | **965** | 8,210（与剪枝后的 965 节点一致） |

规模与 mt6895 参考相当（mt6895.dtsi 4,076 行 + 板级 1,449 行）。

编译验证：板级文件在根节点内 `#include` SoC 文件，`dtc` 能产出 DTB
（`work/mt6893-board-test.dtb`，153K）。

**已核实原因**：编译后 DTB 只有 360 个节点（剪枝版 965）。拆分文件里有
**20 组同名节点**：

```
pins_cmd_dat  × 110     pins_cmd1_dat × 23     pins_cmd2_dat × 8
pins_cmd × 7            pins_dat × 7           pins_clk × 7
port × 4                endpoint × 4           ...
```

这些原本在不同父节点下（不同 pinctrl 组），拆分**重排**后落到同一父节点，
dtc 把同名兄弟节点**合并**，于是丢了 600 个节点。

**结论：必须先做 label 转换，再做拆分。**

原因：厂商 DTS 把 pinctrl 组直接写在 `pio` 节点内部；主线风格是
`&pio { ... }` 片段写在板级文件里。只有先转成 label 引用，板级才能用
`&pio { }` 覆盖方式携带自己的引脚组，拆分才不会改变节点树。

所以顺序修正为：

1. **label 转换**（`__symbols__` 492 个 label，脚本 `work/clean-dts.py` 已有基础）
2. 把板级内容改写成 `&label { ... }` 片段
3. 拆分（保持节点树的父子关系不变）
4. 编译 + 刷 boot 验证

### 下一步

1. 核实同名节点问题
2. label 转换（`__symbols__` 492 个）
3. 生成正式的 `mt6893.dtsi` + `mt6893-xiaomi-chopin.dts`（替换 30 个切片）
4. 编译进内核 + 刷 boot 验证（存储/WiFi/显示/触摸/传感器逐项确认）


## label 自动转换的结论（2026-11-07）

在反编译产物上自动做 label 转换，验证结果是**不可靠**：

- 485 / 673 个 phandle 能映射到 `__symbols__` 里的 label（其余是没名字的）
- 但转换后 DTS 出现语法错误：属性值被错误替换（`interrupt-parent = <interrupt-parent>`）、
  同名节点（`pins_cmd_dat` ×110）无法区分、label 插入位置错乱

**原因**：反编译的 DTS 丢失了类型信息（哪些 `<0x..>` 是 phandle、哪些是普通数字），
也丢失了原始的 label 位置。dtc 反编译出来的文本**不是可逆的**。

**结论：放弃在反编译产物上自动转换 label。** 正确路线是：

1. 以 `work/chopin-pruned.dts`（8,740 行，编译通过）作为**参考**
2. **按外设手工重写**：一个外设一个外设地写主线风格节点（label、`&label` 引用、
   主线 binding），每写完一个就编译验证
3. 已经手工迁移好的部分不要动：`mt6893.dtsi` 里的 mmc0/mmc1、ufs、clk、pinctrl、
   iommu、scpsys 等
4. 板级文件逐节迁移：先引脚和稳压器，再 i2c/spi 器件，最后面板/传感器/摄像头

这是一条**纯手工、可验证**的路线，虽然慢但没有隐藏风险。

### 当前可用基线

- `work/chopin-pruned.dts` — 剪枝后的 DTS（8,740 行，965 节点，编译通过）
- `work/board.dts` — 原始反编译（12,096 行，参考用）
- `work/live-bound.txt` — 92 个真实绑定设备（判断节点是否要保留的依据）
- `build/DTS-RESTRUCTURE.md` — 本文档（全部结论）


## i2c/spi 迁移的实测结论（2026-11-07）

i2c 控制器在 `mt6893.dtsi` 里已经是主线 binding（`mediatek,mt6893-i2c` +
`mt8192-i2c`，label 齐全）。但板级 **器件** 的盘点结果：

```
i2c0  3 个器件：bq2597x / bq27z561 / xmusb350-charger     全厂商
i2c2  4 个器件：camera_*                                   全厂商
i2c4  4 个器件：camera_*                                   全厂商
i2c5  4 个器件：mt6360(✓主线) / usb_type_c / subpmic_*     1 主线
i2c6  8 个器件：ext_buck_* / speaker_amp / cs35l41 / ...   全厂商
i2c7  2 个器件：nfc / willsemi ldo                         全厂商
i2c8  7 个器件：camera_*                                   全厂商
i2c9  6 个器件：et7480 / lcd_* / bq2597x / aw8697 / ...    全厂商
```

**只有 mt6360（PMIC）是主线驱动**（`mediatek,mt6360` MFD + regulator/charger/
adc/led/tcpc 子驱动都在主线）。

其余 ~40 个器件的主线驱动**不存在**：摄像头 EEPROM、副 PMIC（bq2597x）、
充电芯片（xmusb350）、音频开关（et7480）、扬声器功放（cs35l41）、NFC、
LDO（wl2866d）……这些不是 DTS 重写能解决的，需要**驱动移植**。

### 结论

DTS 的完整「mt6895 式重写」被**厂商驱动缺失**卡住，不是卡在 DTS 本身：

| 部分 | 能否重写 |
| --- | --- |
| i2c/spi/pinctrl/时钟/IOMMU 控制器 | ✅ 已主线化 |
| mt6360 PMIC | ✅ 可重写（主线全支持） |
| mmc/ufs | ✅ 已主线化 |
| 音频 AFE | ✅ 可重写（mt8192-afe） |
| 显示 mmsys/dsi | ✅ 可重写（mtk-drm） |
| GPU | ✅ panfrost |
| 摄像头 / 副 PMIC / 充电 / NFC / 音频开关 / LDO | ❌ 需驱动移植 |

### 现实路线

1. 保留 `work/chopin-pruned.dts`（8,740 行）作为**剪枝基线**
2. 只对**主线驱动已就绪**的外设做 label 化重写（mt6360、音频、显示）
3. 厂商外设保持现状（厂商 binding 的节点跟着板级文件走）
4. 完整的 mt6895 式干净 DTS，要等对应驱动移植完成

这个结论也解释了为什么之前的 label 自动转换会失败：**大部分节点注定要保留
厂商风格**，强行统一 label 没有意义。


## mt6360 主线化重写（2026-11-07）

mt6360 是 i2c 器件里**唯一有完整主线驱动**的（MFD + regulator/charger/adc/led/tcpc）。

### 新节点已写好并编译通过

`work/chopin-mt6360-test.dtsi`：

```dts
&i2c5 {
	mt6360: mt6360@34 {
		compatible = "mediatek,mt6360";
		reg = <0x34>;
		wakeup-source;
		interrupts-extended = <&pio 24 IRQ_TYPE_LEVEL_LOW>;
		interrupt-names = "IRQB";
		interrupt-controller;
		#interrupt-cells = <1>;

		regulators {
			BUCK1: buck1 { regulator-name = "VMDLA"; ... always-on; };
			BUCK2: buck2 { regulator-name = "VDRAM1"; ... always-on; };
			LDO1: ldo1  { regulator-name = "VFP";  ... always-on; };
			LDO2: ldo2  { regulator-name = "vtp";  ... always-on; };
			LDO3: ldo3  { regulator-name = "VMC";  ... };
			LDO5: ldo5  { regulator-name = "VMCH"; ... };
			LDO6: ldo6  { regulator-name = "VMDDR"; ... };
			LDO7: ldo7  { regulator-name = "VDRAM2"; ... always-on; };
		};
	};
};
```

用 `cpp + dtc` 对着 `mt6893.dtsi` 编译：**通过**（DTB 23K，只有 dtsi 自带的
mailbox 单元地址警告）。

### 参数来源（全部来自厂商 DTS）

| 稳压器 | 名称 | 范围（V） | always-on |
| --- | --- | --- | --- |
| BUCK1 | VMDLA | 0.3–1.3 | ✓ |
| BUCK2 | VDRAM1 | 0.3–1.3 | ✓ |
| LDO1 | VFP | 1.2–3.6 | ✓ |
| LDO2 | vtp | 1.2–3.6 | ✓ |
| LDO3 | VMC | 1.2–3.6 | |
| LDO5 | VMCH | 2.7–3.6 | |
| LDO6 | VMDDR | 0.5–2.1 | |
| LDO7 | VDRAM2 | 0.5–2.1 | ✓ |

### 消费者引用（只需改 3 处）

```
vfp-supply     = <&LDO1>
vtouch-supply  = <&LDO2>
vmmc-supply    = <&LDO5>
```

（厂商 phandle：LDO1=0x1e5、LDO2=0xf1、LDO5=0x51）

### 集成步骤（下次）

1. 从板级切片里删掉 `mt6360_pmu_dts` / `mt6360_pmic_dts` / `mt6360_ldo_dts`
2. 引入 `chopin-mt6360-test.dtsi`（改名去掉 test）
3. 三处消费者改为 `&LDO1/2/5`
4. 编译 → 刷 boot → 确认：
   - `regulator_summary` 里 VMDLA/VDRAM1/VFP/vtp/VMC/VMCH/VMDDR/VDRAM2
   - 触摸（vtouch）、eMMC（vmmc）、显示屏（VFP）正常
   - 充电、ADC 通道仍可用

风险：charger/led/adc 子节点还没写（主线驱动需要 `mediatek,mt6360-chg` 等），
充电可能暂时失效——需要一并补上，或者分两步（先稳压器，验证电源，再补充电）。


## mt6360 后续：已经主线化，剩清理（2026-11-07）

设备实测发现 **主线 mt6360 栈早就在工作**：

```
/sys/bus/i2c/devices/5-0034 -> mt6360            ← 主线 MFD 已绑定
/sys/bus/i2c/devices/0-0055 -> bq27xxx-battery   ← 电池（主线驱动）
/sys/bus/i2c/devices/6-0040 -> cs35l41           ← 左声道功放（主线）
/sys/bus/i2c/devices/6-0042 -> cs35l41           ← 右声道功放（主线）
/sys/bus/i2c/devices/7-0028 -> wl2866d-regulator ← 摄像头 LDO（主线）
```

板级 DTS 里 `subpmic@34` 就是 `mediatek,mt6360`，charger/adc/tcpc 子节点
已经写好（端口迁移时完成）。所以新写的 `chopin-mt6360-test.dtsi` 与它重复，
真正的任务是**删除厂商的三个死节点**。

### 但删除不是无痛的

厂商三个节点的 phandle 被大量引用：

| 节点 | phandle | 被引用次数 | 引用者 |
| --- | --- | --- | --- |
| `mt6360_pmu_dts` | 0x03 | **40** | 38 个节点的 `interrupt-parent` |
| `mt6360_pmic_dts` | 0xef | 2 | `vib_cont_td` 等 |
| `mt6360_ldo_dts` | 0xf0 | 24 | `regulator-enable-ramp-delay` 等 |

关键冲突：厂商 PMU 是 **2-cell** 中断控制器（`#interrupt-cells = <2>`，
IRQ + 类型），主线 MFD 是 **1-cell**（`#interrupt-cells = <1>`）。38 个
`interrupt-parent = <&mt6360_pmu>` 的节点要改成主线 MFD 的 IRQ 编号
（`interrupts = <MT6360_IRQ_xxx>`），这是**逐节点的人工转换**。

### 结论

mt6360 是「**驱动已主线化、DTS 混用**」的典型：

- 主线栈已经在跑 ✓（充电、电池、功放、摄像头 LDO 都正常）
- 厂商节点是死代码 ✗，但因为**中断控制器语义不同**，删除需要逐节点转换
- 这不是 DTS 重写的障碍，而是**清理工作量**（38 处中断引用）

如果要彻底清理，可以先做一轮「中断引用转换」：把 38 个节点的
`interrupt-parent`/`interrupts` 从厂商 2-cell 改成主线 1-cell。这需要对照
`drivers/mfd/mt6360-core.h` 里的 IRQ 编号（`MT6360_xxx` 枚举）。


## 中断引用转换：实际只需 1 处（2026-11-07 修正）

之前统计的「40 处引用」是把 phandle 值 `0x03` 出现次数都算上了。实际用
`interrupt-parent = <0x03>` 的节点只有 **4 个**：

| 节点 | interrupts | 处理 |
| --- | --- | --- |
| `adc`（厂商） | `<0x29 0x00 0x2b 0x00 0x2c 0x00>` | 随厂商节点删除 |
| `mt6360_pmic_dts`（厂商） | 12 组 | 随厂商节点删除 |
| `mt6360_ldo_dts`（厂商） | 8 组 | 随厂商节点删除 |
| **`type_c_port0`**（板级 Type-C） | `<0x40 0x00>` | **需要转换** |

所以真正的转换只有 **1 个节点**：

```dts
/* 现在（厂商 2-cell） */
type_c_port0 {
    interrupt-parent = <&mt6360_pmu>;      /* 厂商 PMU */
    interrupts = <0x40 0x00>;              /* IRQ 64 + 类型 */
};

/* 目标（主线 1-cell） */
type_c_port0 {
    interrupt-parent = <&mt6360>;          /* 主线 MFD */
    interrupts = <MT6360_IRQ_xxx>;         /* 主线 regmap_irq 编号 */
};
```

需要确认线上 mt6360 的 regmap_irq 编号（`drivers/mfd/mt6360-core.c` 的
`regmap_irq` 表 / `mt6360-irq.h`），然后一对一替换。

### 清理顺序

1. 转换 `type_c_port0` 的中断引用
2. 删除 `mt6360_pmu_dts` / `mt6360_pmic_dts` / `mt6360_ldo_dts`（约 100 行）
3. 删除板级里的厂商 `subpmic@34`（vendor 绑定，约 60 行）
4. 编译 → 刷 boot → 验证：充电、电池、Type-C、触摸、显示


## v191 验证 + 传感器回归修复（2026-11-08）

v191（单文件 DTS）刷机结果：

| 功能 | 结果 |
| --- | --- |
| 开机 | ✅ |
| 显示（主线 mtk-drm） | ✅ |
| 触控 | ✅ |
| GPU / UFS | ✅ |
| **WiFi** | ✅ 恢复 |
| **蓝牙** | ✅ 恢复 |
| **自动旋转 / 加速度计** | ❌ 仍然不工作 |

### 原因：v189 试验遗留的配置

v189（试 rv SCP 驱动）时执行过：

```sh
scripts/config -d MTK_SENSORHUB
```

之后一直没恢复，导致 v190/v191 的内核**不含 sensorhub 驱动**：

```
# CONFIG_MTK_SENSORHUB is not set
```

于是 `sensorhub_iio.c` 不会注册 `chopin-als` / `chopin-accel` 两个 IIO 设备
（`/sys/bus/iio/devices/` 里只剩 MT6359 和 mt6360-adc），加速度计数据为空，
自动旋转自然失效。

**与 DTS 无关** —— DTB 是 282,313 字节的等价版本，传感器节点都在。

### 修复

```sh
scripts/config -e MTK_SENSORHUB
make olddefconfig            # CONFIG_MTK_SENSORHUB=y
port/tools/build-kernel.sh
```

v192：`out-scp-v192-sensor/boot-stockdt.img`（md5 `4f66941e8e0ea1205db89d001705638c`）。

### 教训

内核配置的临时改动（`scripts/config -d/-e`）会**留在 `.config` 里**，跨构建
持续生效。试错性质的配置必须在同一次会话里恢复，或者记录到文档里。这次
`MTK_SENSORHUB` 就是从 v189 一直漏到 v191。


## 设备唯一值清理（2026-11-08，开源前必须做）

单文件 DTS 来自**运行中的 DTB**，因此带上了这台机器的实例标识。检查发现三处：

| 位置 | 内容 | 性质 |
| --- | --- | --- |
| `/firmware/android/serialno` | `ib6dsk6pxofarglb` | **设备序列号** |
| `bootargs` 里 `androidboot.serialno=` | 同上 | 同上 |
| `bootargs` 里 `androidboot.chipid=` / `cpuid=` | `0xa4f8e0…` | **SoC 唯一 ID** |
| `bootargs` 里 `androidboot.vbmeta.device=PARTUUID=` | `1f9b0939-…` | 分区表 UUID |
| `/firmware/android/vbmeta.digest` | `0e522dae…` | 本机 vbmeta 摘要 |

### 影响

- **不会导致其他设备无法运行**：这些都是 Android 侧标识，Linux 启动不依赖它们。
  内核实际 cmdline 来自 boot.img（`chopinport=…`），**DTS 里的 `bootargs`
  根本没被使用**（已在设备上用 `/proc/cmdline` 验证）。
- **但会泄漏并串号**：`/sys/firmware/devicetree/base/firmware/android/serialno`
  对用户态可读。其他 chopin 设备刷这个内核后，会读到自己机器上**不属于它的
  序列号**——既是隐私泄漏，也让依赖序列号的工具行为错乱。

### 已清理（v193）

删除：

- `serialno` 属性（替换为注释）
- `vbmeta.digest` / `vbmeta.size`
- bootargs 里的 `androidboot.serialno` / `chipid` / `cpuid` / `vbmeta.device=PARTUUID`

保留（板级/型号级，所有 chopin 相同）：

- `panel_name=dsi_k10a_36_02_0a_dsc_vdo`
- `androidboot.hwversion=1.9.0`
- `androidboot.product.hardware.sku=chopin`
- `WpAndMaxlum:0x7c,0x7d,0x56`（面板校准，同型号共用）
- `ramoops.mem_address=…`（pstore 保留区）
- 全部 mblock reserved-memory（LK 注入的 modem/GSP/display 区域，**不能删**）

清理后 DTB：282,313 → **281,906 字节**（少 407 字节）。
启动镜像：`out-scp-v193-clean/boot-stockdt.img`（md5 `6dc6fdbed1f516892b8e15914e5e04fe`）。

### 开源前检查清单

```sh
grep -nE "serialno|chipid|cpuid|vbmeta\.digest|PARTUUID=" \
     arch/arm64/boot/dts/mediatek/mt6893-xiaomi-chopin.dts
# 期望：只匹配到注释，没有真实值
```

同类风险还有（本机未出现，但他机可能需要的）：WiFi/BT 的 MAC、`/chosen/random-seed`、
传感器校准数据。若未来从别处导入 DTS，同样要检查。


## 完成：单文件 DTS，切片全部移除（2026-11-08）

验证过没有任何 `#include` 引用后，删除了全部死文件：

```
删除 33 个文件：
  mt6893-xiaomi-chopin-*.dtsi        30 个切片
  mt6893-xiaomi-chopin-androidbase.dts  （厂商完整 DTS 参考副本）
  mt6893.dtsi                            （只被死切片引用）
  mt6893-pinfunc.h                       （DTS 不用 PIN() 宏）

arch/arm64/boot/dts/mediatek/ 现在只剩：
  mt6893-xiaomi-chopin.dts    10,770 行
```

提交：

- `3d77d2177` arm64: dts: mediatek: keep chopin as one self-contained board tree
  （33 files changed, 10,750 insertions, 24,248 deletions）
- `e17cf6e64` arm64: dts: mediatek: drop the unused mt6893 pinfunc header

构建验证：

```
make dtbs               0 errors
DTB                     281,906 字节
kernel embedded FDT     281,906 字节
```

`chopin-dtb.S` 嵌入的是构建后的 `arch/arm64/kernel/chopin-board.dtb`，
与这些源文件无直接引用关系，删除切片不影响它。

### 为什么之前不能删

切片之间是**任意位置切断**的（一个文件以未闭合的 `mfb@15812000 {` 结尾，
下一个文件接着它的属性），`mt6893.dtsi` 又是被这些切片 include 的。只要还有
一个切片在编译，删任何别的文件都会断链。合并成单文件后整条依赖链才消失。


## 拆分完成：mt6893.dtsi + 板级 dts（2026-11-08）

按 MT6895 的组织方式（`mt6895.dtsi` 4,076 行 + `mt6895-xiaomi-xaga.dts`
1,449 行）拆分：

| 文件 | 行数 | 内容 |
| --- | --- | --- |
| `mt6893.dtsi` | **7,307** | SoC：CPUs/GIC/timer/clocks/power/IOMMU/SMI/MMC/UFS/I2C/SPI/PMIC wrapper/display/GPU/SCP |
| `mt6893-xiaomi-chopin.dts` | **2,851** | 板级：model/memory/reserved-memory(mblocks)/regulators/pin states/panel/touch/sensors/WiFi/USB |

### 分类方法

按 `reg` 地址判断：`0x1000_0000 - 0x1FFF_FFFF` 属于 SoC 外设空间。

- 注意 `#address-cells = <2>`，地址在**第二个** cell，第一次按第一个 cell 判
  会把所有外设判成板级（SoC 只剩 31 个）
- 结果：SoC **478** 个直接子节点，板级 **147** 个

### 构建验证

```
mt6893.dtsi               7,307 行
mt6893-xiaomi-chopin.dts  2,851 行
DTB                       281,830 字节（拆分前 281,906，差 76 字节）
embedded FDT              281,830 字节
```

DTB 缩小 76 字节、反编译 diff 3,715 行，但**差异来自节点顺序重排**（SoC 节点
先输出、板级后输出），内容等价。所有 phandle 都是显式数字，重排不影响引用。

启动镜像：`out-scp-v194-split/boot-stockdt.img`（md5 `081139c711f4d015b2b1562196d9b66b`）。

### 后续可以做的

- 板级文件里的 `&label` 覆盖还没做（现在是完整节点定义），进一步可以改成
  `&pio { ... }` / `&i2c0 { ... }` 形式，像 mt6895 那样
- 数字 phandle 可以逐步换成 label
- SoC dtsi 里的厂商 binding 节点（`mediatek,mfb`、`mediatek,venc_gcon` 等）
  要等对应驱动移植后再改


## label 化完成（2026-11-08）

两步做完，**DTB 全程保持不变**（281,602 字节）：

### 1. `__symbols__` → 真 label

板级 dts 里原本有一个 `__symbols__` 节点（492 条 `label = "/path"`，是反编译
时从 DTB 带回来的）。把它删除，按路径把标签写到对应节点上：

```
485 个节点拿到 label（491 个路径里 6 个无标签）
```

`-@` 编译时 dtc 会自动重新生成 `__symbols__`，内容一致。

### 2. 数字 phandle → `&label`

按属性白名单 + provider 的 `#*-cells` 精确消费 cells：

| 属性 | cells 来源 |
| --- | --- |
| `clocks` | provider 的 `#clock-cells` |
| `power-domains` | `#power-domain-cells` |
| `iommus` | `#iommu-cells` |
| `resets` | `#reset-cells` |
| `mboxes` / `dmas` / `pwms` | 各自的 `#*-cells` |
| `interrupt-parent`、`*-supply`、`phys`、`memory-region`、`remote-endpoint`、`extcon`、`usb-role-switch`、`thermal-sensors`、`sound-dai` | 0（单 phandle） |
| `*-gpios` | `#gpio-cells` |

结果：

```
转换属性   1,013 处
&label 引用 343 处
残留数字     3 处（3 个 provider 自身没有 label）
DTB         281,602 字节（与转换前一致）
```

### 遇到的坑

1. `#address-cells = <0x01>` 被误转换——正则没排除 `#` 开头的属性，加
   `(?<![#\w-])` 负向后视解决
2. 属性值没有匹配到 label 时直接返回了裸值（`phandle = 0x0b;`），丢掉尖括号
   → 必须返回 `<原始值>`

### 提交

```
852d97c41  arm64: dts: mediatek: use labels instead of numeric phandles on chopin
```

启动镜像：`out-scp-v195-labels/boot-stockdt.img`（md5 `78da811250fd2dd95ac77e31f4d86a75`）。

### 剩下的一步

板级还在用**完整节点定义**携带引脚状态和 i2c 器件，下一步可以改成主线风格的
覆盖片段：

```dts
&pio {
	pins_...: ... { ... };
};

&i2c0 {
	touchscreen@5d { ... };
};
```

这样板级文件只保留「覆盖」，SoC 定义完全在 `mt6893.dtsi` 里，和 mt6895 一致。


## 板级改覆盖片段（2026-11-08，对齐 mt6895 风格）

把板级从「完整节点定义」改成主线式的 `&label {}` 覆盖，**DTB 全程 281,602
字节不变**（每组都编译验证）：

| 覆盖块 | 移出内容 | 行数 |
| --- | --- | --- |
| `&pio` | 43 个 `pinctrl_state_*` 组 | 384 |
| `&i2c0..13`（8 条总线） | 39 个器件节点 | — |
| `&spi0/1/5` | 3 个器件节点 | — |
| `&dsi0` | panel + ports | — |

### 遇到的问题

1. **`gpio@10005000` 不是引脚控制器**：它只有 7 行（`status = "disabled"`、
   `mediatek,gpio`），真正的控制器是 `pio: pinctrl {`（无 `reg`，靠 `reg` 列表
   里的 9 段地址工作）。第一次找错节点，DTB 少了 6KB
2. **引脚状态是二级子节点**：`pinctrl_state_*` 组本身还有 `pins_*` 子节点，
   直接按「二级子节点」搬移会连组一起丢掉

正确的做法是**按整个组**（`pinctrl_state_*` 子树）搬移，保留组内层级和
phandle。

### 结果

```
mt6893.dtsi                6,235 行  （原 7,307，减少 1,072）
mt6893-xiaomi-chopin.dts   3,555 行  （原 2,357，增加 1,198）
板级 &label 覆盖块          14 个
DTB                        281,602 字节（不变）
```

提交：

```
a42138cfc  move chopin pin states into a board &pio override
7e6fab395  move chopin i2c devices into board overrides
53588a7f8  move chopin spi devices and panel into board overrides
```

启动镜像：`out-scp-v196-overrides/boot-stockdt.img`
（md5 `b45d188a6910ab7b33ff41accb17e780`）。

### 还可以继续的

- `&mmc0` / `&ufs` 的板级参数、`&gpu`、`&mfg` 等
- 板级根节点里的厂商容器（`mt6360_pmu_dts`/`mt6360_pmic_dts`/`mt6360_ldo_dts`）
  可以合并到 mainline 的 `&i2c5 { mt6360@34 { regulators {...} } }` 里
- `mtk_lpm` / `mtk_lpm_sysram` 等厂商节点可以去掉（如果驱动不用）


## phandle 属性全部移除（2026-11-08）

所有引用都是 `&label` 之后，654 个 `phandle = <0xNN>` 属性变成冗余——dtc
会自己分配编号。删除前先清掉最后 3 处数字引用：

| 位置 | 原值 | 改为 |
| --- | --- | --- |
| `phys`（UFS PHY） | `<0xc5>` | `<&ufs_mphy>`（补了 label） |
| `ppi-interrupt-parent` ×2 | `<0x01>` | `<&gic>` |

然后删掉全部 654 个 `phandle` 属性：

```
mt6893.dtsi                7,307 → 5,803 行
mt6893-xiaomi-chopin.dts   3,555 → 3,333 行
DTB                        281,602 → 278,955 字节
```

**phandle 编号全部变了**（dtc 按引用顺序重新分配，486 个），但每个引用仍指向
同一个节点——这是 DTB 大小变化但功能等价的原因。

提交：`639c07b24`。启动镜像：
`out-scp-v197-nophandle/boot-stockdt.img`（md5 `0551030b4b0cec55a9947b2b11a87520`）。

### 验证方法

```sh
# 引用是否都解决（dtc 遇到未定义的 &label 会直接报错）
make dtbs

# __symbols__ 是否仍然完整
dtc -I dtb -O dts <dtb> | grep -c '= "/'
```

## 当前状态（2026-11-07）

- 计划完成；已确认切片问题，路线改为「按外设重写」
- 合并基线：`work/chopin-full.dtsi`（12,096 行，717 节点）
- 工作基线：`out-scp-v187-rootfs`（可用）
- 相关记录：`media/VDEC-PORT.md`（VDEC 依赖 SCP，与 DTS 无关）
