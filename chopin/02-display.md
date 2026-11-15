# 02 · 显示

用的**厂商 `mediatek_v2` DRM 栈**，不是主线 mtk-drm。

## 为什么用厂商栈

自研/主线 DRM 路线在 DSC（压缩）和 DSI 上卡死；厂商栈有 73 个 C 文件、11 万行，
但"能亮"是第一优先级。评估与放弃过程见
[`archive/MAINLINE-DRM.md`](archive/MAINLINE-DRM.md)、[`archive/V2-IMPORT.md`](archive/V2-IMPORT.md)、
[`archive/V2-DRM-AUDIT.md`](archive/V2-DRM-AUDIT.md)。

## 花屏 / underflow 的真正原因

不是笼统的"DRM 不完善"。两个具体缺口（[`archive/DRM-CLOCK-GAP.md`](archive/DRM-CLOCK-GAP.md)）：

1. **MTCMOS 时钟 provider 缺失** → 某个显示通路拿不到时钟；
2. **mmsys 被自己的子设备抢走** → `mtk-mmsys` 探测顺序问题。

表现为 `DDP_COMPONENT_DSI0: buffer underrun` 和首次 doze 时约 100 ms 的花屏。

## 已经修掉的两个真 bug

| Bug | 现象 | 修法 |
| --- | --- | --- |
| fbdev 扫描缓冲未清零 | 开机 logo 上叠横向彩条 | `drm_fbdev_dma.c` probe 尾部 `memset` 清零（内核 `f1c79a7`） |
| 面板 refcount 没初始化 | `refcount_t: addition on 0` → `failed to attach primary display`，偶发进不去桌面 | `panel-k10a-bringup.c` 补 `kref_init()`（内核 `dd3da5e`） |

> 注意：`mediatek_v2/mtk_drm_fbdev.c` 里有个长得一样的 `mtk_fbdev_probe()`，但它
> **不参与编译**，补丁打在那里是死代码——第一版就踩了这个坑。

## 面板

DSI `k10a_36_02_0a_dsc_vdo`，驱动是 `drivers/gpu/drm/panel/panel-k10a-bringup.c`
（移植仓库里自己写的，不是上游驱动）。DSC 的黄金参考是从 **LK 正在运行的实例**里读出来的
寄存器 dump（[`archive/DSC-REFERENCE.md`](archive/DSC-REFERENCE.md)）。

面板安全注意见 [`archive/MAINLINE-DRM.md`](archive/MAINLINE-DRM.md) 末节。
