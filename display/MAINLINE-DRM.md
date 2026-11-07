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

## 参考

- `port/mt6895-linux/arch/arm64/boot/dts/mediatek/mt6895.dtsi`（同代 SoC 的
  mmsys/DDP 节点写法）
- `drivers/gpu/drm/mediatek/mtk_drm_drv.c` 的 mt6893 数据
- 厂商 DTS：`mt6893-xiaomi-chopin-display.dtsi`（738 行，面板初始化参数）
