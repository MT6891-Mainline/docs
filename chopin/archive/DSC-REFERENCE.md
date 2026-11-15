# chopin (MT6893) DSC 黄金参考 —— 从 LK 正在运行的实例里读出来的

来源：`ringfix84` 的 `drm` 寄存器 dump（`dsc0` 段），即**启动引导阶段 LK 配置好、
并且此刻正在正常驱动面板的**那一套原值。用途：B 方案（自研 DSC）逐项对照复刻。

## 1. DSI 侧（压缩模式，`0x1400e000`）

| 偏移 | 值 | 解码 |
| --- | --- | --- |
| `+0x01c` PSCTRL | `0x2c050438` | `PS_WC = 0x0438 = 1080`；`PS_SEL = 5`（**DSC 压缩模式**）|
| `+0x038` SIZE_CON | `0x09600168` | `WIDTH = 0x168 = 360 = PS_WC/3` ✓，`HEIGHT = 0x960 = 2400` |
| `+0x000` START | `0x00010001` | 视频模式运行中 |
| `+0x014` MODE_CTRL | `0x1` | SYNC_PULSE |
| `+0x018` TXRX_CTRL | `0x3c` | 4 lanes |
| `+0x050/+0x054` | `0x14 / 0x10` | HSA/HBP WC |

## 2. DSC wrap（`0x14124000`）

| 偏移 | 值 | 解码 |
| --- | --- | --- |
| `+0x000` CON | `0x00010009` | `DSC_EN`(bit0) + bit3 + `UFOE_SEL`(bit16) |
| `+0x018` PIC_W | `0x01670438` | `w = 1080`，`(w/3)-1 = 359 = 0x167` |
| `+0x01c` PIC_H | `0x095f095f` | `h-1 = 2399 = 0x95f`（两个半字相同）|
| `+0x020` SLICE_W | `0x00b3021c` | `slice_w = 540 = 0x21c`，`(540/3)-1 = 179 = 0xb3` |
| `+0x024` SLICE_H | `0x00770013` | `slice_h-1 = 19 = 0x13`，高位 `0x77` |
| `+0x028` CHUNK_SIZE | `0x0000021c` | `540`（= slice_w）|
| `+0x02c` BUF_SIZE | `0x00002a30` | `10800 = 540 × 20` = chunk × slice_h ✓ |
| `+0x030` MODE | `0x00010101` | `slice_mode = 1`（**2 slices**）|
| `+0x034` CFG | `0x00000022` | 对应厂商 `dsc_cfg = 34` 分支（`0x22`，非 `0x0828`）|
| `+0x038` PAD | `0` | |
| `+0x03c` ENC_WIDTH | `0` | **本 SoC 未使用**（MT6895 的驱动会写它，此处为 0）|
| `+0x070` OBUF | `0x0000055d` | |
| `+0x200` SHADOW | `0x00000023` | |

## 3. PPS（`0x80 .. 0xCC`，20 个字，小端字节序）

```
PPS00 0x000c8089   PPS01 0x020e00aa   PPS02 0x00710020   PPS03 0x000c0007
PPS04 0x0516050e   PPS05 0x1ba01800   PPS06 0x20000c03   PPS07 0x330b0b06
PPS08 0x382a1c0e   PPS09 0x69625446   PPS10 0x7b797770   PPS11 0x00007e7d
PPS12 0x00800880   PPS13 0xf8c100a1   PPS14 0xe8e3f0e3   PPS15 0xe103e0e3
PPS16 0xd943e123   PPS17 0xd185d965   PPS18 0xd1a7d1a5   PPS19 0x0000d1ed
```

字节流（供面板 `0xC1` 写入对照）：
`89 80 0c 00 aa 00 0e 02 20 00 71 00 07 00 0c 00 0e 05 16 05 00 18 a0 1b 03 0c 00 20 …`

注意：**与面板初始化表里那条 `0xC1`（16 字节，`{0x89,0x28,0x00,0x14,…}`）不同** ——
面板那条是 DDIC 侧 PPS，这里的 20 个字是 **DSC 编码器侧** 的实时值；B 方案要复刻的是后者。

## 4. 路由 / mutex（已实测，与我们的表一致）

| 位置 | 值 | 含义 |
| --- | --- | --- |
| `mmsys +0xf54` | `0x3` | `DSI0_SEL_IN ← DSC_WRAP0`（**不是** DITHER0=`0x1`）|
| `mmsys +0xf80` | `0x1` | `DSI0_DSC_WRAP_SOUT_SEL` 使能 |
| `mmsys +0xf50` | `0x8` | DITHER0_MOUT_EN = DSC（bit3）|
| `mmsys +0xf4c` | `0x99` | OVL blendout（LK 原值）|
| `mutex +0x030` MOD0 | `0x261f7bf7` | 含 `DSC0 = BIT(29)` ✓ |
| `mutex +0x02c` | `0x41` | SOF/EOF = DSI0 |
| `mm_disp_dsc_wrap` 时钟 | 546 MHz，**Y（开）** | 与 DSC 组件相关 |
| `mipi_tx0_pll` | 本次 dump 为 `0`（PHY 已释放）| adopt 模式下我们不再关闭它（ringfix79 ✓）|

## 5. 复刻要点（B 方案 checklist）

1. `mtk_disp_dsc` 组件：按上表 CON/MODE/CFG/SLICE/CHUNK/BUF + 20 字 PPS 编程（**不要**写 ENC_WIDTH）；
2. `mmsys` 路由：`0xf50 = 8`、`0xf54 = 3`、`0xf80 = 1`（表里已有对应宏，改成 DSC 分支）；
3. `mutex`：`DSC0 = BIT(29)`（表里已是 29 ✓）；
4. DSI：`PS_SEL=5`、`PS_WC = ((chunk+2)/3*3)*2 = 1080`、`SIZE_CON = (2400<<16)|(PS_WC/3) = 0x09600168`；
5. 面板：`dsi->dsc`（540×20、2 slices、8bpc/8bpp）+ 面板侧 PPS；
6. genpd：DISP/DISP1/MM_INFRA（否则"看起来像 DSC 没生效"）；
7. 切换后逐项 diff 本文件的寄存器值 —— 每个值都有黄金参考。
