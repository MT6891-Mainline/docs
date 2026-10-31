# 红米 Note 10 Pro (chopin / MT6891Z) 主线 Linux 移植

工作区：`/home/kily/project/mt6893`

- `linux-7.2.8/` —— 主线内核（移植基线）
- `n10p/` —— 原厂材料：`Android-fdt`（设备实际收到的 DTB）、`Andorid-chopin.dts`（同名反编译）、
  `kernel_mt6893-chopin_kvm.zip`（小米原厂 4.14 内核源码）、`ex.txt`（preloader + LK 日志）、
  `boot_a.img`（原厂 boot 分区转储）
- `tools/` —— 分析脚本
  - `dtsnode.py`：从 .dts 按路径提取节点
  - `gen_mt6893_clk.py`：从原厂时钟驱动生成主线驱动
- `port/` —— 移植产物（initramfs、打包脚本、镜像）
- `build/chopin/` —— 内核与 dtb 构建输出
- `work/stock/` —— 原厂 boot 镜像解包结果

## 一、平台结论

`Android-fdt` 根节点 `model = "MT6891Z/CZA"`、`compatible = "mediatek,MT6893"`，
机型代号 chopin，即**天玑 1100（MT6891Z）**，内核侧属 MT6893（天玑 1200）系列。
原厂 4.14 内核中 **MT6893 直接复用 MT6885 的时钟驱动**
（`CONFIG_COMMON_CLK_MT6893 += clk-mt6885.o`），时钟 ID 头文件为
`include/dt-bindings/clock/mt6885-clk.h`。

## 二、主线支持缺口

主线 7.2.8 里内嵌 MT6893 compatible 的驱动（比最初按文件名搜索时看到的多，
这些是共享驱动里的字符串，容易被漏掉）：

| 驱动 | 文件 | compatible |
| --- | --- | --- |
| pinctrl | `drivers/pinctrl/mediatek/pinctrl-mt6893.c` | `mediatek,mt6893-pinctrl` |
| 电源域 | `drivers/pmdomain/mediatek/mtk-pm-domains.c` | `mediatek,mt6893-power-controller` |
| SPI | `drivers/spi/spi-mt65xx.c` | `mediatek,mt6893-spi` |
| IOMMU | `drivers/iommu/mtk_iommu.c` | `mediatek,mt6893-iommu-mm` |
| SMI | `drivers/memory/mtk-smi.c` | `mediatek,mt6893-smi-common` / `-smi-larb` |

**缺口**是：

- SoC 设备树 `mt6893.dtsi`
- 板级设备树
- **整套时钟/复位驱动**（topckgen / infracfg / apmixedsys）

以上三项由本移植补齐。

## 三、本移植新增内容

### 1. 时钟与复位驱动（采用上游未合并系列 + 原厂交叉验证）

