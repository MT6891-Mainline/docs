# 主线 DRM（mtk-drm）显示迁移评估

当前显示跑的是**厂商 `mediatek_v2` DRM**（`CONFIG_DRM_MEDIATEK_V2=y`），
不是主线。本文记录迁到主线 mtk-drm 的可行性。

## 好消息：主线驱动已完整支持 MT6893

`drivers/gpu/drm/mediatek/mtk_drm_drv.c` 里已经有：

```
mt6893_mtk_ddp_main[]   OVL0 → RDMA0 → COLOR0 → CCORR → AAL0 →
                        GAMMA → POSTMASK0 → DITHER0 → DSI0
mt6893_mmsys_driver_data
  compatible = "mediatek,mt6893-mmsys"
  compatible = "mediatek,mt6893-disp-mutex"
  compatible = "mediatek,mt6893-dsi"
```

时钟驱动也有：`clk-mt6893-mm`（设备上已绑定，`/sys/bus/platform/drivers` 里
能看到）。电源域 `MT6893_POWER_DOMAIN_DIS` 在 dtsi 里已声明。

**缺的只是 DTS 节点**（mmsys/ovl/rdma/color/ccorr/aal/gamma/postmask/
dither/mutex/dsi），以及面板驱动。

## 面板

厂商 DTS 里的面板节点：

```
dsi@1400e000 → panel@0 compatible = "k10a_36_02_0a_dsc_vdo,lcm"
dsi@1410e000 → panel@0 compatible = "k10a_42_02_0b_dsc_vdo,lcm"
```

`k10a` 是小米的面板代号，**不是标准 IC 名**。主线现有的候选驱动：

```
panel-ebbg-ft8719.c      (FT8719)
panel-himax-hx8394.c
panel-ilitek-ili9881c.c
panel-novatek-nt36523.c
panel-novatek-nt36536.c
```

需要从面板的初始化序列（DSC + VDO 模式 + 分辨率 1080x2400）反推实际 IC，
然后决定是复用现有驱动还是新写一个。**这是整个迁移里最不确定的部分。**

## 工作量评估

| 部分 | 状态 | 工作量 |
| --- | --- | --- |
| mtk-drm 驱动 | ✅ 已支持 mt6893 | 0 |
| 时钟（clk-mt6893-mm） | ✅ 已有并绑定 | 0 |
| 电源域 | ✅ dtsi 已声明 | 0 |
| DTS：mmsys + DDP 组件 + mutex + dsi | ❌ 缺 | 1–2 天（可参照 mt8192.dtsi） |
| 面板驱动（k10a） | ❌ 未知 IC | **3–10 天**（反推 + 写驱动 + 调初始化） |
| pinctrl（TE / bias / reset） | 部分有 | 1 天 |
| 调试验证 | — | 2–3 天 |

合计 **2–3 周**，且面板部分有风险（初始化序列错会导致黑屏/花屏）。

## 面板安全注意

`status/STATUS.md` 里记录过：**绝不要触发真正的 suspend**，DPMS off/on 是安全的。
显示调试时只做 DPMS 切换，不要用 `echo mem > /sys/power/state`。

## 建议顺序

1. 先只加 **DTS 节点**（mmsys/DDP/mutex/dsi），用厂商面板节点占位，
   `CONFIG_DRM_MEDIATEK_V2` 保持开启——确认节点能被主线驱动 probe（不接管）
2. 用 `drm_info` / `modetest` 确认 DDP 组件都注册成功
3. 反推面板 IC，写/改面板驱动
4. 切到主线 DRM（关 `DRM_MEDIATEK_V2`），逐项验证：开机、背光、DPMS、
   自动旋转（需要 DRM 的旋转属性）
5. 回滚路径：随时可以刷回 v187（厂商 DRM）


## DTS 迁移的实际障碍（2026-11-07）

对照厂商 DTS 与主线 mtk-drm 的期望，发现**地址模型不一致**：

### 厂商 DTS（反编译）