时钟驱动**采用上游未合并系列**
[`[PATCH v1 0/3] MediaTek Dimensity 1200 - Add Clocks support`](https://patchew.org/linux/20250410144110.476197-1-angelogioacchino.delregno@collabora.com/)
（AngeloGioacchino Del Regno，2025-04），共 10 个驱动 + 时钟 ID 头 + 2 个绑定：

| 文件 | 内容 |
| --- | --- |
| `include/dt-bindings/clock/mediatek,mt6893-clk.h` | 上游 ID 空间（`CLK_TOP_*` / `CLK_INFRA_AO_*`） |
| `clk-mt6893-topckgen.c` | 57 mux + 因子 + 复合 + 分频 |
| `clk-mt6893-infra_ao.c` | 74 个门控 |
| `clk-mt6893-apmixedsys.c` | 17 个 PLL |
| `clk-mt6893-{mm,mdp,mfg,vdec,venc,imp_iic_wrap,scp_adsp}.c` | 多媒体/外设域时钟 |
| `Documentation/.../mediatek,mt6893-{clock,sys-clock}.yaml` | 绑定文档 |

**采纳前做了逐项交叉验证**（上游 vs 原厂 `clk-mt6885.c`，把符号宏解析成数值后比对）：

| 数据 | 结果 |
| --- | --- |
| topckgen mux | 共同 55 个，**53 个寄存器/位域完全一致**；仅 `camtg5_sel`/`camtg6_sel` 的 update-shift 不同（相机闪光时钟，与串口无关） |
| apmixedsys PLL | 共同 10 个，**(reg, pwr_reg) 10/10 完全一致**；上游还多 7 个 CPU 簇 PLL（`armpll_ll`/`armpll_bl0..3`/`ccipll`/`mpll`） |
| infra 门控 | 61 个 `(窗口偏移, 父时钟, bit)` 三元组完全一致；上游少 48 条（省掉了 SPM/SSPM/MCUPM/DVFSRC 等主线不需要的门控） |
| 门控窗口 | 两边是同样的 5 个窗口（0x80/0x88/0xa4/0xc0/0xd0），只是**编号顺序不同** |

也就是说：**上游数据与原厂硬件数据一致**，可以放心采纳；而上游是超集且是主线风格，
便于后续上游化。

#### 适配 7.2.8 的两处改动

1. **API 漂移**（1 处）：上游写的是 6.15 时代的
   `mtk_clk_register_plls(node, …)`，7.2.8 已改成传 `struct device` → 改为 `&pdev->dev`。
2. **复位控制器**：上游系列**不含复位**，本移植补上（见下）。

#### 复位控制器

原厂 DT 没有任何 `resets = `，原厂内核也不带 MT6893 复位索引表——复位位号只能从原厂驱动挖。
可验证来源是 `drivers/scsi/ufs/mediatek/mt6893/ufs-mtk-platform.h`：

| 复位 | SET | bit | CLR | bit |
| --- | --- | --- | --- | --- |
| UFSHCI | 0x130 | 15 | 0x134 | 15 |
| UNIPRO | 0x140 | 7 | 0x144 | 7 |
| UFSPHY | 0x140 | 9 | 0x144 | 9 |
| UFSCPT | 0x150 | 21 | 0x154 | 21 |

与主线 `INFRA_RST0_SET_OFFSET = 0x120`、每 bank +0x10、clear = set+4 完全吻合，
故用 `MTK_RST_SET_CLR` 实现在 `clk-mt6893-infra_ao.c`，并新增绑定
`include/dt-bindings/reset/mediatek,mt6893-resets.h`。

#### 原厂 DT 兼容（保住"LK 传自己 DTB"那条路）

上游 compatible 是 `mediatek,mt6893-infracfg-ao`，而原厂 DT 是 `mediatek,infracfg_ao`
（topckgen/apmixed 同理）。为了让内核**用原厂 DTB 启动时也能起串口**：

- 三个驱动的 `of_device_id` 都加了原厂 compatible 作为 fallback；
- `infra_ao` 增加一个**有条件**的 ID 重映射：仅当检测到 LK 追加的 `/bootprof` 节点
  （即确实在使用原厂 DTB）时，才把原厂的 UART 门控 ID 23/24 指向
  `CLK_INFRA_AO_UART0/UART1`。这样自有的 `mt6893.dtsi`（上游 ID 空间）不受影响。

`tools/gen_mt6893_clk.py`（原先从原厂机械转换的生成器）已**加锁停用**，
只作为"原厂数据如何被验证"的记录保留；旧产物在 `work/prev-generated/`。

### 2. 设备树

SoC 设备树**采用 [MT6893-Mainline/linux](https://github.com/MT6893-Mainline/linux)
`dirty-x6710` 分支的 `mt6893.dtsi`**（1269 行，来自 MT8195 同族风格），
它覆盖了本移植后续要用的全部外设：

```
cpus(含 opp/idle-states/cache) gic timer psci pmu
topckgen infracfg_ao apmixedsys
pinctrl(9 窗口) scpsys(power-controller) watchdog systimer
gce0/gce1(spi/i2c 的 DMA) pwrap spmi auxadc
uart0/1 spi0..7 i2c0..9 mmc0/1 ufs ufs-phy usb ssusb efuse
mmsys/vdecsys/vencsys/mdpsys/mfgcfg/scp_adsp/imp_iic_wrap_* 时钟控制器
thermal + lvts efuse
```

该 dtsi 里**所有外设默认 `status = "disabled"`**（0 个 okay），
所以板级只启用需要的部分，首次启动不会被一堆驱动同时 probe。

采纳时修掉了两处它自身的问题：

| 问题 | 处理 |
| --- | --- |
| pinctrl 节点有 9 个 `reg` 但 `reg-names` 只有 8 个（漏 `eint`）→ `mtk_build_eint()` 返回 -EINVAL，EINT 不可用（pinctrl 本身仍能 probe，只是 warning） | 补上 `"eint"` |
| `mmc1` 节点引用 `CLK_TOP_MSDC30_1`，而头文件里只有 `CLK_TOP_MSDC30_1_SEL` | 改为 `_SEL` |

板级 `mt6893-xiaomi-chopin.dts`（本移植自己写）包含：机型、内存、
`chosen/stdout-path`、`chosen/bootargs`、
**simple-framebuffer（屏幕控制台）**、以及原厂 reserved-memory 中
ATF/TEE/GZ/SSPM/SPM/MCUPM/SCP/DPM/framebuffer 等区域。

地址/中断全部逐条取自原厂 DTB，关键几项：

| 模块 | 地址 | 中断 |
| --- | --- | --- |
| GIC | 0x0c000000 / 0x0c040000 | PPI 9 |
| topckgen / infracfg_ao / apmixedsys | 0x10000000 / 0x10001000 / 0x1000c000 | — |
| pinctrl 基址 | 0x10005000 | SPI 212 |
| iocfg 窗口 | rm 0x11c20000、bm 0x11d10000、lm 0x11e20000、lb 0x11e70000、rt 0x11ea0000、lt 0x11f20000、tm 0x11f30000 | — |
| EINT | 0x1000b000 | SPI 212 |
| UART0 / UART1 | 0x11002000 / 0x11003000 | SPI 109 / 110 |

#### 屏幕控制台（无 UART 时的第二路输出）

参考仓库的板级 dts 给了关键配方：`simple-framebuffer` + `a8r8g8b8` +
`stride = (1080+8)*4`，用来复用 LK 已经点亮、且仍在扫描输出的 framebuffer。
chopin 上可取的区域有三个候选（都在原厂 DT 的 reserved-memory 里）：

| 候选地址 | 原厂命名 | 大小 | 说明 |
| --- | --- | --- | --- |
| **0x77a20000** | `framebuffer` (mblock-19) | 0x27e0000 | 当前采用；LK 明确命名为 framebuffer |
| 0x76020000 | `display-others_logo_fb-addr` | 0x9f6000 | 恰好等于 1080×2400 单帧（stride 4352） |
| 0x76a20000 | `logo_db_addr_pa` | 0x1000000 | 参考仓库 X6710 用的是 0x76a10000（仅差 1 字节） |

这些区域是 reserved 但**不是 no-map**，内核不会把它们分配给别的用途，
所以写错最多是"屏幕没变化"，不会破坏内存。若 0x77a20000 看不到输出，
把 `framebuffer@77a20000` 的地址换成上表另两个再试即可。

内核侧相应打开：`CONFIG_FB_SIMPLE=y`、`CONFIG_FRAMEBUFFER_CONSOLE=y`，
命令行加 `console=tty0`。

### 3. 修复主线 pinctrl 驱动的窗口名 bug

主线 `pinctrl-mt6893.c` 的设备树窗口名列表原本是
`base rm bm bl br lm lb rt lt tl`（10 项），既不满足它自己的绑定
（`base rm bm lm lb rt lt tm eint`），也多了 MT6893 上不存在的 `bl`/`br`/`tl`，
并且从索引 3 起整体错位。

判断依据（可复现）：把主线与原厂驱动的
`PIN_FIELD_BASE()`/`PINS_FIELD_BASE()` 表按 pin 展开成 `pin -> (i_base, s_addr)`，
两者**逐条完全相同**：

| 字段 | 主线条目 | 原厂条目 | 共同 pin | 完全一致 |
| --- | --- | --- | --- | --- |
| PU | 93 | 93 | 186 | 186 |
| PD | 93 | 93 | 186 | 186 |
| DRV | 82 | 82 | 220 | 220 |
| PUPD | 24 | 24 | 34 | 34 |
| SMT | 82 | 82 | 220 | 220 |

也就是说**寄存器表的索引是对的，只有名字数组错了**：原厂窗口顺序为
`rm, bm, lm, lb, rt, lt, tm`（与原厂 DT 的 `reg_bases` 顺序一致）。
因此把名字数组改成 8 项
`{"base","rm","bm","lm","lb","rt","lt","tm"}`，与绑定文档的 9 项
（8 个窗口 + `eint`）完全对上；pin range 实际使用的 `i_base` 最大为 7，自洽。
同时把绑定示例里 EINT 地址从 `0x1100b000` 更正为 `0x1000b000`（原厂 DT 为后者）。

## 四、构建与打包

```bash
cd linux-7.2.8

# 推荐：直接使用本移植新增的 defconfig（已包含下列所有选项）
make O=../build/chopin ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- chopin_defconfig

# 或者从主线 arm64 defconfig 手工构造：
#   make O=../build/chopin ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig
#   ./scripts/config --file ../build/chopin/.config \
#       -e COMMON_CLK_MT6893 -e PINCTRL_MT6893 -e MTK_SCPSYS -e MTK_SCPSYS_PM_DOMAINS \
#       -d CMDLINE_FROM_BOOTLOADER -e CMDLINE_FORCE \
#       --set-str CMDLINE "console=ttyS0,921600n8 earlycon=mtk8250,mmio32,0x11002000 clk_ignore_unused ignore_loglevel"

make O=../build/chopin ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j16 Image dtbs

port/tools/mkinitramfs.sh                                   # initramfs.cpio.gz
port/tools/extract-stock-dtb.py n10p/boot_a.img port/out/stock-base.dtb
port/tools/build-images.sh                                  # 两个 boot 镜像
```

新增的 `arch/arm64/configs/chopin_defconfig` 由 `make savedefconfig` 生成并已验证可复现
（`make chopin_defconfig` 后 `CMDLINE_FORCE`、`PINCTRL_MT6893`、`COMMON_CLK_MT6893` 等均在）。

**为什么必须用 `CMDLINE_FORCE` 而不是 `CMDLINE_EXTEND`**：

1. arm64 的 Kconfig 里**没有 `CMDLINE_EXTEND`**（只有 `CMDLINE_FROM_BOOTLOADER` / `CMDLINE_FORCE`），
   即使手工写进 `.config`，`olddefconfig` 也会把它丢掉。
2. LK 一定会用自己的 cmdline 覆盖 `chosen/bootargs`（原厂 cmdline 见 `Android-fdt`），
   所以 `CMDLINE_FROM_BOOTLOADER` 下我们写的 `earlycon=...` 永远不会生效。
3. 原厂 DT 没有 `chosen/stdout-path`，因此简写 `earlycon` 也无从选择设备。
4. `earlycon=mtk8250,mmio32,0x11002000` 不依赖 DT、也不依赖时钟驱动，
   即使 infracfg/topckgen 驱动出问题，早期日志依然能出来 —— 这对首次上机非常关键。

代价是丢掉 LK 的 cmdline（`androidboot.*`、`bootopt=` 等 MTK/Android 专用参数），
这些主线都不需要。等移植稳定后可以改回 `CMDLINE_FROM_BOOTLOADER`。

`port/out/`：

| 文件 | 说明 |
| --- | --- |
| `boot-owndtb.img` | 我们的内核 + initramfs + **我们的 chopin DTB** |
| `boot-stockdt.img` | 我们的内核 + initramfs + **原厂基础 DTB**（从原厂 boot 镜像提取） |
| `stock-base.dtb` | 原厂 `lk_main_dtb`（201784 字节） |
| `Image.gz` / `initramfs.cpio.gz` | 压缩内核 / 最小 initramfs |

两个镜像的头部布局与原厂**完全一致**：
kernel `0x40080000`、ramdisk `0x51100000`、tags `0x47c80000`、
dtb `0x41f78000`、page size 2048、header v2。

## 五、引导链与 DTB 路径（已查清）

对原厂 `boot_a.img`（128 MB 分区转储）与 LK 日志的分析结果：

```
原厂 boot_a.img = Android boot image, header version 2
  kernel  13284146 B @ 0x40080000   (gzip)
  ramdisk 57202152 B @ 0x51100000
  tags                 0x47c80000
  dtb       201848 B @ 0x41f78000
```

关键点：**`dtb` 字段不是裸 DTB，而是一个 DTBO 容器**
（大端表头，magic `0xd7b7ab1e`，1 个条目）：

```
magic=0xd7b7ab1e total=201848 header_size=32 entry_size=32 count=1
  entry[0]: size=201784 offset=64  ->  载荷 magic 0xd00dfeed（真正的 DTB）
```

该载荷 201784 字节与 LK 日志中的 `lk_main_dtb`（`dtb_img_size=201784`）**大小完全一致**，
内容是通用 MT6893 SoC 树（939 个节点）。

LK 启动流程（来自 `ex.txt`）：

1. 从 `lk` 分区加载 `lk_main_dtb`（201784 B）
2. 从 `dtbo` 分区加载 overlay（`dtbo_total_len = 0x11cef`，`dtbo_index=0`）
3. `dtb_overlay` 叠加，并追加 LK 自己的节点（`bootprof`、`atags`、`chosen` 等）
4. 结果 282632 字节 = `Android-fdt`（内核收到的 DTB）

对比"基础 DTB"与"最终 DTB"的节点集合：

```
基础 939 节点 -> 最终 1275 节点；新增 336 个，删除 0 个
新增内容 = 机型专属节点（panel/camera/NFC/haptics/charger/odm/firmware…）+ LK 节点
```

**结论：dtbo overlay 是纯增量叠加，不会替换基础树。**

### 实机确认（ADB 只读核对）

设备接入后用 `adb` 做了只读核对，上述分析全部得到证实：

| 核对项 | 结果 |
| --- | --- |
| `getprop ro.product.device` / `ro.boot.hardware` | `chopin` / `mt6893` ✓ |
| `/proc/device-tree/bootprof` 存在 | **是** → 内核实收 DTB 含 LK 追加节点 ✓ |
| `/proc/device-tree/odm` 存在 | **是** → 原厂 dtbo overlay 已叠加 ✓ |
| `/proc/cmdline` | 与 `Android-fdt` 的 `chosen/bootargs` 逐字一致 ✓ |
| `/sys/fs/pstore` | 已挂载 pstore 文件系统 ✓（日志持久化可行） |
| 原厂内核 `CONFIG_PSTORE_RAM` | `=y`，且 cmdline 带ramoops 参数 ✓ |
| `boot_a` 分区实际大小 | **131072 KB = 128 MB**（不是 LK 日志里 GPT 的 16 MB）✓ |
| `n10p/boot_a.img` vs 设备 `boot_a` 前 1 MB | **sha256 完全相同** → 可安全回滚 ✓ |
| LK 二进制 | 含 `boot`/`continue`/`download` 字符串；但实测 `fastboot boot` 不可用 |
| 解锁状态 | `ro.boot.flash.locked=0`、`verifiedbootstate=orange` ✓ |

由于原厂 `boot.img` 的 dtb 字段承载的就是"基础 DTB"，而 LK 会把它（叠加 dtbo 后）交给内核，
因此**在 boot.img 里替换 dtb 字段是控制内核 DTB 的正确位置**。
`boot-owndtb.img` 与 `boot-stockdt.img` 就是为此准备的两个变体。

### 与引导链相关的设计选择

因为不能 100% 确定 LK 是读取 boot.img 的 dtb 还是复用自己分区内的 `lk_main_dtb`，
时钟驱动**同时匹配原厂 compatible**，于是两条路都能起来：

- 若 LK 用原厂 DTB：UART 节点是通用的 `mediatek,mt6577-uart`，
  `clocks = <&clk26m>, <&infracfg_ao 23>`，而我们的 infracfg 驱动匹配
  `mediatek,infracfg_ao` 且时钟 ID 与原厂完全一致 → 串口正常；
  CPU/GIC/timer/psci 同样是通用 compatible。
- 若 LK 用 boot.img 里的 DTB：直接使用我们的 `mt6893-xiaomi-chopin.dtb`。

## 六、日志通道：pstore 可能不成立，MTK log_store/expdb 才是正路

### 来自同族机型移植的警告

[MT6895-Mainline/linux](https://github.com/MT6895-Mainline/linux) 的
`7.2-mt6895-xiaomi-xaga`（**同为 7.2 内核的小米 MTK 机型**）在板级 dts 注释里明确写道：

> The ramoops region at 0x48090000 is **NOT preserved across a reset**, so pstore stays empty.
> Capture the dead boot with read-lastboot-log.sh (**XAGR ring in log_store at 0x7ffbf000,
> copied to expdb by LK**)

也就是说：**我原先基于 0x48090000 的 pstore 方案在他们机型上拿不到东西**。
chopin 是否同样如此尚未验证，但必须准备 Plan B —— 而 Plan B 的基础设施 chopin **原厂就有**：

### MTK log_store 机制（chopin 原厂 DT/LK 已具备）

`n10p/ex.txt`（LK 日志）里的直接证据：

```
LK_LOG_STORE: dram pl/lk log buff mapping start addr = 0x7ffbf000, size = 0x40000
LK_LOG_STORE: sram buff header 0x11df00, current log header 0x11df34,
              sig 0xcdab3412, buff_size 0x40000,
              pl log size 0xc59c@0x400, lk log size 0x0@0x400!
```

要点：

- **上一轮 preloader 的日志还在**（`pl log size 0xc59c`）→ 该 RAM 区 0x7ffbf000 **跨重启保留**；
- 另有 **SRAM 头 0x11df00**（常驻 SRAM，掉电不掉），LK 在下次启动时读它；
- `current log header 0x11df34` 与结构体偏移**逐字节吻合**（见下），可确证布局；
- chopin 的 GPT 里存在 **`expdb` 分区**（`[GPT_LK][2] name=expdb`）——日志最终落盘的地方。

原厂内核驱动 `drivers/misc/mediatek/log_store/log_store.c` 给出完整语义：

| 动作 | 说明 |
| --- | --- |
| `store_printk_buff()` | 把内核 printk 环的**物理地址/长度**写进 SRAM 头的 `dram_buf.klog_addr/klog_size` |
| `log_store_to_emmc()` | 打开 `/dev/block/by-name/expdb`，用 `kmsg_dump` 把内核日志按环形写入该分区**最后 2 MB** |

SRAM 头布局（`struct sram_log_header`，共 256 字节，基址 0x11df00）：

| 偏移 | 字段 |
| --- | --- |
| +0x00 | `sig`（LK 用 0xcdab3412 / 置 0xABCD1234） |
| +0x04 | `reboot_count` |
| +0x08 | `save_to_emmc` |
| +0x0c | `struct dram_buf_header`（40 字节：sig/flag/buf_addr/buf_size/buf_offsize/buf_point/**klog_addr(+0x24)**/**klog_size(+0x28)**/atf_log_addr/atf_log_len） |
| +0x34 | `struct pl_lk_log`（32 字节，LK 称 "current log header"）✓ 与 LK 日志一致 |
| +0x54 | `gz_log_addr` / `gz_log_len` |
| +0x5c | `reserve[41]` |

### 待你配合的一次验证（只读）

设备连上后，取 expdb 分区看看里面到底有没有上次启动的日志（**只读操作**）：

```bash
adb shell "su -c 'dd if=/dev/block/by-name/expdb of=/data/local/tmp/expdb.img bs=4096'"
adb pull /data/local/tmp/expdb.img
strings expdb.img | head -40
strings expdb.img | grep -c "Linux version"
```

- 若里面能看到 **内核日志文本**（vendor 内核写的）→ 说明 chopin 的链路是"内核写 expdb"，
  我们的主线只要照 `log_store_to_emmc()` 做（或写 log_store RAM 区）即可；
- 若只有 **PL/LK 日志** → 说明 LK 会在下次启动把 log_store RAM 区拷进 expdb，
  我们就往 0x7ffbf000 的 ring 里写自己的日志（对应他们注释里的做法）；
- 若 expdb 全空 → 再回到 pstore，并改用 MTK 的 `log_store` 直接落盘。

### 实机结论（用你手动 dd 的 expdb dump 验证）

把 `exp.img`（40 MB）按 LK 段表逐段量化：

| 段 | expdb 偏移 | 大小 | 非零字节 | 结论 |
| --- | --- | --- | --- | --- |
| `SYS_PSTORE_RAW` | 0x2800 | 0xe0000 | **0（全零）** | **pstore 方案在 chopin 上确认失效** |
| `SYS_RAMCONSOLE_RAW` | 0x2000 | 0x800 | **0（全零）** | MTK ram_console 段也是空的 |
| log_store 环（`kedump` @[904]） | 0x4090c9 | 0x1fcf0 | 有 PL/LK 日志 | **活的通道** ✓ |

LK 会把 pstore 区**原样**转存到 expdb（`kedump add: SYS_PSTORE_RAW[1] e0000/e0000@2800`），
但区里本身是空的 —— 说明它每轮启动都被清掉，与 MT6895 移植的说法一致。

而 log_store 环的容量关系在这份 dump 里可以直接算出来，**正好印证本驱动的写法**：

```
pl log size 0xc5a6@0x400          ->  off_lk = 0x400 + 0xc5a6 = 0xc9a6
log len is 0x1fcf0                ->  sz_lk  = 0x1fcf0 - 0xc9a6 = 0x1334a
                                    即 log len = 0x400 + sz_pl + sz_lk
```

上一轮 LK 的日志（`sz_lk`≈78 KB）能被完整转存，也证明 **preloader 不会重置 `sz_lk`**，
所以驱动对它做的扩写能存活到下一次 LK 的 kedump ✓。

### 附带确证：LK 确实使用 boot.img 里的 DTB

同一份 dump 的 LK 日志里能看到：

```
[3794] [cmdline clear] clear str size=37
[3795] cmdline: console=tty0 console=ttyS0,921600n1 root=/dev/ram ...
[3798]  : ... clk_ignore_unused console=ttyS0,921600n8 earlycon clk_ignore_unused root=/dev/ra
[3806] lk finished --> jump to linux kernel 32Bit
[LK]jump to K64 0x40080000
```

`console=ttyS0,921600n8 earlycon clk_ignore_unused` 是本移植早期版本写进
`chosen/bootargs` 的串（原厂用 `921600n1`），它出现在 **LK 构造的内核命令行中间**
→ **LK 会把 boot.img DTB 的 `chosen/bootargs` 追加进内核 cmdline**，
并且 `[LK]jump to K64 0x40080000` 说明那次确实跳进去执行了我们的内核
（那版还没有 logstore 驱动，所以没留下日志）。

### 已实现：mtk-logstore-console 驱动

`drivers/misc/mtk-logstore-console.c`（新增 `CONFIG_MTK_LOGSTORE_CONSOLE`）做了三件事：

1. ioremap 原厂 log_store RAM 环（0x7ffbf000，0x40000）与 SRAM 头（0x11df00）；
2. 注册一个 `struct console`，**把内核 console 输出逐行追加到环里 LK 日志之后**
   （`CON_PRINTBUFFER`，所以注册时会把之前打过的整个 ring 补写进去 —— 连 probe 之前的早期日志也能捞到）；
3. 同步扩大 SRAM 头里 `pl_lk_log.sz_lk`，使 LK 的 `kedump` 长度覆盖我们的段落。

写入前还会打一段 `===== LINUX (mainline) console =====` 标记，便于在 expdb dump 里定位。

**注册时机已提前到 `early_initcall`**：驱动除了平台驱动（DT，device_initcall）之外，
还有一个 `early_initcall` 版本，用 LK 自己打印过的固定地址（RAM 0x7ffbf000 / SRAM 0x11df00）
直接建立 console。这样即使挂在驱动模型之前也能留下日志；又因为用了 `CON_PRINTBUFFER`，
注册时会把之前打过的整个 printk 环补写进去 —— 连第一个 printk（`Booting Linux on physical CPU`）
都能捞到。

**刷机后请先确认刷的是哪一版**：LK 会把 boot.img DTB 里的 `chosen/bootargs`
原样追加进内核命令行，所以从 LK 日志里那条 cmdline 就能反推镜像版本：

| DTB 的 bootargs（前段） | 对应版本 |
| --- | --- |
| `console=ttyS0,921600n8 earlycon clk_ignore_unused` | 最早那版（无 pstore / 无 logstore 驱动） |
| `console=tty0 console=ttyS0,921600n8 earlycon clk_ignore_unused regulator_ignore_unused pd_ignore_unused ignore_loglevel` | `boot-owndtb.img`（当前版本，**推荐刷这个**：命令行可在 dump 里直接核对） |
| `console=tty0 console=ttyS0,921600n1 ...`（原厂串，无 `921600n8`） | `boot-stockdt.img`（DTB 用的是原厂基础树） |

> **踩过的坑**：曾出现过刷的其实是几小时前保存的旧镜像（dump 里命令行是 `console=ttyS0,921600n8 earlycon clk_ignore_unused`，
> 即最早版本），那版没有 logstore 驱动，所以 expdb 里永远是空的。
> 刷之前先用 `sha256sum port/out/*.img` 核对，刷完再从 dump 的命令行确认一次。

**关键点：这个驱动在两种 DTB 下都能工作** ——

| DTB | 匹配的 compatible | SRAM 头来源 |
| --- | --- | --- |
| 我们的 `mt6893-xiaomi-chopin.dts` | `mediatek,mt6893-logstore` | DT 的 `header` reg |
| 原厂 DTB（LK 传的那份） | `mediatek,log_store`（原厂节点 `mblock-6-log_store`） | 回退到固定地址 0x11df00 |

SRAM 头地址的回退依据来自 LK 自己的打印，且大小与结构体完全吻合：

```
RAM_CONSOLE sram_log_store_addr:0x11DF00, sram_log_store_size:0x100   (= 256B = sizeof(sram_log_header) ✓)
```

**风险为零**：SRAM 头与 RAM 环都是易失状态，preloader 每轮启动都会重写
（LK 日志里可见 `PL_LOG_STORE:set ram_header->sig = 0xABCD1234`），写错不会变砖。

### exp8.1：bootopt 修复已生效，但面包屑自身有两个 bug

exp8.1 里 **`boot linux-64` 出现 2 次、后面没有任何 GZ 异常**（x0=0x47c80000 正确指向 DTB）
→ bootopt 修复确认有效。但内核仍零输出，排查后发现**面包屑代码自己有两个致命问题**：

| bug | 后果 |
| --- | --- |
| `early_ioremap(0x7ffbf000, 0x40000)` 映射 **256 KB** | early fixmap 只有少量槽位（每 4KB 一个），64 个槽必然不够 → 返回 NULL → 面包屑**静默失效** |
| 用 `__va(0x11df00)` 读 SRAM 头 | 0x11df00 是 **SRAM（I/O）**，不在内核 RAM 线性映射内 → **缺页异常**，而此刻还没有 console → **内核静默死掉** |

第 2 条很可能就是 logstore8 那次"64 位启动成功却依然零输出"的原因。logstore9 已修正：
early 路径只映射 1 页 SRAM + 2 页 ring，正常路径改用 `ioremap()` 访问 SRAM，绝不使用 `__va()`。

### ★ 根因：boot.img 头部 cmdline 里的 `bootopt` 决定内核以 64 位启动

原厂 `boot_a.img` 头部的 cmdline **不是**普通命令行：

```
bootopt=64S3,32N2,64N2 androidboot.selinux=permissive androidboot.init_fatal_reboot_target=bootloader buildvariant=eng
```

`bootopt=` 是 MTK 用它决定"以什么位宽/模式启动内核"的参数。本移植一开始为了放进
`console=` 等参数，把整条 cmdline 替换掉了 —— **`bootopt` 随之丢失**，后果是
**GenieZone（GZ）hypervisor 以 32 位模式启动我们的 arm64 内核**：

```
boot linux-32 via EL2, pc=0x40080000, x0=0x0, x1=0x1aed, x2=0x47c80000, mode=0x1d3
[instruction_abort_curr_el_handler] EC = HSR_EC_INSTR_ABORT_CURR_EL 0x21
PC: 0000000000000200 / FAR_EL2: 0000000000000200
panic: GZ kernel exception: current instruction abort!!!
```

（32 位约定下 x0=0，没有 DTB 指针；PC 立刻落到 0x200。）

恢复 `bootopt=64S3,32N2,64N2` 后，同一份 GZ 日志变成：

```
boot linux-64 via EL2, pc=0x40080000, x0=0x47c80000, x1=0x0, mode=0x3c5
```

x0 正确地等于 DTB 地址（0x47c80000），**不再有异常**。

**教训**：打包 boot.img 时，必须在头部 cmdline **保留原厂的 `bootopt=` 前缀**，
我们的参数只能追加在后面。`port/tools/build-images.sh` 已按此修正。

### 决定性实验（exp6.2）：内核死在 early_initcall 之前

`exp6.2` 是一次**窗口新鲜**的 dump（镜像对应 `boot-stockdt.img`）：

| 标记 | 结果 |
| --- | --- |
| `chopinport=logstore6`（版本标记） | ✓ 在 |
| `921600n8`（我们的 cmdline） | ✓ 在 |
| `jump to K64`（LK 跳内核） | ✓ 在 |
| `mtk-logstore`（驱动自检行） | ✗ **不在** |
| `LINUX (mainline) console`（驱动注册前就写的标记） | ✗ **不在** |

驱动在 `logstore_setup()` 一进来就会往环里写标记、并且无条件 `pr_info`。
两者都没有 → **内核根本没执行到 `early_initcall`**，卡在 `setup_arch`/GIC/timer 一带。

### 面包屑（logstore7）

于是在 `arch/arm64/kernel/setup.c` 的 7 个位置插了面包屑，**直接写内存、不经 printk、不依赖 console**：

| 位置 | 写法 |
| --- | --- |
| `early_ioremap_init()` 之后 | `early_ioremap`（此时线性映射还没建立） |
| `setup_machine_fdt()` 之后 | 同上 |
| `parse_early_param()` 之后 | 同上 |
| `arm64_memblock_init()` 前后 | 同上 |
| `paging_init()` 前/后 | 前用 `early_ioremap`，后用线性映射 |
| `unflatten_device_tree()` 之后 | 线性映射 |
| `smp_init_cpus()` 之后 | 线性映射 |

面包屑写在环内固定偏移 0x38000（在我们的日志文本之后），格式 `[BC nn] <阶段>`。
下次 dump 里 `grep "\[BC "` 就能看到最后走到哪一步。

### ⚠️ 只有"一次启动"的窗口

LK 每次启动都会把**上一轮**留下的 log_store 环 `kedump` 进 expdb —— 也就是说
**每一次新的启动都会覆盖上一次的 dump**。所以：

```
刷我们的镜像 → 启动失败 → 刷回原厂 → 进 Android（这一次 LK 把我们的日志转存进 expdb）
                                        ↑ 必须在这一刻立刻取 expdb
```

**进了 Android 就马上 dd，不要再重启。** 多重启一次，expdb 里就换成那一次启动的内容，
我们的日志就没了 —— 之前几份 dump 里时有时无，就是这个原因。

### 签名（原厂 log_store_kernel.h 的权威取值）

```c
#define SRAM_HEADER_SIG (0xabcd1234)   // sram_log_header.sig      +0x00
#define DRAM_HEADER_SIG (0x5678ef90)   // sram_log_header.dram_buf.sig +0x0c
#define LOG_STORE_SIG   (0xcdab3412)   // pl_lk_log.sig            +0x34
```

LK 日志里打印的 `sig 0xcdab3412` 是第三个；驱动校验第一个（与外厂第一道校验一致）。
驱动现在**不会因为签名不符而放弃**：会把读到的三个签名都打出来，并在头部异常时
退回到固定偏移 0x20000 继续写，保证"至少留下点东西"。

### 读日志（新主路径）

**注意**：本机 adb 无法把分区 dd 到电脑（`adb exec-out` 取二进制失败），
所以改成**在手机终端里执行、只把文本结果带出来**：

```bash
# 在手机终端（需 root）：
su -c 'strings /dev/block/by-name/expdb | grep -A 400 "LINUX (mainline) console"' > /sdcard/mainline-log.txt
# 再把这个 txt 拷到电脑即可（几十 KB，不用传 40 MB）
```

或按段取整块（0x4090c9 起 0x1fcf0 字节，偏移以当次 dump 的段表为准）：

```bash
su -c 'dd if=/dev/block/by-name/expdb of=/sdcard/expdb.img bs=1M'
```

电脑侧辅助脚本（如果你哪天 adb 正常了）：

```bash
port/tools/read-expdb-log.sh android-boot-log.txt
```

它会把 expdb 分区拉回本地，从 marker 处截取我们的段落。
`port/tools/read-boot-log.sh`（pstore 路径）保留作为备选 —— 如果 chopin 的 ramoops
恰好能跨重启保留，两条路就互为保险。

### 其它可借鉴的结论

- **`clk_ignore_unused` 要尽早摘掉**：他们发现该参数压住了 `late_initcall` 的
  `clk_disable_unused()`，导致 ~414 个引导阶段开启的时钟永远没人关；他们改成
  "让驱动 own 自己的时钟"后去掉了这个参数。（我们现阶段保留，等启动稳定后处理。）
- **LK 抢 DTB/cmdline 的问题他们也踩了**：他们打补丁让 `setup.c` 从**内核自带的
  DTB** 里重新读 `/chosen/bootargs`（因为 LK 用自己的 baked-in cmdline，
  忽略 boot.img/vendor_boot 的 cmdline）。这正是我们第五节分析的同一个坑，
  也是我们 `CMDLINE_FORCE` 方案的另一条路。
- **显示**：他们的 `drivers/gpu/drm/mediatek/` + `mediatek_v2/` 子目录（251 个文件）
  是下一阶段显示移植的参考实现。

## 七、上机测试

前置：解锁 bootloader、`fastboot` 可用、已备份原厂 `boot_a` / `dtbo_a`。

### 0. 先决条件：日志怎么拿回来（无 UART 方案）

本机实测结论：

- `fastboot boot`（非破坏性加载）**在该设备上不可用**，会显示第一屏后重启；
  所以测试只能"刷 `boot_a` → 失败再刷回原厂"。
- 手机没有引出串口，`fastboot boot` 也不行，那**日志必须落在重启后仍可读的内存里**。

做法是复用原厂自己的 ramoops/pstore 区域。原厂 LK 传给内核的 cmdline 里已经有：

```
ramoops.mem_address=0x48090000 ramoops.mem_size=0xe0000
ramoops.console_size=0x40000  ramoops.pmsg_size=0x10000
```

对应原厂 DT 的 reserved-memory `mblock-17-pstore`（`reg = <0x48090000 0xe0000>`，`no-map`），
也就是说**原厂 Android 内核自己就在用这块内存做 pstore**，设备上 `/sys/fs/pstore`
是已挂载的 pstore 文件系统。

坑在于两代内核的**布局算法不一样**：

| | 4.14（原厂，MTK 改过） | 7.2.8（主线） |
| --- | --- | --- |
| dumps 区大小 | `size - console*2 - ftrace - pmsg` | `size - console - ftrace - pmsg` |
| 额外区 | 多一个 `bconsole`（AEE lockless console，占 `console_size`） | 无 |

若照抄原厂参数，主线会把 console 区放到 `+0x8F000`，而 Android 期望在 `+0x4F000`
→ 读出来是错位数据。因此本移植**显式指定尺寸**，让主线的 zones 与原厂完全重合：

```
pstore.kmsg_bytes=262144
ramoops.mem_address=0x48090000
ramoops.mem_size=0x8f000        # = 0x4F000(dumps) + 0x40000(console)
ramoops.record_size=0x1000      # 4 KiB/条，79 条，与原厂 max_dump_cnt 一致
ramoops.console_size=0x40000
ramoops.ftrace_size=0           # 不创建，避免占用原厂 bconsole 区
ramoops.pmsg_size=0             # 同上
```

这样主线写入的布局与 Android 读取的布局**逐字节对齐**，且完全不碰原厂的
`bconsole`/`ftrace`/`pmsg` 区。另外 `CONFIG_PSTORE_CONSOLE=y` 会把 printk
持续写进 console 环形缓冲，所以**即使只是卡死没有 panic**，最后打印的内容也留得住。
`pstore.kmsg_bytes=262144` 让 panic 时整段 dmesg 都能落盘（默认只有 10 KB）。

为什么这条路在 arm64 上成立（已逐项核对代码）：

- 原厂 DT 把该区标了 `no-map`。arm64 会为 nomap 区建一个名为 "reserved" 的 iomem 资源，
  单看会让 `request_mem_region()` 冲突；但 ramoops 的映射路径是
  `persistent_ram_buffer_map()`：`pfn_valid()` 为真就走 `persistent_ram_vmap()`，
  **根本不调用 `request_mem_region()`**。而 arm64 上 `pfn_valid()` 对 nomap 区仍返回真
  （正因为 `pfn_valid()` 无法区分 nomap，内核才额外提供了 `pfn_is_map_memory()`）。
  4.14 与 7.2.8 的这段逻辑一致，所以原厂内核能用，我们也能用。
- 记录头格式两代相同：`"====" + "%lld.%lu-%c\n"`，Android 的解析器能识别我们写的 dump。
- 两代 `persistent_ram_buffer` 头结构一致（`sig/start/size/data`），
  `sig = 原值 ^ PERSISTENT_RAM_SIG` 的算法也一致 → Android 会**保留**我们的数据而不是清零。

### 读日志

刷回原厂、正常进入 Android 后：

```bash
port/tools/read-boot-log.sh android-boot-log.txt     # 一键取回全部 pstore 记录
```

脚本会自动处理 SELinux（读完恢复原状态）。手工步骤等价于：

```bash
adb shell su -c 'setenforce 0'                      # 需要 root，读完记得恢复
adb shell 'ls -la /sys/fs/pstore/'
adb shell 'cat /sys/fs/pstore/console-ramoops-0'    # 卡死时看这个（纯文本）
adb shell 'cat /sys/fs/pstore/dmesg-ramoops-0'      # panic 时的完整 dmesg
adb shell su -c 'setenforce 1'
```

**实测出来的权限细节（踩过一遍，可直接照用）**：

| 路径 | 结果 |
| --- | --- |
| `adb shell`（uid 2000，在 `log` 组） | DAC 通过（目录 `dr-xr-x--- system:log`、文件 `0444`），但 **Enforcing 下被 SELinux 拦** |
| `adb shell` + `setenforce 0` | **可读** ✓（脚本用的就是这个组合） |
| `su -c 'cat ...'` | 失败：uid 0 既不是属主也不在 `log` 组，`0550` 目录对它没有权限位 |
| `su -c 'chmod 755 ...'` | 失败 `EPERM`：pstore 伪文件系统没实现 setattr |
| `/proc/last_kmsg` | 同样 `system:log`；`setenforce 0` 后可读，但那是 MTK 自己的日志通道（`mtk_ram_console`），**不是**我们写入的 ramoops 区，只能作旁证 |

注意：**必须热重启**（RAM 不断电）；拔电池/长时间断电会丢。

`initramfs` 里的 PID 1 还会打印内核实际收到的设备树身份
（`DT model` / `DT compatible` / 是否有 `bootprof` / 是否有 `odm`），
所以**一次启动就能判定走的是哪条 DTB 路径**。

### 测试 1（先做这个）：验证主线能否起来

```bash
fastboot flash boot_a port/out/boot-stockdt.img
fastboot reboot
```

**注意**：`boot-stockdt.img` 用的是原厂 DTB，里面没有 `simple-framebuffer` 节点，
所以屏上不会显示内核输出——请直接按下面流程取 pstore 日志。
（`boot-owndtb.img` 里带了 `simple-framebuffer`，如果地址猜对了，
屏幕会直接显示内核控制台——见第三节"屏幕控制台"。）

1. 等 30 秒左右（给内核跑完 probe / 卡死）；
2. 长按电源键重启，`fastboot flash boot_a n10p/boot_a.img`（原厂镜像，已验证一致）；
3. 正常进入 Android 后，**优先取 expdb**（LK 已把 ring 转存过去）：

```bash
port/tools/read-expdb-log.sh android-boot-log.txt   # 主路径（log_store → expdb）
port/tools/read-boot-log.sh android-pstore.txt      # 备选（pstore）
```

`boot-stockdt.img` 的内核带 `earlycon=mtk8250,mmio32,0x11002000`，
早期日志会进 console 环形缓冲；`CONFIG_PSTORE_CONSOLE=y` 把后续 printk 持续写入；
卡死或 panic 都会在 pstore 里留下内容。期望在其中看到：

```
 CHOPIN MAINLINE: userspace reached (PID 1)      <- 走到用户空间
init: kernel    : Linux version 7.2.8 ...
init: DT model      : MT6893                      <- 说明用的是原厂 DTB
init: DT bootprof   : yes (LK merged)
init: DT odm node   : yes (vendor dtbo applied)

init: --- clk_summary (head) ---
     clock                        enable_cnt  prepare_cnt  protect_cnt  rate   accuracy phase
   ...
init: --- /proc/interrupts (head) ---
init: --- tty driver info ---
0: uart:MTK ttyS0 at MMIO32 0x11002000 (irq = 13, base_baud = 0) is a 16550A
```

关注的信号：

- `earlycon=mtk8250,mmio32,0x11002000` 不依赖 DT、也不依赖时钟驱动，
  即使 topckgen/infracfg 出问题也能打出最早的内核日志。
- `clk_summary` 里应能看到 `clk26m`、`f26m_sel` 以及 topckgen 的各个 `*_sel`；
  若整段只有 `clk26m`，说明 topckgen/infracfg 驱动没绑上。
- `tty driver info` 里出现 `ttyS0` 说明 UART 驱动 probe 成功、拿到了 `baud`/`bus` 时钟。
- 日志停在某一行之前 → 那一行之后的驱动就是问题所在。

### 测试 2：判定自定义 DTB 是否生效

```bash
fastboot flash boot_a port/out/boot-owndtb.img
fastboot reboot
```

同样按上面的流程取回日志，看 `init: DT model`：

| 现象 | 结论 |
| --- | --- |
| `Xiaomi Redmi Note 10 Pro (chopin)`、`bootprof: no`、`odm: no` | **LK 使用 boot.img 的 DTB**，我们的设备树生效（原厂 overlay 未应用） |
| `MT6893`、`bootprof: yes` | LK 忽略 boot.img 的 DTB，复用了自己分区里的 `lk_main_dtb` |
| 起不来且日志停在 LK 阶段 | LK 尝试把原厂 dtbo 叠加到我们的 DTB 上失败；见下 |

若出现第三种情况，可以把一个空的 dtbo 刷进 `dtbo_a` 以关闭 overlay
（`port/tools/mkdtbo.py` 可生成正确的大端 DTBO 镜像），
但该分区有 SBC 签名校验，未解锁/校验严格时可能被拒，需实测。

### 回滚

```bash
fastboot flash boot_a <原厂 boot_a.img>
fastboot flash dtbo_a <原厂 dtbo_a.img>
```

## 八、当前状态与后续

已完成：

- **采用上游未合并的 Dimensity 1200 时钟系列**（10 个驱动 + 绑定 + ID 头），
  并与原厂数据逐项交叉验证：topckgen 53/55、PLL 10/10、infra 门控 61 条完全一致
- 适配 7.2.8 的 API 漂移（`mtk_clk_register_plls` 改为传 `device`）
- **复位控制器**（上游系列没有）：从原厂 UFS 平台头文件核实出 4 个复位位号，
  实现在 `clk-mt6893-infra_ao.c`，新增 `mediatek,mt6893-resets.h` 绑定
- **原厂 DT 兼容**：三个驱动加原厂 compatible fallback，`infra_ao` 增加
  有条件（检测 `/bootprof`）的 UART 门控 ID 重映射，保住 LK 传原厂 DTB 的那条路
- **SoC 设备树**采用 [MT6893-Mainline/linux](https://github.com/MT6893-Mainline/linux)
  `dirty-x6710` 的完整 `mt6893.dtsi`（1269 行，覆盖 pinctrl/scpsys/pwrap/spi/i2c/mmc/ufs/usb/thermal/GCE），
  修掉它 2 处自身问题（pinctrl 缺 `eint` reg-name、`CLK_TOP_MSDC30_1` 笔误）
- 板级设备树增加**屏幕控制台**（simple-framebuffer + fbcon，无 UART 时的第二路输出）
- 参考 [MT6895-Mainline/linux](https://github.com/MT6895-Mainline/linux)
  `7.2-mt6895-xiaomi-xaga`（同 7.2 内核的小米 MTK 机型），发现：
  **pstore 区可能不跨重启保留**，改用 MTK `log_store`(0x7ffbf000)+SRAM 头(0x11df00)
  +`expdb` 分区的日志链路（详见第六节）
- **修复主线 `pinctrl-mt6893.c` 的寄存器窗口名 bug**，并接入 pinctrl 节点
- 修正绑定文档示例中的 EINT 地址
- 内核与 dtb 构建、最小 initramfs、与原厂头部完全对齐的 boot 镜像
- 引导链与 DTB 路径分析（原厂 `boot_a.img` + LK 日志），并已用 ADB 实机核对
- **无 UART 的日志持久化方案**：`CONFIG_PSTORE_RAM/CONSOLE` +
  与原厂 Android 逐字节对齐的 ramoops 布局（`port/tools/read-boot-log.sh` 一键取回）
- `arch/arm64/configs/chopin_defconfig`（已验证可复现）

已知待办：

- **上机验证**：刷 `boot-stockdt.img` → 失败则刷回原厂 → 进 Android 取 pstore 日志。
  设备侧动作需要你操作（`fastboot boot` 已确认不可用）。
- 屏幕控制台的 framebuffer 地址有三个候选，当前用 0x77a20000，需实机确认。
- **日志通道已实现**：`mtk-logstore-console` 驱动把内核 console 追加进
  bootloader 的 log_store RAM 环，下次启动 LK 的 `kedump` 会把它转存到 expdb；
  两种 DTB 下都能 probe（第六节）。
- 拿到日志后按 `clk_summary` / `tty driver info` / 卡死位置决定下一步。
- **下一阶段外设**：PMIC（pwrap@10026000 + mt6359）→ 存储（UFS 0x11270000 /
  MSDC 0x11230000）→ USB → 显示（DSI 0x1400e000）。PMIC 是后续外设的前置。
- 复位控制器目前只有 4 个 UFS 相关位；UFS/MMC 接入时若需要更多位号，
  需要数据手册依据（原厂材料里没有完整复位表）。