```
dispsys {  compatible = "mediatek,dispsys";  /* 没有 reg！只是时钟/复位提供者 */ };
disp_ovl@14000000 { reg = <0x00 0x14000000 0x00 0x1000>; };   ← OVL0 在 0x14000000
disp_ovl@14001000 { ... };  disp_ovl@14002000 { ... };
dsi@1400e000 { reg = <0x00 0x1400e000 0x00 0x1000>; };        ← DSI 在 0x1400e000
```

### 主线期望（参照 mt8192）

```
mmsys: syscon@14000000 { reg = <0 0x14000000 0 0x1000>; };     ← MMSYS 在 0x14000000
ovl0:  ovl@14005000   { ... };
dsi0:  dsi@14010000   { ... };
```

**冲突**：厂商把 **OVL0 放在 0x14000000**，而主线把 **MMSYS 放在
0x14000000**。同一地址不能既是 MMSYS 又是 OVL0。

可能的原因：
1. 厂商 BSP 的 MMSYS 基址其实不是 0x14000000（`dispsys` 节点没写 reg，
   地址硬编码在厂商驱动里）
2. 或者 MT6893 的 DDP 布局确实和 MT8192 不同（OVL0 从 mmsys 基址开始）

### 需要确认的信息

- 厂商内核里 `dispsys` 的基址（`mtk_disp_*` / `mtk_drm_ddp` 的
  `DISP_REG_*` 宏定义）
- 主线 `mt6893_mmsys_driver_data` 对应的 DTS 节点写法（`mt8192.dtsi` 是
  同代参考，但地址未必相同）
- MT6893 的 DDP 组件地址表（`DISP_OVL0`/`DISP_RDMA0`/… 的偏移）

**在拿到准确的寄存器基址前，DTS 节点不能凭猜测写**——写错地址轻则 probe
失败，重则访问到别的硬件模块。

### 下一步

1. 从厂商内核源码找 `dispsys` 的基址定义（`mt6893` 的 `dispsys` 节点是
   时钟提供者，基址在驱动里）
2. 对照 MTK BSP 的 `mt6893/mt6885` 的 `disp_dts_gpio` / `DDP` 地址表
3. 确认后再写 dtsi 节点


## 显示 DTS 的机械迁移（2026-11-07，驱动不变）

用户确认：显示驱动不需要动，只迁移 DTS。按此重新分类——**保持厂商
`mediatek_v2` DRM 驱动不变**，只把节点从切片文件重组成 SoC/板级两部分。

### SoC 部分：84 个节点，993 行

已提取到 `work/chopin-display-soc.dtsi`：

| 类别 | 节点 |
| --- | --- |
| 显示子系统配置 | `dispsys_config@14116000`、`disp_mutex@14117000` |
| DDP 主路径（DSI0） | `disp_ovl@14000000/14001000/14002000`、`disp_rdma@14003000/14005000`、`disp_wdma@14006000`、`disp_color@14007000`、`disp_ccorr@14008000`、`disp_aal@14009000`、`disp_gamma@1400a000`、`disp_dither@1400b000`、`disp_postmask@1400d000`、`disp_rsz@1400c000` |
| DDP 第二通道（DSI1） | 1410xxxx 段同样的组件 |
| DSI / PHY | `dsi@1400e000`、`dsi@1410e000`、`mipi_tx_config@11e50000/11e60000`、`disp_dsc_wrap@14124000` |
| MDP（相机/视频后处理） | `mdp_*@1f0xxx` 共 29 个 |
| SMI / LARB | `disp_smi_common@14002000`、`mdp_smi_common@1f002000`、`smi_larb4/5/13-20` |
| PWM / 其他 | `disp_pwm0@1100e000`、`disp_merge@14015000/14115000`、`dsi_te`、`mtkfb` |

### 板级部分：面板

当前生效的面板是 **`l16_36_02_0b_dsc_vdo,lcm`**（不是 k10a；DTS 里两个都
有，`panel@0` 用 l16）。已提取到 `work/chopin-panel.dtsi`：

```dts
&dsi0 {
	panel@0 {
		compatible = "l16_36_02_0b_dsc_vdo,lcm";
		reg = <0x0>;
		reset-gpios = <&pio 42 GPIO_ACTIVE_HIGH>;
		dvdd-supply = <&...>;
		backlight = <&...>;
		port {
			endpoint {
				remote-endpoint = <...>;
			};
		};
	};
};
```

### 迁移还剩的步骤

1. 给 `dsi@1400e000` 加 label（厂商节点没有 label，`&dsi0` 需要它）
2. 把 `chopin-display-soc.dtsi` 的 84 个节点并入 `mt6893.dtsi`（或作为
   `mt6893-display.dtsi` 被 dtsi include）
3. 把 `chopin-panel.dtsi` 并入板级 dts
4. 编译 → 刷 boot → 验证显示（DPMS off/on，不做真 suspend）

**驱动不变**：`CONFIG_DRM_MEDIATEK_V2=y` 保持，`CONFIG_DRM_MEDIATEK=y`
（主线）可以继续留着不绑定。

### 注意：厂商时钟提供者

显示节点用 `dispsys`（厂商的显示时钟/复位提供者，节点无 reg）作为时钟源，
而 `mt6893.dtsi` 里已经有主线的 `clk-mt6893-mm`。两者在现有工作 DTB 里
**共存**（显示正常工作），迁移时保持这个状态即可，不要删除任一。


## 显示 DTS 接入构建时的阻碍（2026-11-07 实测）

把 `chopin-display-soc.dtsi`（84 节点，dsi 加了 `dsi0:` label）接入构建，
替换原来的 `mt6893-xiaomi-chopin-display.dtsi`，编译失败：

```
Error: mt6893-xiaomi-chopin-camera.dtsi:10.3-23
       Properties must precede subnodes
```

原因还是**切片问题**：原 `-display.dtsi` 的结尾是一个未闭合的
`mfb@15810000 {`，由后面 `-camera.dtsi` 开头接上（`mfb@15812000` 的属性
和闭合）。把 display 换成自包含的新文件后，camera 的切片失去了前半段，
节点结构就坏了。

**结论：显示节点不能单独替换，必须和 camera 切片一起处理**——而 camera
又依赖后面的 multimedia 切片……这就是任意切片的连锁依赖。

### 唯一干净的解法

用已经验证过能编译的**剪枝单文件 DTS**（`work/chopin-pruned.dts`，
8,740 行 / 965 节点）**整体替换** 30 个切片：

1. 把 `mt6893-xiaomi-chopin.dts` 改成包含这一个文件（或直接用它）
2. `mt6893.dtsi` 保持现状（它的节点已经在剪枝文件里合并过，注意去重）
3. 编译 → 对比 DTB → 刷 boot 验证

这样：
- 切片依赖问题一次性消失
- 死节点已经剪掉（12,096 → 8,740 行）
- 后续任何外设迁移都在**自包含**的文件里做，不再有跨文件节点

风险：单文件里混着厂商和主线节点，phandle 是数字形式。但**能编译、
能启动**是已验证的（剪枝版 dtc 通过），比继续在切片上打补丁安全。

### 当前状态

- 显示 DTS 拆分产物保留：`work/chopin-display-soc.dtsi`、`work/chopin-panel.dtsi`
- 构建已回退到原切片文件，DTB 282,313 字节（与工作版本一致）
- `mt6893-display.dtsi` 已从源码树移除

## 参考

- `port/mt6895-linux/arch/arm64/boot/dts/mediatek/mt6895.dtsi`（同代 SoC 的
  mmsys/DDP 节点写法）
- `drivers/gpu/drm/mediatek/mtk_drm_drv.c` 的 mt6893 数据
- 厂商 DTS：`mt6893-xiaomi-chopin-display.dtsi`（738 行，面板初始化参数）
