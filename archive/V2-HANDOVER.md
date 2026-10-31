# 红米 Note 10 Pro (chopin / MT6893) Linux 主线移植 — v2 (vendor DRM) 交付文档

> **当前版本 `chopinport=v2ninetyfifth`**（内核 `git/chopin-linux-7.2.8`，port `port`，两边 `master` 一致）。
> 本文档面向"新会话接手"。**正文是按接手顺序写的**；所有历史记录（编年史与早期快照）完整保留在
> **附录 A / 附录 B**，一个字都没有删 —— 那些失败与误判的记录本身就是最有用的部分。

---

## 接手先读（一页）

### 现状一句话

**这块面板已经被完整驱动，而且休眠省电也通了**：供电（DVDD/AVDD/AVEE）、背光亮度、
120 Hz 刷新率、屏幕控制台、s2idle 挂起与恢复全部在设备上实测通过，且能从提交源码复现。

### 能力表（每项都有实测证据，不是"应该能用"）

| 能力 | 状态 | 判据 |
|---|---|---|
| 面板点亮 / DRM 驱动 / **屏幕上的控制台** | ✅ | `fbcon → mediatekdrmfb`；`/proc/fb` → `mediatekdrmfb` |
| **120 Hz 刷新率** | ✅ | `crtc` 的 `mode: "1080x2400": 120 ...`；`irqwatch` 的 mutex 速率 484/2s |
| **DVDD 供电由内核掌控** | ✅ | `wl2866d-dvdd2 use=1 open=1`，消费者 `1400e000.dsi.0-dvdd` |
| **AVDD / AVEE 供电** | ✅ | `lm36273_bias_enable enable = 1` |
| **亮度用户可调** | ✅ | `/sys/class/backlight/lm36273/brightness` 0..2047，写入后屏幕实际变化 |
| **亮度跨开关 / 跨挂起保持** | ✅ | 关前 300 → 恢复后仍是 `bl level = 300`（不是默认 1023） |
| **s2idle 挂起 / 恢复** | ✅ | 串口断开又回来 + `/proc/uptime` 连续（未重启） |
| **挂起时真正切断供电与背光** | ✅ | 挂起段 `wl2866d-dvdd2: off` + `bias_enable = 0` + `close`；恢复段全部回来 |

### 构建 / 刷机 / 校验循环（照抄）

```bash
# 1) 改内核源码后构建（构建树 linux-7.2.8/，仓库工作树 git/chopin-linux-7.2.8/，两者靠 cp 同步）
cd linux-7.2.8
make O=/home/kily/project/mt6893/build/chopin ARCH=arm64 \
     CROSS_COMPILE=aarch64-linux-gnu- chopin_defconfig
make O=/home/kily/project/mt6893/build/chopin ARCH=arm64 \
     CROSS_COMPILE=aarch64-linux-gnu- -j16 Image

# 2) 打 tag（版本号写在 cmdline 里，用于"设备在跑哪一版"）+ 打包 boot.img
#    build-images.sh 会自己重建 initramfs；编译失败它就停（不再打包过期 ramdisk）
sed -i 's/chopinport=v2XXX/chopinport=v2YYY/' \
    linux-7.2.8/arch/arm64/configs/chopin_defconfig
port/tools/build-images.sh

# 3) 交付镜像 + 哈希 + 同步仓库 + 校验
cp port/out/boot-stockdt.img port/dist/chopin-v2YYY-stockdt.img
sha256sum port/dist/chopin-v2YYY-stockdt.img | tee -a port/dist/SHA256SUMS
python3 port/tools/audit-repo.py --sync     # 复制缺失/过旧文件并刷新 manifest
port/tools/verify-drm.sh                    # 必须 all local checks passed

# 4) 提交（master 与 dev 都跟上，见"分支约定"）
```

**刷机后第一条命令永远是 `dmesg chopinport`** —— 确认设备在跑你要验证的那一版。
这一步省掉了整整两轮白工（详见 §0.30 一带的记录）。

### 最容易踩的坑（都是真踩过的）

| 坑 | 正确做法 |
|---|---|
| **用名字/字符串推断事实** | 先读实际内容。反复踩过：`mtk_smi_stub.c` 按名字当插桩删掉（其实必需）、按 `MI_` 前缀搜不到真实节点、grep 用被改名的旧标识符、正则写成 `MISSING FROM THE REPO`（工具实际打 `MISSING FROM REPO`） |
| **管道吞掉退出码** | 构建命令若接管道（`\| tail`、`\| grep`）必须 `set -o pipefail`，否则失败会被吞、过期产物被打包 |
| **initramfs 的工具是极简的** | 没有重定向；`dmesg` 只吃**一个词**；`cat` 曾把内容写进 kmsg 而非控制台；`grep` 是"子串 文件"。**给命令前先读它的实现** |
| **`master` vs `dev`** | `master` = 已烧机验证；`dev` 是暂存基线。**整理后两者应一致**；`curate-history.py` 负责按主题折叠 |
| **以为配置生效了** | `make chopin_defconfig` 会**静默丢弃**未定义符号并报成功；判据是 Image 里的字符串/实际行为，不是 `build=0` |
| **`sed`/`grep` 的锚点凭印象写** | 用**结构化定位**（找函数体、数括号）或先用 `cat -A` 看清真实空白 |

### 硬件与驱动地图

| 部件 | 位置 | 说明 |
|---|---|---|
| DSI | `dsi@1400e000` | vendor `mediatek_v2` 栈；`mtk_dsi.c` |
| 面板 | `panel@0`（`/dsi@1400e000/panel@0`） | 合成节点；`compatible = "l16_36_02_0b_dsc_vdo,lcm"`；`drivers/gpu/drm/panel/panel-k10a-bringup.c` |
| **DVDD 1.2 V** | WL2866D LDO1 @ **i2c7 `0x28`** | `willsemi,ldo@28/on,wl2866d1@1`；`drivers/regulator/wl2866d.c`；面板节点 `dvdd-supply` |
| **AVDD/AVEE + 背光** | LM36273 @ **i2c6 `0x11`** | 节点 `i2c_lcd_bias@11`，`compatible = "mediatek,i2c_lcd_bias"`；`drivers/gpu/drm/panel/leds-lm36273.c`；面板节点 `backlight` |
| 背光接口 | `/sys/class/backlight/lm36273/brightness` | 0..2047；`init` 的 `w` 命令可写 |
| 面板供电拓扑（原厂注释） | — | AVDD→VSP→LM36273 DISP_P；AVEE→VSN→LM36273 DISP_N；DVDD→WL2866D DVDD2 |

### 设备树的结构（v2ninetyseventh 起就是 MT6895 的形态）

```
arch/arm64/kernel/chopin-board.dts（3 行）
   └── #include "../boot/dts/mediatek/mt6893-xiaomi-chopin.dts"
          ├── /dts-v1/ + 8 个 dt-bindings 头文件
          ├── 15 个区段文件 mt6893-xiaomi-chopin-<子系统>.dtsi   ← 实机树的连续区段
          └── 14 个覆盖文件 mt6893-xiaomi-chopin-ovl-<主题>.dtsi ← 主线驱动需要的改动
```

**与 MT6895 的异同（重要）**：MT6895 的 `xaga-board.dts` 同样是 3 行 ✓、同样只 include 一个
板级文件 ✓ —— **形态完全一致** ✓。但它的板级源码是**手写的** ✓（1449 行、0 个 hex phandle ✓），
而 **chopin 的原始手写源码已丢失** ✗ ⇒ 我们这份是**实机树重新切分**的产物 ✓，
**形态相同、来源不同** ✓ —— 可读性/可导航性到手了，但没有"人写的"那层语义 ✓。

**为什么实机树不能换**：LK 运行时注入 ~14 段 mblock 保留内存（含调制解调器 `ap_md_c_smem` 87 MB、
`md_mem_usage` 57 MB ✓）。另一份候选 `mt6893-xiaomi-chopin.dts`（早期构建的反编译物 ✓，
后来被本次重构的顶层文件取代 ✓）**一个 mblock 都没有** ✗ ⇒ 换上它内核会把调制解调器内存当空闲用。
判据是硬的：`mblock` 条目数 **34 vs 0** ✓。

**为什么不能"在 `mt6893.dtsi` 上写板级源码"**：我们的 `mt6893.dtsi` 是联发科上游提交的**部分**
SoC 描述 ✓ —— 带 `@地址` 的节点只有 **73** 个 ✓，而实机树有 **478** 个 ✓、两者重合仅 **56** 个 ✗。
缺的那 422 个里既有板级器件、也有**上游 dtsi 根本没描述的 SoC 块** ⇒ 缺了它们设备会坏 ✗
（不只是 mblock 的问题）。**MT6895 能那样做是因为它有一份完整的 `mt6895.dtsi`** ✓
—— 这才是两者真正的结构性差异 ✓。

**验收**：整个重构以 **DTB 逐字节不变** 为条件 ✓ —— 最终
`e615daac0c2c53ad7352a43b6c0a48890bdc442fb20a87d7f89d6fe3e5168b0b` ✓，
而**每次验收都先删掉 DTB 产物再构建** ✓（否则会拿到假通过 ✓）。

**这次重构踩到的坑**（都记下来）：
① DT 的 `.dts` 目标不能用 `make arch/arm64/kernel/chopin-board.dtb` —— `dtc` 的隐式规则会把源解析到
`arch/arm64/boot/dts/` 下面 ✗，板树是随 `Image` 构建的 ✓；
② 基树**没有 `/dts-v1/;`**（那一行原本在 `chopin-board.dts` 里 ✓）、也**没有那 8 个 dt-bindings
头文件**（同样在 board 文件头部 ✓）⇒ 换成 3 行的 board 文件后，这两样都必须搬进板级源码 ✓，
否则报 `Unexpected 'KEY_POWER'` / `CLK_INFRA_AO_UFS` ✓；
③ **两次假通过** ✗ —— 都比对了哈希却**没有重建 DTB** ✓，是构建的**退出码**暴露的 ✓。
**验证产物，不是验证哈希。**

### 触控（Novatek NVT36672C，已烧机验证 ✓）

**硬件**：触摸控制器**就在面板自己的显示驱动 IC 里**（Novatek NT36672C ✓ TDDI）✓，
挂 **`spi0@1100a000` 的 `ts_novatek@0`**（CS0 ✓，9.6 MHz ✓，中断 = `mt-eint 19` → GIC 33 ✓）。
**DT 从来就是完整的** ✓ ⇒ **一行都不用改** ✓（`compatible = "novatek,NVT-ts-spi"` 与原厂驱动完全一致 ✓）。

**驱动选择**：主线 `novatek-nvt-ts.c` **用不上** ✗（它要 `novatek,nt36672a-ts` + 真正的 `interrupts` ✓）。
用的是原厂 `drivers/input/touchscreen/NVT36672c/` ✓ —— 它**自带 `spi_driver`** ✓，
**不需要** `mtk_tpd` 框架 ✓。与 vendor 树的耦合**只有 4 个显示侧 ESD 符号** ✓，已全部移除 ✓。

**v4.14 → 主线 6.x 的适配**（都在提交里 ✓）：`of_gpio.h` → `devm_fwnode_gpiod_get_index()` ✓
（`con_id` 要传 `novatek,irq` ✗ 不能传完整属性名 ✓ —— `of_find_gpio()` 会往后拼 `-gpio` ✓）；
`SPI_MASTER_HALF_DUPLEX` → `SPI_CONTROLLER_HALF_DUPLEX` ✓；`.remove` 返回 `void` ✓；
`proc_create` → `proc_ops` ✓；`mtk_chip_config` 的成员变化 ✓；
`earlysuspend` 的注册本来就在 `CONFIG_HAS_EARLYSUSPEND` 里 ✓ 只需删 include ✓。
`nt36xxx_ext_proc.c` / `nt36xxx_mp_ctrlram.c`（调试与产测接口 ✓）**不移植** ✓，
由 `nt36xxx_stubs.c` 提供那 7 个符号 ✓。

**固件必须内置** ✗：Kconfig 自己写着这是 **"no flash"** 的芯片 ✓ ⇒ **每次开机都要主机下载固件** ✓
（每次 resume 也会重下 ✓，约 98 ms ✓）。原厂**只提供 `.ihex`** ✗ ⇒
`port/tools/ihex2bin.py` 转换 ✓（注意要处理 **`02` 扩展段地址**记录 ✓），
再用 `CONFIG_EXTRA_FIRMWARE` 编进内核 ✓（5 个候选名共 680 KB ✓）。

**两个把我卡了很久的坑**：

1. **CS 时序是我自己在移植时删掉的** ✗ —— 原厂 `spi_ctrdata` 里
   `cs_setuptime = 25, cs_holdtime = 25` ✓，而主线 `mtk_chip_config` **没有这些字段** ✓；
   我当时认定"主线没有这个能力"✗就没管 ✓ —— **其实主线有等价机制** ✓，只是换了形式：
   **`spi-cs-setup-delay-ns` / `spi-cs-hold-delay-ns`** ✓。删掉后
   `mtk_spi_set_hw_cs_timing()` **两个值都是 0 时什么都不做** ✗ ⇒ CS 几乎没有建立/保持时间 ✓
   ⇒ **Novatek 每次寄存器读前面那个 3 字节 page 写锁不住** ✓ ⇒ 读回来的是**无关寄存器** ✓。
   症状极有迷惑性：**响应格式完全规范**（`FF FF` + 6 字节 ✓，正好是驱动 `+2` 位移期望的两个
   turnaround 字节 ✓），**内容却和触摸 MCU 毫无关系** ✓。恢复后立刻识别成功 ✓。
2. **终端宽度吃掉关键字节** ✗ —— 原厂打印把 6 个 ID 字节挤在一行 ✓，而 **ID 表匹配的正是最后两个**
   （所有条目都是 `0x?? 0x?? 0x03` ✓）✓ ⇒ 前几轮我根本看不到判据 ✓。拆成两行后一次命中 ✓。

**验证手段（零代码 ✓）**：`grep NVT-ts /proc/interrupts` 读数两次 ✓，中间用手触摸 ✓ ——
**计数从 4 涨到 868** ✓✓。这比任何日志推断都硬 ✓。

### ⚠️ `=m` 的驱动全部不生效（下一个模块移植前必读）

只构建 `Image`（**0 个 `.ko`**），initramfs 也**不加载任何模块** ⇒
**defconfig 里 995 个 `=m` 的项全部等于关闭** ✗。要让一个驱动生效，必须：

1. 把它改成 `=y` ✓（或让构建产出模块并在 initramfs 里加载 ✓ —— 目前两者都没有）；
2. **DT 节点的 compatible 与驱动匹配** ✓ —— 这条独立于上一条，而且很容易忽略：
   触摸的节点写的是 `compatible = "mediatek,touch"`（原厂包装层 ✓），而树里的 goodix 驱动要的是
   `goodix,gt9916` / `gt1151` 之类 ✗ ⇒ **即使把它改成 `=y` 也绑不上** ✓。

**已移植并生效的子系统**（全部 `=y`）：显示（`DRM_MEDIATEK_V2` + 面板）、面板供电
（`REGULATOR_WL2866D` + LM36273）、USB（`USB_MTU3`/`USB_XHCI_MTK`）、UFS、i2c/spi、
按键、RTC、PMIC ADC、efuse ✓。

**未移植**（驱动未启用 ✓，都符合预期）：触摸、WiFi/BT、音频、GPU、充电/电量、
thermal、视频编解码、相机、NFC ✓。

### 启动失败时的诊断通道（一直都有 ✓ —— 见下面的更正）

`CONFIG_CMDLINE` 里**从很早就有** `ramoops.mem_address=0x48090000` ✓
（`mem_size=0x8f000`、`record_size=0x1000`、`console_size=0x40000` ✓），
配合 `CONFIG_PSTORE=y` + `CONFIG_PSTORE_RAM=y` + `CONFIG_PSTORE_CONSOLE=y` ✓ ⇒
**任何一次"起不来"的启动，控制台日志都会落在 pstore 里** ✓。

**取证步骤**（顺序重要 ✓，pstore 落在 RAM 里 ✗ —— 断电会丢 ✓）：

```bash
# 失败后【直接进 fastboot，不要断电】
fastboot flash boot_a <任何一个能正常启动的镜像> && fastboot reboot
chopin# ls /sys/fs/pstore/
chopin# cat /sys/fs/pstore/console-ramoops-0
```

**⚠️ 更正**：触控移植期间我曾在提交 `e942662` 的信息里写"我们的 cmdline 没有让它启用"✗ ——
**那是错的** ✗，而且**没有查证就写下了** ✓。事实上 `ramoops.*` 在触控之前就在 cmdline 里 ✓，
那次改动**只改了版本号** ✓。因此当时那张"触控关闭 + ramoops 打开"的诊断镜像
（`v2onehundredth` ✓）其实**不比普通镜像多任何能力** ✓ —— 而我在你遇到启动失败时
说"读不出日志"✗，**把一个本来可用的诊断通道说没了** ✗。

### 未做 / 已知问题

| 项 | 说明 |
|---|---|
| **`bias` GPIO** | 原厂注释写 `GPIO126/bias0`，编号未经核实，**故意没驱动**；i2c 侧的 `bias_enable(0,3)` 才是功能性开关 |
| **`deep` 挂起** | 未测。`mem_sleep` 默认 `[deep]`，而原厂 DT 明写 `suspend-method = "s2idle"` ⇒ s2idle 才是设计目标 |
| **挂起功耗未实测** | 只能从日志确认供电被切断；实际电流没法从控制台测 |
| **`KTZ8863A` 驱动仍在树里** | 编进去了但 DT 无匹配节点 ⇒ 从不 probe（它的 12 处调用已成空操作）|
| 已知无害的报错 | `mtk-mmsys ... -EBUSY`、`read ccorr_* failed`、`mi_disp ... debugfs`、i2c 相机/NFC `-16`、`mt6360-led of_node`、`DDP_COMPONENT_DSI0:buffer underrun`（模式切换时偶发）|

### 工具状态

| 工具 | 用途 |
|---|---|
| `port/tools/verify-drm.sh` | 全量本地校验（配置、DTB、面板序列、仓库一致性、镜像哈希）。**必须 all local checks passed** |
| `port/tools/audit-repo.py [--sync]` | 比对编译树与 git 仓库；`--sync` 复制缺失/过旧文件并先刷新 manifest。用**退出码**判断，别解析文本 |
| `port/tools/curate-history.py SPEC.json` | 按主题折叠提交（先校验区间无缝隙/无重叠，再用 tree hash 证明零丢失）|
| `port/tools/build-images.sh` | 打包 boot.img；**会自己重建 initramfs**，失败即停 |
| `port/tools/mkinitramfs.sh` | 单独重建 initramfs（`init.c` 编译失败会返回非零）|
| initramfs 内命令 | `drm`（含 wl2866d 供电计数器段）、`cat <文件>`、**`w <文件> <值>`**、`dmesg <一个词>`、`ls`、`setmode` |

### 镜像与回退

```bash
# 当前交付
port/dist/chopin-v2ninetyfifth-stockdt.img     # 已烧入验证（挂起省电）
# 已知良好
port/dist/chopin-v2ninetythird-stockdt.img     # suspend/resume 首次验证
port/dist/chopin-v2ninetysecond-stockdt.img    # 亮度可调
port/dist/chopin-v2eightyninth-stockdt.img     # P1 供电 + 背光
# 更早的见 port/dist/SHA256SUMS
```

```bash
# 历史备份 tag（整理前的原始提交）
git -C git/chopin-linux-7.2.8 tag -l '*pre-curate*'
git -C port tag -l '*pre-curate*'
# 回退：
#   git -C <repo> reset --hard <tag>
```

---

# 附录 A：编年史（§0.1 ~ §0.35，按编号排序）

> **内容一字未改**，只是按编号排好。这是本移植的真实过程记录 —— 包括所有失败、
> 误判与后来的纠正。**接手时值得通读**，尤其注意反复出现的那几类错误：
> 用名字/字符串推断事实、用管道判断成败、没先读工具实现就给命令、
> 没确认设备在跑哪一版就分析日志。这些错误每一次都真实地浪费过一轮或更多。

## 0. v2fiftyfirst 更新（接手先读这一节）

### 0.1 面板 probe `-12` 的根因已经找到并修掉了

**它不是 of_graph、不是 mipi-dsi 封装 —— 是插桩时漏掉的花括号。**
`panel-k10a-bringup.c` 的 `lcm_probe()` 里：

```c
ctx = devm_kzalloc(dev, sizeof(struct lcm), GFP_KERNEL);
if (!ctx)
	dev_err(...);          /* 只守住这一行 */
	return -ENOMEM;        /* ← 无条件执行，probe 必定 -12 */
```

这完美解释了旧文档 §3.2 的全部三条"已排除"证据：`return at line N` 插桩打不出来，是因为
它们被 `!ctx` 守住而 `ctx` 其实分配成功了；`ctx=… sizeof(lcm)=…` 打不出来，是因为它位于那条
无条件 `return` 之后，是**死代码**。同型问题在 `drm_panel_of_backlight()` 之后还有一处
（`return ret;` 无条件）。


### 0.2 修完之后会立刻撞上的下一个失败（也已修）

编译出来的 DT（`build/chopin/arch/arm64/kernel/chopin-board.dtb`，实测反编译）里
`panel@0` 只有：

```
compatible = "l16_36_02_0b_dsc_vdo,lcm";
reg = <0x00>;
reset-gpios = <0x02 0x2a 0x00>;   /* /pinctrl 有 gpio-controller + #gpio-cells，能解析 ✓ */
status = "okay";
```

**没有** `leden-gpios` / `dvdd-gpios` / `cam-gpios` / `vrf18` supply / `backlight`。
所以原来的 `devm_gpiod_get(dev, "leden", …)` 会让 probe 以 `-2 (-ENOENT)` 失败。
现已把这几个都改成 `*_optional()`，并让 `prepare`/`unprepare` 遇到缺失描述符就跳过。

另外顺手修了一个潜在 oops：`vrf18` supply 不存在时 `regulator_get()` 返回 `ERR_PTR`，而
`lcm_prepare()` 会**无条件**调用 `lcm_panel_vddi_enable()` → `regulator_set_voltage(ERR_PTR,…)`
解引用崩溃（本板 `CONFIG_REGULATOR_DUMMY` 未开，走的正是 `ERR_PTR(-ENODEV)` 分支）。


### 0.3 新发现：DSC 编码器根本没进管线（**下一个必须解决的坑**）

`chopin-board.dts` 里 `disp_dsc_wrap@14124000` 用的是 `compatible = "mediatek,mt6893-disp-dsc"`，
但它**匹配不到任何东西**：

| 位置 | 是否有 mt6893 |
|---|---|
| v2 `mtk_disp_dsc.c` 的 `of_match` | ❌ 只有 mt6885/6983/6895/6873/6853/6879/6855 |
| v2 `mtk_drm_drv.c` 组件匹配表 | ❌ 同上，无 6893 |
| `mt6885_mtk_ddp_main[]` 主通路 | ❌ 根本没有 `DDP_COMPONENT_DSC0`（只有 `mt6885_dual_data_ext[]` 里有） |

⇒ k10a 是 **DSC 压缩面板**，只换参数而不把 DSC 接进主通路，"点亮"这一步大概率仍然不出图。
这也和 §2 里"bound 的 12 个组件"不含 `0x14124000` 相吻合。


### 0.4 当前 bringup 驱动仍是 l16 参数（本版**故意**没换）

| 项目 | 现状（本版仍是 l16） | k10a 真值（厂商驱动） |
|---|---|---|
| 分辨率 | 1080×**2460** | 1080×**2400** |
| `DATA_RATE` | 1100 | 834 |
| `DSC_XMIT_DELAY` | 512 | 170 |
| `DSC_INCREMENT_INTERVAL` | 488 | 113 |
| `DSC_FINAL_OFFSET` | 4336 | 7072 |

**本版目标只是"确认面板注册成功 + connector 出现模式"**，所以刻意保留 l16 参数，
一次只验证一个变量。k10a 真参数留给下一张。


### 0.5 v2fiftysecond —— v2fiftyfirst 在真机上暴露的两个问题（**已修**）

v2fiftyfirst 已刷机实测。**好消息：`-12` 修好了**：

```
CHOPIN-PANELDBG: probe enter
CHOPIN-PANELDBG: ctx=ffff0000c1d5fc80 sizeof(lcm)=280
CHOPIN-PANELDBG: drm_panel_add done        ← 面板真的注册了
```

但同一份日志暴露了两个新问题，都在 v2fiftysecond 修掉：

**问题 1：`drm` 命令卡死（系统 hang）。**
面板一旦注册，`lcm_probe()` 设的 `output_mode = MTK_PANEL_DSC_SINGLE_PORT` 就生效，
`mtk_drm_crtc.c` 里所有 DSC 分支开始执行，而 `priv->ddp_comp[DDP_COMPONENT_DSC0]` 是 **NULL** ——
因为板级树把 DSC 节点的 compatible 改成了 `mediatek,mt6893-disp-dsc`，这个字符串
**谁都匹配不上**（v2 `mtk_disp_dsc.c` 的 of_match 与 `mtk_drm_drv.c` 的组件表都没有它）。
于是 `dsc_comp->mtk_crtc = mtk_crtc;`（connect 路径）和 dump 路径直接空指针解引用。
`drm` 工具正好会 dump CRTC 寄存器 → 卡死。

**修法（比 §0.3 当初设想的简单得多）**：live base 树本来就有**正确**的这个节点 ——
`compatible = "mediatek,disp_dsc_wrap", "mediatek,mt6885-disp-dsc"`，而且带
`interrupts = <GIC_SPI 281 ...>`（`mtk_disp_dsc_probe()` 里 `platform_get_irq()` 是**必需**的）。
我们只需要把 compatible 改回 `mediatek,mt6885-disp-dsc`，**不需要**动任何驱动表，
也**不需要**往 `mt6885_mtk_ddp_main[]` 里插 `DDP_COMPONENT_DSC0` ——
`mtk_drm_crtc.c` 是按 `mmsys_id == MMSYS_MT6885` **运行时动态插入** DSC 的，
而本板 dispsys 节点第一个 compatible 就是 `mediatek,mt6885-mmsys`。只保留我们的时钟下标
（`CLK_MM_DISP_DSC_WRAP = 44`；base 树的裸下标 43 是厂商 binding 的编号）。

另外给 `mtk_drm_crtc.c` 里 5 处无条件解引用加了判空（dump / mode-switch / connect / disconnect），
这样万一 DSC 绑定失败也只是打一行日志跳过，而不是把系统带下去。

**问题 2：connector 只有 generic modes（1024x768/800x600/848x480/640x480），没有面板模式。**
`mtk_dsi_probe()` 跑在面板注册之前，把 `dsi->panel` 置成 NULL 就继续了（§7.11 那个 hack），
而 `mtk_dsi_connector_get_modes()` 只是 `drm_panel_get_modes(dsi->panel, ...)`——
**没有任何地方再去解析面板**，所以 connector 一直用 DRM 的兜底模式表。

**修法**：在 `mtk_dsi.c` 加 `mtk_dsi_resolve_panel()`，让 `mtk_dsi_connector_get_modes()`
在 `dsi->panel` 为空时按需重新解析；再让面板驱动在 `mtk_panel_ext_create()` 成功后调用
`drm_helper_hpd_irq_event(drm_dev)` 踢一次 connector 重探。（`drm_dev` 是 v2 栈的全局变量，
在 `mtk_drm_bind()` 里经 `disp_dbg_init()` 赋值，而面板 probe 在 bind 之后。）

预期刷 v2fiftysecond 后：`drm` 不再卡死；connector 出现 **1080×2460**（仍是 l16 参数）；
`dmesg CHOPIN` 里出现 `mediatek-disp-dsc 14124000.disp_dsc_wrap: ...` 绑定日志，
`bound` 列表多出 `14124000.disp_dsc_wrap`。


### 0.6 v2fiftysecond 实测 + v2fiftythird 修复（**当前版本**）

v2fiftysecond 实测结论：

- ✅ **DSC 绑定成功了** —— `CHOPIN[kms_init_done] DSC CON=0x…` 那几行能打出来，
  说明 `priv->ddp_comp[DDP_COMPONENT_DSC0]` 非空（§0.5 那个 compatible 修复奏效）。
- ✅ 面板仍然注册成功（`drm_panel_add done`）。
- ❌ connector 还是只有 generic modes（1024x768/800x600/848x480/640x480）。
- ❌ `drm` 还是卡在 `/proc/fb`。
- 关键线索：`hpd kick done (drm_dev=0000000000000000)` —— **`drm_dev` 是 NULL**，
  因为面板是在 `mtk_dsi_probe()` 内部、`mipi_dsi_host_register()` 里**同步探到**的，
  那时 DRM 还没 bind（`CHOPIN-STAGE kms_init` 在 11.93，面板 probe 在 11.88）。
  §0.5 里"面板 probe 在 bind 之后"的假设**是错的**。

**两个症状其实是同一个根因**：DSI 节点上有**两个** port。

| | 是谁 | 指向 |
|---|---|---|
| `ports/port`（base 树，**排在前面**） | 厂商的 | `panel1@0`，compatible `k10a_36_02_0a_dsc_vdo,lcm`，**disabled 且本内核没有驱动** |
| `ports/port@1`（我们加的） | 我们的 | `panel@0`，compatible `l16_36_02_0b_dsc_vdo,lcm` ← 真正在跑的那个 |

`mtk_dsi_probe()` 与 `mtk_dsi_resolve_panel()` 都用 `of_graph_get_next_endpoint()`，
即**取第一个 endpoint** —— 也就是 base 的 port 0。板级树注释里写的
"mtk_dsi 会先找 port 1"**是错的，代码里没有这种优先级**。于是：

1. `of_drm_find_panel(panel1@0)` → `-ENODEV`（该节点 disabled）⇒ `dsi->panel` 与
   `dsi->ext` **一直是 NULL**（日志里 `CHOPIN-DBG[dsi_probe] panel=0 ext=0` 就是它）；
2. connector 只能拿 DRM 兜底的 generic modes；
3. 第一次 modeset 走到 `mtk_dsi_ps_control_vact()`：
   ```c
   struct mtk_panel_ext *ext = mtk_dsi_get_panel_ext(&dsi->ddp_comp);
   struct mtk_panel_dsc_params *dsc_params = &ext->params->dsc_params;  /* ext == NULL */
   ```
   **空指针解引用** ⇒ 机器卡死（`panic=0` 不重启）⇒ `drm` 工具读 `/proc/fb` 时阻塞。
   所以"卡死"不是 `drm` 工具的问题，是内核已经躺了；`hpd kick` 也永远不会触发。

**v2fiftythird 的修法**：

- `chopin-board.dts`：在 DSI 的 `ports` 里 `/delete-node/ port;`，只留 `port@1`，
  这样第一个（也是唯一一个）endpoint 就指向我们的 `panel@0`。
  已在编译产物里核对：`ports` 下只剩 `port@1`，其 endpoint 的 remote-endpoint
  正是 `panel@0` 的 endpoint（phandle 互指）。
- `mtk_dsi.c`：给 `mtk_dsi_ps_control_vact()` 加 `ext` / `ext->params` 判空 ——
  即使面板没解析出来也只是打一行日志跳过，不会再躺。
- `panel-k10a-bringup.c`：`drm_dev` 为空时不再谎报 "hpd kick done"，
  而是说明"面板由 mtk_dsi_probe 自行解析"。

预期刷 v2fiftythird 后：`CHOPIN-DBG[dsi_probe] panel=ffff… ext=ffff…`（**非 0**）；
`drm` 不再卡死；connector 出现 **1080×2460**（仍是 l16 参数）；
不应再出现 `panel extension not ready` 或 `panel/bridge not up yet`。


### 0.7 v2fiftythird 实测 + v2fiftyfourth 修复（**当前版本**）

v2fiftythird 实测：`drm_panel_add done` 照旧，但 `CHOPIN-DBG[dsi_probe] panel=0 ext=0`
**仍然是 0**。也就是说：面板明明加进列表了，紧接着的 `of_drm_find_panel()` 却找不到。

**§0.6 的诊断只对了一半。** 真正的原因在 `dsi_find_slave()`（`mtk_dsi.c:4704`）：

```c
static struct device *dsi_find_slave(struct mtk_dsi *dsi)
{
	remote = of_graph_get_remote_node(dsi->dev->of_node, 1, 0);   /* ← port 1 */
	...
	pdev = of_find_device_by_node(remote);
	if (!pdev)
		return ERR_PTR(-EPROBE_DEFER);     /* ← 面板不是 platform device */
	...
}
```

**DSI 的两个 port 各有用途，而且分工是反的**：

| port | 谁在用 | 用途 |
|---|---|---|
| **port 0** | `mtk_dsi_probe()` 的 `of_graph_get_next_endpoint()` | **面板** |
| **port 1** | `dsi_find_slave()` 的 `of_graph_get_remote_node(node, 1, 0)` | **双 DSI 的 slave** |

所以把面板放到 `port@1`（v2fiftythird 干的事）会变成：

1. `dsi_find_slave()` 把**我们的面板**当成 slave，`of_find_device_by_node()` 在
   **platform 总线**上找它（面板在 mipi-dsi 总线上）⇒ 返回 NULL ⇒ `-EPROBE_DEFER`；
2. `mtk_dsi_host_attach()` 返回该错误 ⇒ `mipi_dsi_attach()` 失败；
3. `lcm_probe()` 的失败分支调用 `drm_panel_remove(&ctx->panel)` ——
   **把刚加进 `panel_list` 的面板又删掉了**；
4. 于是 `mtk_dsi_probe()` 随后 `of_drm_find_panel()` 查一个已经没有它的列表 ⇒ NULL。

这完美解释了"`drm_panel_add done` 打了、`panel=0` 却也是 0"这个矛盾。
板级树里那句"mtk_dsi 会先找 port 1"大概就是看到了 `dsi_find_slave` 里的 port 1 ——
但那是给 **slave** 用的，不是面板。

**v2fiftyfourth 的修法**：把**面板放回 port 0**，并**彻底删掉 port 1**：

```c
	ports {
		#address-cells = <1>;
		#size-cells = <0>;

		/delete-node/ port@1;          /* 单 DSI，没有 slave */

		port {
			dsi_out: endpoint {
				remote-endpoint = <&panel_in>;
			};
		};
	};
```

已在编译产物里核对：DSI 的 `ports` 下只剩 `port`，其 endpoint 与 `panel@0` 的 endpoint
phandle 互指；没有 port 1。

另外把 `lcm_probe()` 里 `mipi_dsi_attach()` 的失败从**静默**改成打印错误码 ——
正是这个静默让这个 bug 多花了两次刷机：

```c
	ret = mipi_dsi_attach(dsi);
	if (ret < 0) {
		dev_err(dev, "CHOPIN-PANELDBG: mipi_dsi_attach failed %d, removing the panel\n", ret);
		drm_panel_remove(&ctx->panel);
	}
```

预期刷 v2fiftyfourth 后：**不再出现** `mipi_dsi_attach failed`；
`CHOPIN-DBG[dsi_probe] panel=ffff… ext=ffff…` **非 0**；
`drm` 不再卡死；connector 出现 **1080×2460**。


### 0.8 v2fiftyfourth 实测：**通路打通** + v2fiftyfifth（k10a 真参数，当前版本）

v2fiftyfourth 实测结果 —— **面板通路彻底通了**：

- `dmesg CHOPIN-PANELDBG` 里**没有** `mipi_dsi_attach failed`（§0.7 的 port 0 修复奏效）；
- `drm` 的 connector 段出现 **`1080x2460` × 7** —— 也就是面板驱动自己报的
  30/48/50/60/90/120/144 Hz 七个模式，**不再是 DRM 的 generic 模式表**。
  这说明 `dsi->panel` 与 `dsi->ext` 都已非 0，连接器拿到了真面板模式。

至此：面板注册 → attach → 解析 → connector 模式，**四步全部打通**。

> 仍然存在的小问题：`drm` 这个自检命令读到 `/proc/fb` 时还是没有后续输出。
> 那是**自检工具**的问题（`/proc/fb` 是它第一个需要 DRM 锁的读取），
> 不是内核又躺了 —— 否则 connector 的 modes 也读不出来。后续可以让
> `port/initramfs/init.c` 的 `cmd_drm()` 把 lock-free 的
> `/sys/kernel/debug/mtk-disp-trace` 挪到 `/proc/fb` 之前。

**v2fiftyfifth = k10a 真参数**（本次）。把 bringup 驱动里残留的 l16 数值换成
厂商 k10a 驱动的真值：

| 项目 | 原来（l16） | 现在（k10a 厂商值） |
|---|---|---|
| `FRAME_HEIGHT` | 2460 | **2400** |
| `PHYSICAL_WIDTH/HEIGHT` | 67392 / 153504 µm | **70200 / 152100 µm** |
| `DATA_RATE` | 1100 | **834** |
| `DSC_XMIT_DELAY` | 512 | **170** |
| `DSC_INCREMENT_INTERVAL` | 488 | **113** |
| `DSC_FINAL_OFFSET` | 4336 | **7072** |

模式时序也换成厂商值：`HSA 10 / HBP 8 / VSA 10 / VBP 10`，
`HFP 484`(60Hz) / `174`(90Hz) / `20`(120Hz)，
低功耗 `VFP 549 / 673 / 2528`(50/48/30Hz)，
时钟 `316325 / 369170 / 422163 kHz`。

`mtk_disp_dsc.c` 是**从这些 `dsc_params` 现算 PPS** 的（它有
`CHOPIN-DSC-PARAMS ver=…` 插桩），所以这些值就是编码器真正会写进
`DSC_PPS0..19` 的东西 —— 也是与 `port/DSC-REFERENCE.md` 黄金值对账的落点。

**初始化序列刻意没动**：`lcm_probe()` 里 `ctx->prepared = true`，
所以 `lcm_prepare()` 会提前返回、**不会**推 l16 的初始化表，
面板保持 LK 留下的（正确的 DSC 模式）状态。

> 厂商 k10a 表里**没有 144 Hz**；`mode_144hz` 现在是占位（clock 398294，
> 与 120Hz 同 porches），只为不动 mode-index 那一套管线，永远不会被选为 preferred。


### 0.9 v2fiftyfifth 实测：面板与参数全对，但 modeset 卡住 —— `is_dual_pipe=1`（v2fiftysixth 修复）

v2fiftyfifth 实测：

- ✅ `CHOPIN-DBG[dsi_probe] panel=ffff0000c1c79a88 ext=ffff0000c1c21800` —— **都非 0**；
- ✅ connector 变成 **`1080x2400` × 7** —— k10a 真参数生效；
- ❌ `drm` 仍然卡在 `/proc/fb`。

**新线索**：`CHOPIN-STAGE crtc_enable: is_dual_pipe=1`（v2fiftythird 时是 0）。

为什么变了？`mtk_crtc_is_dual_pipe()` 的条件里有 `panel_ext`：
面板没解析出来时 `panel_ext == NULL`，条件不成立 ⇒ `is_dual_pipe=0`；
面板一解析出来，条件就成立了 ⇒ `is_dual_pipe=1`。**这不是参数改动引起的，是面板修好后才暴露的。**

而 `is_dual_pipe=1` 会让 CRTC 走**第二条 pipe**（OVL1/RDMA1/COLOR1/CCORR1/AAL1/
GAMMA1/POSTMASK1/DITHER1）—— 这些正是本树**刻意 disable 掉的 MT6895-only 节点**，
本板一个都没有。于是第一次 modeset 走进未绑定的 DDP 组件，卡死。

`/proc/fb` 卡住的机制也终于清楚了（`drivers/video/fbdev/core/fb_procfs.c`）：

```c
static void *fb_seq_start(struct seq_file *m, loff_t *pos)
{
	mutex_lock(&registration_lock);      /* ← register_framebuffer() 也拿这把锁 */
	return (*pos < FB_MAX) ? pos : NULL;
}
```

modeset 卡在 `register_framebuffer()` 里 ⇒ `registration_lock` 不释放 ⇒
读 `/proc/fb` 阻塞 ⇒ `drm` 命令停在第五项。**所以"卡住"确实是内核卡了，
而且卡在 modeset 里**，不是工具的问题（工具只是把它暴露出来）。

**v2fiftysixth 的两处修复**：

1. **内核/DT**：在 `dispsys_config@14116000` 上加 `force_no_prim_dual_pipe;`。
   `mtk_drm_helper.c` 里有现成的开关：
   ```c
   if (of_property_read_bool(dev->of_node, "force_no_prim_dual_pipe"))
           mtk_drm_helper_set_opt_by_name(tmp_opt, "MTK_DRM_OPT_PRIM_DUAL_PIPE", 0);
   ```
   根因是 stock 基树 dispsys 节点的 `helper-name`/`helper-value` 数组**最后一项**就是
   `"MTK_DRM_OPT_PRIM_DUAL_PIPE"` = `1`（我们一直继承着它）。已在编译产物里核对
   `force_no_prim_dual_pipe` 存在。
2. **initramfs**：`cmd_drm()` 重排 —— 先读 lock-free 的 `mtk-disp-trace` 和寄存器，
   把需要 DRM/fbdev 锁的 `/sys/kernel/debug/dri/0/state` 与 `/proc/fb` **放到最后**。
   这样即使 modeset 还卡，也能先拿到 trace/寄存器。

预期刷 v2fiftysixth 后：`is_dual_pipe=0`；`drm` 能打印到 trace ring 与寄存器；
若卡死消失，则能看到 modeset 是否真正上屏。

---


---



### 0.10 v2sixtieth 实测：**屏幕点亮，DRM 在驱动**（里程碑）

v2fiftyninth 已经把 `top_clk` 死锁修掉，屏幕**第一次出图**；但每个显示中断都打
`mtk_drm_top_clk_isr_get, top clk off at mutex_irq`，每秒几千行，把串口 console 饿死。
v2sixtieth 补上"没有 top clock"时的计数与锁初始化（`mtk_drm_top_clk_give_up()`），
刷屏消失，串口恢复。

实测证据（**DRM 确实在驱动，不是 simplefb**）：

| 证据 | 内容 |
|---|---|
| DRM fbdev | `[drm] fb0: mediatekdrmfb frame buffer device`；`drm` 的 `/proc/fb` → `0 mediatekdrmfb` |
| 扫描输出 | `drm` atomic state：`plane[37] crtc=crtc-0 fb=94 allocated by = [fbcon] size=1080x2400 pitch=4320` |
| CRTC | `crtc-0: enable=1 active=1`，`mode: "1080x2400": 81 316325 1080 1564 1574 1582 2400 2454 2464 2474` |
| modeset | `CHOPIN-STAGE crtc_enable DONE (all 15 steps)`（之前只到 step3） |
| 中断 | `disp_mutex` irq 316 计数持续增长（2s 约 244 次）；DSC/DSI 时钟引用计数 2（在用） |

**屏幕上看到的 console 就是 `fbcon → mediatekdrmfb → 我们的 CRTC`。** `CONFIG_FB_SIMPLE`
本来就是关的，所以不可能是 simplefb。

**额外收获：DSC / DSI 寄存器与 LK 黄金值逐字节吻合**（对照 `port/DSC-REFERENCE.md`）：

| 寄存器 | 我们 | 黄金值 |
|---|---|---|
| DSC `CON` | `0x00010009` | `0x00010009` ✓ |
| DSC `PIC_W` | `0x01670438` | `0x01670438` ✓ |
| DSC `PIC_H` | `0x095f095f` | `0x095f095f` ✓ |
| DSC `SLICE_W` | `0x00b3021c` | `0x00b3021c` ✓ |
| DSC `SLICE_H` | `0x00770013` | `0x00770013` ✓ |
| DSC `CHUNK` | `0x0000021c` | `0x0000021c` ✓ |
| DSC `BUF` | `0x00002a30` | `0x00002a30` ✓ |
| DSC `MODE` | `0x00010101` | `0x00010101` ✓ |
| DSC `PPS0/PPS1` | `0x000c8089` / `0x020e00aa` | 同 ✓ |
| DSI `PSCTRL` | `0x2c050438` | `0x2c050438` ✓ |
| DSI `SIZE_CON` | `0x09600168` | `0x09600168` ✓ |
| DSI `TXRX` | `0x3c` | `0x3c`（4 lanes）✓ |

**尚未处理 / 需要留意的遗留项**：

1. **帧率与上报值不一致**：DRM 把本模式算成 **81 Hz**（`316325 / (1582*2474)`），
   而厂商把同一组 porches + clock 标为 **60 fps**。实测 `disp_mutex` 中断约
   244 次/2s ≈ 61 帧/s（按 SOF+EOF 两次中断算），**实际约 60 Hz**。
   即厂商的 `.clock = 316325` **并不等于 `htotal*vtotal*fps`**，这与标注不符，
   需要在定稿前把 clock 与 porches 的关系理顺（否则 DRM 上报的刷新率是错的）。
2. `CHOPIN-STAGE config_default_path: mode 1080x2400 vrefresh=90` —— 来自从 l16 继承的
   `dyn_fps.vact_timing_fps = 90`；k10a 的 dfps 表应重新核对。
3. `mode_144hz` 仍是占位（厂商 k10a 表里没有 144 Hz），定稿时应删掉。
4. 背光/亮度（KTZ8863A）尚未验证。
5. 所有定位用插桩仍在，见 §0.12。

---


### 0.11 120 Hz：DRM 上报对了，但线上还是 60 Hz —— **已在 §0.15 解决**

v2sixtysecond 把 120 Hz 设为 preferred 之后，DRM 侧完全正确：

```
mode: "1080x2400": 120 331912 1080 1100 1110 1118 2400 2454 2464 2474 0x48 0x0
modes: 6 条（144 Hz 占位已删）
```

**但 `dmesg HFP` 给出了相反的结论。** DSI 的字节时序由 `mtk_dsi.c` 算出：

```c
	horizontal_frontporch_byte = ALIGN_TO((t_hfp * dsi_tmp_buf_bpp - 12), 4);
```

代入两组数：

| 模式 | 像素 HFP | 公式应得 |
|---|---|---|
| 60 Hz | 484 | 484 × 3 − 12 = 1440 = **0x5a0** |
| 120 Hz | 20 | 20 × 3 − 12 = 48 = **0x30** |

实测（v2sixtysecond，即 120 Hz 那版）：

```
CHOPIN-DSI[ddp_prepare] HSA=0x00000014 HBP=0x00000010 HFP=0x000005a0 HSTX_CKL=0x00010000
```

**和 60 Hz 那版一字不差。** 也就是说 0x5a0 精确对应 **484**，即 **60 Hz 那一档的 HFP** ——
模式里的 HFP 从 484 改成 20，**根本没有写到 DSI 上**。

这同时解释了那个疑点：`disp_mutex` 中断速率两版都是 244 次/2.012 s
（≈121/s ÷ 2 = 60.6 Hz），**真实帧率一直是 ~60 Hz**。

| 层面 | 状态 |
|---|---|
| DRM connector 上报 | 120 Hz ✓（是 `mode->clock` 改动的效果） |
| DSI / 面板实际 | **~60 Hz** |
| 120 Hz 目标 | **尚未实现** |

**下一步（已定位到两处之一）**：`dsi->vm.*` 的赋值是对的（`mtk_dsi.c:3202-3210`
逐项取自 `adjusted`），写入也在（`1895-1897` `writel(dsi->hfp_byte, ...)`），
所以断点在：

1. `mtk_dsi_config_vdo_timing()` 在 120 Hz 这次 modeset 里是否真的被调用；
2. `t_hfp` 是否真的取自 `dsi->vm.hfront_porch`，还是取自某个固定值 / golden setting
   （0x5a0 精确等于厂商 60 Hz 档的 484，**倾向这一种**）。

建议做法：在 `t_hfp` 的计算处插一组日志（`t_hfp` / `t_hsa` / `t_hbp` /
`dsi_tmp_buf_bpp` / `dsi->vm.hfront_porch` / 该函数的调用次数），一次刷机定性。
**这正是 §0.12 里插桩暂时不能删的原因。**


### 0.11.1 顺带修掉：`dmesg <词>` 只能搜最后 512 KiB

`dmesg HFP` 一开始返回 0 行，不是那行不存在，而是
`port/initramfs/init.c` 里：

```c
#define KLOG_MAX	(512 * 1024)
...
n = syscall(SYS_syslog, 3, buf, KLOG_MAX);   /* READ_ALL 只返回缓冲区最后 len 字节 */
```

环是 4 MiB（`log_buf_len=4M`），而 `drm` 工具的输出是写进 `/dev/kmsg` 的，
跑几次 `drm` 就把窗口推过了早启动那段。v2sixtythird 改成按
`SYSLOG_ACTION_SIZE_BUFFER` 分配后再读（上限 16 MiB）。

**临时绕过**：重启后第一件事跑 `dmesg HFP`，别先跑 `drm`。


### 0.11.2 **根因已定**（v2sixtyfourth 埋点结果）

埋点后 `dmesg CHOPIN-DSI` 只出现 `[mode_set]`，**没有 `[calc]`，也没有 `[write]`**：

```
CHOPIN-DSI[mode_set] in: clock=331912 hdisplay=1080 hss=1100 hse=1110 htotal=1118 ...
```

⇒ 模式确实带着 20 像素 HFP 进了 `mtk_dsi_mode_set()`（`dsi->vm.hfront_porch = 20`），
但 **`mtk_dsi_calc_vdo_timing()` 与 `mtk_dsi_config_vdo_timing()` 一次都没被调用**。

完整链条：

```
mtk_dsi_probe()
  #ifndef CONFIG_MTK_DISP_NO_LK                 ← 未定义 ⇒ 块生效
      if (stage == DISP_HELPER_STAGE_NORMAL)    ← MTK_DRM_OPT_STAGE = 0 ⇒ 成立
              phy_power_on / clk_prepare_enable
      dsi->output_en = true;                    ← 无条件设置
  #endif
        ↓
mtk_output_dsi_enable()
  if (dsi->output_en) { ...; return; }          ← 第一次就命中，直接返回
        ↓
mtk_preconfig_dsi_enable()          ✗ 不执行
        ↓
calc_vdo_timing() / config_vdo_timing()  ✗ 永不调用
        ↓
DSI 的 HSA/HBP/HFP/VSA/VBP/VFP 保持 LK 留在寄存器里的值
```

**所以 `HFP_WC=0x5a0` 不是"算错"，而是"从没算过"** —— 那是 LK 的配置。

这**不是我们的移植 bug，是厂商的既有设计**：`CONFIG_MTK_DISP_NO_LK` 未定义时，
驱动假定 LK 已经把 DSI 配好并流起来，内核直接沿用。厂商 Android 在运行期改刷新率
走的是 **DFPS / mipi-hopping 路径**（`mtk_dsi_clk_change()`、`mtk_dsi_dy_fps_cmdq_cb()`），
那两条**会**重算并重写时序；但普通 modeset 不会。

**后果**：连接器上报的 120 Hz 与线上时序无关 —— 实际刷新率是 LK 决定的那一个，
DRM 侧的 "120 Hz" 只存在于软件记账里。

**两条修法**：

| | 做法 | 代价 / 风险 |
|---|---|---|
| **B（已做，在 `dev`）** | 在 adopt 的提前返回分支里补上 calc+config，用当前 mode 编程 DSI 时序 | 最小改动、不重新初始化面板，能立刻回答"mode 是否真正掌控了线上时序"。风险：DSI 正在流式输出时改时序可能有一次瞬态 |
| **A（终极）** | 定义 `CONFIG_MTK_DISP_NO_LK`，内核不再沿用 LK，自己完成整个 bring-up | 架构上正确，并且自然带出 §0.4 的"原厂初始化序列"；风险最高（初始化序列不对就是黑屏） |

**先做 B，因为它是判定性实验**：若上屏正常**且** `[write]` 显示
`wrote HSA/HBP/HFP=... read back 0x...` 与模式一致、中断速率随之改变
⇒ mode 真正掌控了硬件，120 Hz 名副其实；若花屏/黑屏 ⇒ LK 的时序是承重的，必须走 A。


### 0.11.3 判定性实验结果：**B 不够，必须走 A**（v2sixtyfifth）

在 adopt 分支里补上 `calc+config` 之后：

| 观测 | 结果 |
|---|---|
| `[calc] #1 hop=0 vm(hfp=20 hbp=8 hsa=10 vfp=54) -> t_hfp=20 ... bpp=3 lanes=4` | 模式的值正确进入计算 |
| `[write] wrote HSA/HBP/HFP=20/16/48 ...; read back 0x14/0x10...` | **写生效**：HFP=48（模式值），不再是 LK 的 `0x5a0` |
| `disp_mutex` 中断 | **244 → 488 每 2.012 s（正好 ×2）** |
| 画面 | **花屏**（品红横向条带 + 竖直接缝） |

**两个重要结论**：

1. **刷新率确实翻倍了（~60 Hz → ~120 Hz）**，这也给"121/s 到底是 60 还是 120"
   这个悬案定了性：中断是**每帧两次**（SOF+EOF），`121/s ÷ 2 = 60.6 Hz`
   ⇒ **LK 把面板配在 60 Hz**。mode 确实开始掌控线上时序了。
2. **但画面坏了** ⇒ 只补 DSI 时序**不充分**。

**原因就在我们自己的面板驱动里。** `lcm_prepare()` 内：

```c
if (ctx->dynamic_fps == 120) {
        ...
        lcm_dcs_write_seq_static(ctx, 0x18, 0x21);   /* 告诉 DDIC 跑 120 Hz */
}
```

**面板的帧率寄存器（page 0x25 / reg 0x18）从未被写过** —— adopt 模式下
`ctx->prepared` 在 probe 就被置 true，`lcm_prepare()` 整个不执行。
于是：**DSI 按 120 Hz 送数据，DDIC 还以为自己在 60 Hz。**

⇒ 改 DSI 时序却不通知面板，正是产生这种"半配置"花纹的原因。

**⇒ 修法确定为 A**：定义 `CONFIG_MTK_DISP_NO_LK`，让驱动不再沿用 LK，
自己走完整有序的 bring-up：
`poweron → 面板初始化（含 0x18）→ DSI 时序 → 开流`。

**A 所需的代码其实已就位**：`lcm_prepare()` 里的 120 Hz 写入与
`ctx->dynamic_fps` 同步逻辑都写好了，只是 adopt 模式下永不执行。
**所以 §0.4 的"原厂初始化序列"不是收尾项，而是 120 Hz 的必要前提。**

**状态**：`001ec6e` 回退该改动，`24e8d1f` 仅递增 tag；
当前 `dev` = master + 日志，画面可用（`v2sixtysixth`）。

**一次操作失误（已修正，记录备查）**：第一次回退时我只在**镜像仓库**里做了
`git revert`，**没有改工作树** —— 而工作树才是构建的源头。后果有两个：
从带 patch 的工作树又构建了一次；并把带 patch 的文件 `cp` 回镜像仓库、
把 revert 覆盖掉了。修正方式：工作树与 `001ec6e` 逐字节一致（`diff -q` 验证）；
`mtk_dsi.o` 晚于源码、`Image` 晚于 `.o`；并用 `objdump` 确认
`mtk_output_dsi_enable()` 对 `calc/config` 的调用数为 **0**。那个坏镜像已删除。

**顺带确认**：`v2sixtysixth` 与 `v2sixtyfourth`（已知可用）的差异 = tag 串 +
`CONFIG_IKCONFIG` 内嵌的 gzip 配置块（tag 在 `CONFIG_CMDLINE` 里，改 tag 会
重新生成整个压缩块，约 70 KB）+ 版本串导致的偏移，**行为等价**。


### 0.11.4 原厂内核到手后的核对（重要，含两处真实错误）

原厂内核在 `n10p/vendor/kernel_mt6893-chopin_kvm/`，面板驱动是
`drivers/gpu/drm/panel/panel-k10a-36-02-0a-dsc-vdo.c`（1740 行）。
逐项核对结论：

**① DDIC 帧率寄存器映射：我们错了。** 原厂初始化表的注释与
`mode_switch_to_{60,90,120}()` 一致地说明：
`0x22 = 60.5Hz`、`0x21 = 91.2Hz`、`0x20 = 121.1Hz`。
我们沿用 l16 的约定（`0x21` = 120 Hz），于是：90 Hz 写 `0x22`（实为 60.5），
120 Hz 写 `0x21`（实为 91.2），**`0x20` 在驱动里根本没出现过** ——
也就是说**这个驱动从来没有真正到过 120 Hz**。已修 5 处。

**② 初始化序列是 l16 的，对 k10a 是错的**（已换成原厂表）。差异包括：
缺 `{0x3B,5}`、`0xC1` 的 DDIC DSC 字节不同、`0xC2` 是 `0x10F0`(4336) 而非
`0x1BA0`(7072)、缺整个 DSC 使能块、缺 `{0x51,2,{0x03,0xFF}}` 背光、
多一条 `0xE9`、`0x9F` vs `0x9C`、`0x29` 后缺 30ms delay。
**把 l16 的表发给 k10a 必然黑屏。**

**③ `mode_flags`：我们抑制 EOT 包，原厂要发。** 主线无正向 flag
（默认就发 EOT），等价做法是**不设** `MIPI_DSI_MODE_NO_EOT_PACKET`。已修。

**④ `vfp_low_power` 是 l16 的值**（2600/1330/2600），原厂逐档为
2528/673/549/879/1294/2528。已修。

**⑤ 其余核对**：porch（HFP 484/174/20、VFP 54、HSA 10/HBP 8/VSA 10/VBP 10）、
`DATA_RATE 834`、整个 `dsc_params` 块、`pll_clk`、`output_mode`、
`lcm_color_mode` —— **逐项与原厂一致** ✓。
缺的功能项（非花屏因）：`esd_check_enable`/`lcm_esd_check_table`、
`lfr_*`、`round_corner_en`、`.dyn`（原厂 `switch_en=0`，本身禁用）、
`physical_*_um`（原厂 68785/152856 vs 我们 70200/152100）。


### 0.11.5 三次"局部补寄存器"均失败 —— 结论：改走 A

| 版本 | 做了什么 | 结果 |
|---|---|---|
| v2sixtyfifth | 只补 DSI 字节时序 | 帧率 60→120 Hz ✓，**花屏** |
| v2sixtyseventh | 再加 DDIC 的 `0x18`（DSI+DDIC 一起，学原厂 DFPS） | 仍**花屏** |
| v2sixtyeighth | 再加 EOT 与 `vfp_low_power` 修正 | 仍**花屏** |

三者都**确实生效**（`[write]` 读到 `0x14/0x10`、中断 244→488/2s），
但画面始终坏。⇒ 问题不是某一个寄存器，而是**整体配置的一致性与建立顺序**：
我们在 LK 不透明的状态上做局部改写，而原厂是让整条流水线（DSC / DDP / DSI / 面板）
按同一个模式从头建立。

**动手 A 之前必须先分清的一个岔路**（否则 A 可能白做）：

- **(i) 流水线状态不一致** ⇒ 修法是 A（驱动自己走完整 bring-up）；
- **(ii) DDP 带宽/时钟不足** —— 120 Hz 要求 DDP 吞吐翻倍，而所有显示时钟仍停在
  **546 MHz**，本板的 **mmdvfs 动态调频并未接入**；跟不上就是 RDMA FIFO 欠载 ⇒ 错行 ⇒
  同样这种花纹 ⇒ **修法不是 A**。

判定命令（v2 栈里有 `RDMA_FIFO_UNDERFLOW` 中断）：

```bash
chopin# dmesg underflow
chopin# dmesg UNDERRUN
```

有 underflow ⇒ (ii)；没有 ⇒ 倾向 (i)，走 A。


### 0.11.6 **根因确定：显示侧从未投过带宽票**（v2seventieth 待验证）

`dmesg underflow` 给出了定论：

```
[DISP][E][IRQ] DDP_COMPONENT_OVL0_2L: frame underflow! cnt=0
[DISP][E][IRQ] DDP_COMPONENT_OVL0:    frame underflow! cnt=0
```

**OVL 欠载** ⇒ 是 DDP 取不到像素，不是"配置顺序不一致"。**A 修不了这个，幸好先查了。**

**成因**：`mtk_drm_helper_init()` 是**按名字**匹配 DT 的 `helper-name`，
**名字缺失的选项一律留 0**：

```c
	index = of_property_match_string(dev->of_node, "helper-name", help_info[i].desc);
	if (index < 0)
		value = 0;
```

而原厂 DTB 的列表里**既没有 `MTK_DRM_OPT_MMQOS_SUPPORT` 也没有
`MTK_DRM_OPT_MMDVFS_SUPPORT`** —— 因为**原厂内核的枚举里根本没有这两项**，
原厂把同一段代码gate在**编译期** `MTK_FB_MMDVFS_SUPPORT` 上。
MT6895-Mainline 把它改成了 DT 选项，于是到了我们这棵树上**两项都是 0**：

- 所有 `mtk_drm_set_mmclk()` 都被 `MTK_DRM_OPT_MMDVFS_SUPPORT` 守着 ⇒ **时钟不缩放**；
- **显示侧从不投互联带宽票** ⇒ SMI 侧不升优先级 ⇒ FIFO 欠载。

`mtk_dsi_set_mmclk_by_datarate_V1()` 本身是好的 —— 它明确用
`drm_mode_vrefresh()` 算所需像素时钟，**能区分 60 与 120 Hz**。

**注意：时钟不是杠杆。** `disp_parents[]` 最快只到 `mainpll_d4` = **546 MHz**，
而显示本来就跑在 546 MHz —— 所以缺的是**带宽投票**，不是频率。
两项都以 `0x01` 追加（按名字匹配，位置随意；注意最后一个已有名字后必须补逗号，
否则 DTS 会把相邻字符串字面量拼接）。


### 0.11.7 操作记录（两次失误）

1. **命名冲突**：我写了 `v2thirtieth`，但该名字在老的 v2 序列里**已经用过**
   （`SHA256SUMS` 第 100 行）。规律是 `v2sixtieth → …sixty-ninth → seventieth`，
   应为 `v2seventieth`。`cp` 时**覆盖了那个老镜像**（哈希 `613ae0db…` 仍在
   `SHA256SUMS`，镜像本体已丢失）。已改名重编为 `v2seventieth`。
2. **`build=2` 却照样打包**（v2sixtyeighth 首次构建失败仍打包了 stale Image）。
   靠比对哈希发现，已删除该产物与对应 `SHA256SUMS` 行。

两次都属于"改了但没生效"，现在每轮都做哈希/符号级校验（`objdump` 看调用、
`sha256sum` 对比前后版本、`dtc` 反解 DTB 确认属性真的进去了）。


### 0.12 内核仓库历史整理（已完成）

`git/chopin-linux-7.2.8` 从 `99df977^`（`5baaae9`）起重做：**121 个迭代提交 → 10 个逻辑提交**，
改动文件按功能单元归类，**0 遗漏**。

| # | 提交 | 文件数 |
|---|---|---|
| 1 | `3b61b11` include: headers the vendor display stack links against | 8 |
| 2 | `e0e8cea` soc: mediatek: MT6893 mmsys, mutex and scpsys for the v2 master | 4 |
| 3 | `407ff53` memory: mediatek: downstream SMI common/larb for the display path | 3 |
| 4 | `9f2e0ac` misc: mediatek: cmdq, interconnect and mmqos glue for the v2 stack | 69 |
| 5 | `27f22eb` drm/mediatek: import the vendor mediatek_v2 display stack | 215 |
| 6 | `b1894e1` drm/mediatek: import the MML (media memory layer) block | 46 |
| 7 | `c19c032` drm/mediatek: MT6893 DSC wrap support and bring-up trace | 13 |
| 8 | `5464bf8` drm/panel: k10a 36 02 0a (DSC) and the KTZ8863A backlight | 9 |
| 9 | `db6ee28` arm64: dts: chopin board tree for the v2 display stack | 1 |
| 10 | `6aa05ae` arm64: configs: chopin_defconfig for the v2 display stack | 1 |


**顺带清掉一个杂物**：镜像仓库里提交了 `drivers/memory/chopin_defconfig` ——
一份 1944 行的 defconfig 旧快照被误写进驱动目录，**没有任何东西引用它**，
而且**工作树里从来就没有它**（是早期某次 `cp -a` 带进去的）。
整理时把它排除了，所以现在的对应关系是：

```
git diff chopin-v2-working master      → 只有 1 项：D drivers/memory/chopin_defconfig
committed tree vs linux-7.2.8 工作树   → 0 处差异          ← 真正该保证的不变量
```

第二条是用 `GIT_DIR=<repo>/.git GIT_WORK_TREE=linux-7.2.8 git status --porcelain`
对**全部 95k 文件**做的，不只是 port 那 404 个 —— 也就是说
**"提交的源码"与"我们构建、刷机的源码"现在逐文件相同**。

- 回退点：tag **`chopin-v2-working`** 仍指向整理前的旧历史（`21d1ba4`），随时可回；
- 全部历史 35 个提交，从 `2ca60ae Linux 7.2.8 (clean upstream baseline)` 算起；
- 整理脚本留在 `port/tools/cleanup-kernel-history.py`（含 `--dry-run`，
  会强制要求每个文件命中恰好一组规则，漏一个就报错退出 —— 这是"文件不会悄悄掉出重写"的保证）；
- **插桩没有删**：§0.11 的 120 Hz 问题还没解决，`CHOPIN-*` 那套日志正是查它要用的。
  等 120 Hz 定性后再单独做那一次改动（届时 tree 会变，需要新的对照 tag）。


### 0.13 整理后从零重建 boot.img 的验证（通过）

担心的是"整理历史时悄悄丢了东西"。所以在一个**全新的 O= 目录**
（`build/chopin-verify`，先前不存在）里从零构建了一遍：

| 检查 | 结果 |
|---|---|
| `make chopin_defconfig` | ok |
| `make -j16 Image` | **build=0，0 errors** |
| `make dtbs` | ok |
| 内嵌板级 DTB `chopin-board.dtb` | `c1c40b2d…` 与既有构建**逐字节相同** |
| `mt6893-xiaomi-chopin.dtb` | `e5d0b6ac…` **逐字节相同** |
| Image 大小 | 44882432，两边相同 |
| Image 字符串全量 diff | 126,499 条里**只有 10 行不同**，全是时间戳 |

```
#244 SMP PREEMPT Mon Oct  5 10:33:37 CST 2026   ← 既有构建
#1   SMP PREEMPT Mon Oct  5 10:54:53 CST 2026   ← 全新构建（新 O=，版本计数器归 1）
```
另有内嵌 initramfs 的 cpio mtime（`6ABCAB4C` vs `6AC30FFE`）。

用全新构建打包出的 boot.img（`port/out-verify/boot-stockdt.img`）按段对比已发布镜像：

| 段 | 结果 |
|---|---|
| ramdisk | **完全相同** |
| dtb | **完全相同** |
| kernel（解压后 44,882,432 B） | **141 B 不同（0.000314%）**，7 个区域 |

141 字节恰好对应上面那两处时间戳（`linux_banner` 的构建时间 + 内嵌 cpio 的 mtime）。

**结论：整理后的提交源码从零构建出的就是同一个内核**，功能等价、只差构建时间戳。
已发布镜像仍是 `chopin-v2sixtythird`（未改动，SHA256SUMS 未动）；重建产物留在
`port/out-verify/` 供对照，`build/chopin-verify/` 是那次全新构建的目录。

**已刷机确认（2026-10-05）**：从整理后源码构建的镜像刷入后**正常上屏**，
`dmesg`（含过滤器）工作正常（`kernel log buffer is 4194304 bytes`、
`recovered 165349 bytes` —— §0.11.1 的修复生效）。现场日志确认：

| 项 | 值 |
|---|---|
| connector 模式 | `1080x2400` × 6 |
| CRTC 模式 | `mode: "1080x2400": 120 331912 1080 1100 1110 1118 2400 2454 2464 2474 0x48 0x0` |
| `dsi_probe` | `panel=ffff0000c45f9288 ext=ffff0000c03c8980`（都非 0） |
| 12 个组件 | 全部 `bound`，含 `14124000.disp_dsc_wrap` |
| fbdev | `[drm] fb0: mediatekdrmfb`，`/proc/fb → 0 mediatekdrmfb` |
| 显示时钟 | `mm_disp_dsc_wrap` / `mm_dsi0_mm_clk` / `mipi_tx0_pll` 引用计数均为 1 |
| `disp_mutex` 中断 | 14174，且以 **244 次/2.012 s（≈121.3/s）** 稳定增长 |

**所以：整理（10 个逻辑提交）+ 从零重建 + 刷机 三步全绿，成果可复现。**

⚠️ 但 §0.11 关于 120 Hz 的疑点**依然在**：这次的中断速率还是 244 次/2.012 s，
与"60 Hz preferred"那一版**完全相同**。两个独立指标（DSI `HFP=0x5a0` 不变 +
中断速率不随模式变化）都指向同一件事：**模式里的 porches 没有到达 DSI**。

而 121.3/s 这个绝对值，若按"中断=每帧一次"读，恰好是 **120 Hz** —— 结合
"原厂 Android 就是 120 Hz、LK 配好后才交给内核"，最可能的真相是：

> **面板确实在跑 ~120 Hz，但那是 LK 留下的配置，不是我们的驱动设的。**
> 我们的模式切换只改变了 DRM 的上报和 `cfg.vrefresh`，没有改变线上时序。

这对"能不能显示"没影响（你看到的就是 120 Hz 的画面），但意味着**驱动并不掌控时序** ——
一旦需要重新初始化面板（休眠唤醒、模式切换、ESD 恢复）就会暴露。
`V2-HANDOVER` §0.11 里写的下一步（查 `t_hfp` 是否取自当前 mode）仍然有效，
而且**插桩必须留着**才能查。

---
（一句话）


### 0.14 分支约定（dev / master）

从这一轮起，**两个仓库都按同一个约定**：

| 分支 | 放什么 |
|---|---|
| **`master`** | 只放**已刷机验证过**的状态。内核仓库目前是 10 个逻辑提交（`6aa05ae`），已通过"从零重建 + 刷机上屏"验证 |
| **`dev`** | 阶段性、探索性的进展先提交到这里；验证通过后再整理成干净提交合回 `master` |

- 内核仓库 `dev` 当前：`24e8d1f`（= master + `5f8c891` 的埋点；中间的
  `97b9524` 是 §0.11.3 那个失败实验，已被 `001ec6e` 回退）。进 dev 不进 master。
- `tag chopin-v2-working`（`21d1ba4`）是整理前的旧历史，作为最终回退点，两个分支都保留。
- 进 dev 的插桩在问题定性后要**折叠掉**（不是简单 revert：要和之前那批 `CHOPIN-*`
  一起清），那时再决定哪些进 master。

---


### 0.15 **120 Hz 达成**（v2seventyseventh，已刷机确认）

**结论：120 Hz 真正跑起来了，而且是走 GNU/Linux 的标准路径实现的 —— 不是接管
bring-up，也不是在流中间硬改时序。**

**实测依据**：

```
disp_mutex 中断:  delta 244/2.012s (≈60 Hz)   →   delta 486~488/2.012s (≈242/s ⇒ 120 Hz)
```

**实现路径（三步，缺一不可）**：

1. **`ext_param` 同步**（内核 `5cea847`）：GNU/Linux 改刷新率走标准 CRTC 模式，
   不会像 Android 的显示 HAL 那样写 MTK 私有的 `CRTC_PROP_DISP_MODE_IDX`。
   栈里那段同步代码本来就有，但位置在 adopt 的 early-return **之后** ——
   正常开机永远不执行。抽成 `chopin_sync_panel_ext_params()` 并在提前返回前也调用。
   （这正是 MT6895 开发者留言指出的修法。）
2. **一次真正的模式切换**：initramfs 新增 `setmode` 命令（`909b22d`），用 DRM uapi
   做 `SETCRTC`。这会触发 DRM 的 **disable + enable**，于是
   `mtk_output_dsi_enable()` 这次 `output_en == false`，走完整路径：
   `mtk_preconfig_dsi_enable() → calc_vdo_timing() → config_vdo_timing()`，
   **用新模式的参数编程 DSI 时序**（正规时序，不是流中间硬改）。
3. **面板侧自然补齐**：模式切换会 `unprepare`，清掉 `ctx->prepared`，于是
   `lcm_prepare()` 真正执行（原厂 k10a 初始化序列），且此时 `dynamic_fps` 已是 120
   ⇒ 写 `0x18 = 0x20`（121.1 Hz）⇒ **DDIC 与链路终于一致**。

**实测日志**：

```
[mode_set] in: clock=331912 htotal=1118        ← 120 Hz 模式
[ext_param] sync idx=0 fps=120                ← 第 1 步
[output_enable] output_en=0                   ← 第 2 步：不再走 adopt 提前返回
[preconfig+] → [calc] #2 vm(hfp=20 ...) → [write] HFP=48 ... read back 0x14/0x10
DSI START=0x00000001                          ← 流在跑
XAGA-PANEL[prepare] dynamic_fps=120           ← 第 3 步：面板初始化 + 0x18=0x20
```

**顺带获得的能力**：六档模式（120/90/60/50/48/30）现在都能用 `setmode` 在**运行期**切换 ✓
—— 这是"驱动真正掌控显示"的标志，而不只是把面板点亮。

**三条被实际排除的路径**（都做过，不是推测）：

| 路径 | 结果 |
|---|---|
| 在 adopt 分支里直接补 `calc+config`（流中间改时序） | 花屏 —— **路径错了**，不是参数错了 |
| `CONFIG_MTK_DISP_NO_LK` 让驱动接管整个 bring-up | 黑屏；且**原厂全树从未定义过它**（4 处 `#ifndef` 在原厂全部生效），MT6895 也只是注释状态 |
| DDP 带宽 / `MMQOS` / `MMDVFS` 选项 | 与花屏无关。启动时一次性 OVL underflow（各组件一次，`cnt=0`）是**良性**的 —— 我先前"underflow ⇒ 花屏"的推断过强 |

**一条重要的诊断教训**：`chopin_dump_dsi()` / `chopin_dump_disp()` 用了 5 个
**硬编码且全错**的地址（DSI 读 `0x14017000`，实际在 `0x1400e000`），
所以那些寄存器 dump 在很长时间里全是零、毫无意义（`9edb60a` 已修）。
**用 `dsi->regs` 读的 `CHOPIN-DSI[...]` 行一直是可信的** —— 早先
"HFP=0x5a0 是 LK 的 60 Hz 值"这个关键结论正是建立在那上面，事后证明成立。


### 0.16 收尾：插桩清除 + 开机即在 120 Hz（v2eightieth，已刷机确认）

**两件都完成并实测通过。**

**① 插桩清除。** `mtk_dsi.c`（47 条 transfer/calc/write/poweron/enable/irq_enable）、
`mtk_drm_drv.c`（41 条 + 三个 dump 函数本体 + probe 横幅）、`mtk_drm_crtc.c`（16 条 + 5 处
dump 调用）、`panel-k10a-bringup.c`（20 条，含 53 条 per-command 打印 —— 最费时的一批）、
以及 12 个显示子块与 `mtk_mipi_tx`/`mtk-smi-downstream`。

判据（不是 `build=0`，而是 Image 内容）：`CHOPIN` 只剩 **1** 处（有意保留的
`CHOPIN-DTB: overriding LK FDT…` 启动提示），`XAGA` = **0**，面板驱动与背光字符串仍在。
镜像体积 16277504 → 16267264。

**这不是纯整洁**：`console=ttyS0` 是同步的，cmdline 要 921600 而 UART 实际 115200 ⇒
每行约 9 ms 自旋，而面板初始化一次要打约 50 行。

**误删救回**：5 条真实错误信息（`remote panel node missing`、`kzalloc failed`、
`drm_panel_of_backlight failed`、`mipi_dsi_attach failed` 等）只是恰好带 tag，已去标签写回。

**工具与教训**：`port/tools/strip-trace.py` 是可用的清理器 ——
*只有当打印宏位于该行行首、且括号配平到末尾 `;` 时才删*，并顺带删掉前面的无花括号 guard；
其余一律报告而不猜。`strip-instrumentation.py` 保留仅作记录，它的两个错误都**只被编译器发现**：
向后找语句边界时忽略括号内的 `;`（在 `mtk_disp_c3d.c` 里删坏了没有插桩的区域）；
用 `^[^\n]*…\n` 删 Makefile 规则时把 `mediatek-drm-y :=` 目标一起删掉。
另外 `mtk_smi_stub.c` **不是**插桩残桩 —— 它定义了 `mtk_dma_buf_set_name`，`mtk-mml-buf.c` 需要它，
按文件名想当然地删掉会导致链接失败；已恢复并保留。

**② 开机即在 120 Hz。** 内核沿用 LK 的显示状态，所以线上是 LK 的速率，而 DRM 已经上报
preferred（120 Hz）—— **直接设 120 Hz 是空操作**。initramfs 因此**先切 60、再切回 preferred**，
两步都是真实模式切换，第二步把 DSI 编程为 120 Hz。

**第一次尝试失败，原因是时间点**：bring-up 在 t=2.26 s 执行，而 DRM 到 t=11.95 s 才注册
`card0`（显示组件 defer probe）⇒ `setmode: cannot open /dev/dri/card0`。
修法：**轮询等待 `card0`（最多 30 s，超时明确报错），并放在 `fork()` 出的子进程里执行**，
父进程直接进 shell —— 否则控制台要陪着等那 10 秒。

**实测**：`delta` 从 244/2s（60 Hz）变为 **484~486/2s（240+ /s ⇒ 120 Hz）**，
切换日志与两次 `mi_dsi_display_wakeup_pending_doze_work` 吻合。

**⇒ 这同时使高刷对任何 DRM 桌面环境可用**：桌面自己的初始 modeset 设 preferred 也是空操作，
但硬件在此之前已经被切对了。

**③ 日志噪音清理（v2eightysecond，已刷机确认）。** initramfs 的 IRQ 监视器原本每 2 秒
打 6 行、永不停止 —— 原因是判据 `!was || delta * 10 < was * 9 || ...`，而**显示中断里有一批
永远不触发**（ccorr/AAL/dither/OVL×2/wdma/DSC wrap），它们的 `delta` 恒为 0 ⇒ `prev_delta`
恒为 0 ⇒ `!was` **永远为真**。这是我在同一个 6 行判断上连着犯的第二个错（第一个是漏掉空闲
中断）。

正确判据**先判"变了没有"**：

```c
if (delta != was && (delta == 0 || was == 0 ||
    delta * 10 < was * 9 || delta * 10 > was * 11))
```

| 情形 | 结果 |
|---|---|
| 0 → 0（空闲） | 静默 |
| N → N（稳定） | 静默 |
| 372 → 486（速率变化） | 打印（486 > 372×1.1） |
| 0 → N（变活） | 打印一次 |
| N → 0（停止） | 打印一次 |

实测：注册清单一次 + 速率变化的 `#6`/`#7` + 每 30 秒一行心跳，之后完全安静 ✓。

**教训**：这类条件我以后**先列真值表再写**。前两次都是我没把边界情形（`delta` 恒为 0）
在动手前想清楚，两次都靠设备日志才发现。

**命名**：`v2thirtieth`/`v2thirtyfirst`/`v2thirtysecond` 三个名字都重用了早期序号（其中两个在
`SHA256SUMS` 里已有条目）；本版按 `v2seventyninth` 之后的高位续为 **`v2eightieth`**。


### 0.17 P1：WL2866D 驱动（v2eightythird，已刷机确认绑定）

面板 DVDD 是第一颗 WL2866D 的 LDO1（1.2 V），而树里一直没有驱动 —— 面板驱动查的是
`vrf18`（本板不存在），所以这条轨从移植开始就一直是 LK 留下的状态。这就是 P1。

**先核对、后动手，抓到三处会写错的地方：**

1. **有两颗，不是一颗**：`willsemi,wl2866d1`(index 0)@**i2c7**、`willsemi,wl2866d2`(index 1)@
   **i2c9**，同地址 `0x28`，且**两颗的 4 个 LDO 名字完全相同** ⇒ 面板靠 **phandle** 而非名字
   找到供电 ⇒ 驱动必须注册两颗（但见下）。
2. **原厂那条 guard 的 index 与本板不符**：它针对 k10a 面板时检查 `index == 1`（第二颗），
   而本板 DT 的 `dvdd-supply` 指向**第一颗**的 `on,wl2866d1@1` ⇒ **以 DT 为准**，guard 条件写成
   index 0 + offset 1，并在注释里写明这个差异。
3. **原厂 703 行里约 400 行是 camera-LDO 字符设备**（`camera_ldo` class、`wl2866d_dev_list`）
   ⇒ 面板完全用不到，整块丢掉；`parent_supply` 也没有任何节点声明 ⇒ 同样丢掉。

**实测验证（这是关键）**：

```
[1.297] wl2866d-regulator 7-0028: wl2866d-dvdd1: set to 1050000 uV (vset 75)
[1.299] wl2866d-regulator 7-0028: wl2866d-dvdd2: set to 1200000 uV (vset 100)
[...]   四个 LDO 全部 registered

regulator_summary:  wl2866d-dvdd2   0  0  0 unknown  1200mV
```

`600000 + 100 × 6000 = 1,200,000 µV` 与 DT 的 `init-voltage = 0x124f80` **精确吻合** ⇒
**电压数学正确、驱动确实在与芯片通信** ✓。

**一颗没绑上（原因未定，不猜）**：只有 `7-0028`（i2c7 = wl2866d1）注册，第二颗没有。已排除的：
两个节点 `status` 均未声明（默认 okay）、i2c9 节点是 `status = "ok"`、其总线确实起来了
（**注册名是 `i2c3`@`11cb0000`**，DT 里节点名叫 `i2c9@11CB1000`，靠 aliases 映射）。
**怀疑**是那颗节点并非 `i2c9` 的子节点（两者相隔 79 行，中间可能有闭合括号），但**没有验证**。
**与面板无关**（面板在第一颗上），若要接摄像头轨再查。

**那条 guard 尚未触发**（`dmesg keeping` 为空）✓ —— 预期，因为目前没有任何东西申请过这条轨，
且 cmdline 有 `regulator_ignore_unused`。

**下一步**（才是有风险的那一步）：让面板驱动真正使用 `dvdd` —— 面板节点加 `dvdd-supply`，
驱动把 `vrf18` 换成 `dvdd`。相对安全：provider 在 1.29 s 注册，面板在 11.88 s 才 probe ⇒
**不会 EPROBE_DEFER** ✓。验证判据：`wl2866d-dvdd2` 的 `use_count` 变成 1。


### 0.18 **P1 前半闭环：面板 DVDD 由内核掌控并已使能**

判据取自 **regulator 框架自己的计数器**，不是我加的打印：

```
=== wl2866d regulators (use = enable refcount, open = consumers) ===
  wl2866d-dvdd2          use=1 open=1 bypass=0
      consumer 1400e000.dsi.0-dvdd
  wl2866d-dvdd1          use=0 open=0 bypass=0
  wl2866d-avdd1          use=0 open=0 bypass=0
  wl2866d-avdd2          use=0 open=0 bypass=0
```

`use=1`（使能引用计数）× `open=1`（取到它的消费者数）× 消费者是 `1400e000.dsi.0-dvdd`，
三条独立成立 ⇒ **面板持有并启用了这条轨** ✓；其余三条 `use=0 open=0` ⇒ **没有误动** ✓。

**达成它需要三处同时正确**，缺一不可：
1. DT 给面板节点 `dvdd-supply`（指向 `wl2866d1@1`，构建期由 dtc 校验路径）；
2. 面板驱动取 `dvdd` 而非本板不存在的 `vrf18`；
3. WL2866D 驱动设置 `REGULATOR_CHANGE_STATUS` —— **少了它与内核会以 -EPERM 拒绝消费者的
   `regulator_enable()`**，表现为 `open_count=1` 而 `use_count=0`（挂上了但打不开）。

**原厂那条"第一次 disable 拒绝"的 guard 是必需的**，实测触发了
`init phase: keeping wl2866d-dvdd2 enabled`：掩码修好、供电变得可关之后，确实有 disable 尝试。

**这一段花了很久，值得记下原因**：`init` 里的 `print_file()` 把读到的内容送到 `/dev/kmsg`
而不是 stdout ⇒ `cat <debugfs文件>` **每次都读对了值却什么都不显示**。之后加了
`cat_file()`（输出到控制台）并在 `drm` 里加了上面那段计数器 —— **从此验收不依赖手抄，
也不依赖我写的打印**。


### 0.19 背光与 P1 的另一半是**同一颗芯片**：LM36273

**结论：做 P1 剩下的 AVDD/AVEE，就是在做亮度调节。** 原厂注释给出拓扑（也验证了已完成的那条）：

```
/*AVDD --> VSP      --> LM36273_VREG_DISP_P(GPIO126/bias0)*/
/*AVEE --> VSN      --> LM36273_VREG_DISP_N(GPIO126/bias0)*/
/*DVDD --> LCD_DVDD --> WL2866_DVDD2(GPIO148/bias1)*/
```

原厂 k10a 面板驱动通过它调亮度：`lm36273_brightness_set()` / `lm36273_bias_enable()` /
`lm36273_bl_bias_conf()` —— **直接函数调用，不走 backlight class**。

**DT 节点一直都在**，只是没人绑上：

```dts
i2c_lcd_bias@11 {                       /* 在 i2c6@11F01000 上 */
    compatible = "mediatek,i2c_lcd_bias";
    reg = <0x11>;
    status = "okay";
};
```

**为什么编进去的背光驱动从不 probe**：`of_compat_cmp` 就是 `strcasecmp` ⇒ compatible 比对
**忽略大小写** ✓ ⇒ 原厂的 `mediatek,I2C_LCD_BIAS` **能**匹配 `mediatek,i2c_lcd_bias` ✓，而树里
已有的 ktz8863a 驱动要的是 `mediatek,MI_I2C_LCD_BIAS`（多一个 `MI_`）✗ ⇒ **永不匹配** ✓。
**⇒ 移植 LM36273 不需要改 DT** ✓，总线地址（`i2c6@11F01000` @ `0x11`）来自 DT 本身 ✓。

**端口改动**（7.2.8 API 三处）：`of_gpio.h` 已删除；`probe` 去掉 `i2c_device_id` 参数；
`remove` 返回 `void`。原厂 `remove` 里还调了 `i2c_unregister_device(client)`（注销正在被注销的
设备 ✗）—— 一并去掉。

**并且给 probe 加了一次 REV 寄存器读取**：原厂 probe 不校验任何东西 ⇒ "绑上了"只证明 DT 里
有个节点 ✗；**寄存器能读通才是"芯片在、地址对"的第一个能失败的判据** ✓。


### 0.20 三个检查同时漏掉了一个新文件（交付物差点不完整）

移植 LM36273 时，**新驱动文件没进仓库** ✗，而**三个检查全都没发现**：

1. **我的同步脚本解析错**：工具打印 `MISSING FROM REPO`，脚本里写的是 `MISSING FROM THE
   REPO` ⇒ 那个正则从未匹配 ✗（又一次"按猜的名字写代码"）；
2. **`verify-drm.sh` 只查了一半**：它只断言 `OLD COPY IN REPO : 0`，**从不查
   `MISSING FROM REPO : 0`** ✗ ⇒ 文件缺失照样通过；
3. **我的 dirty 检查指向错误的树**：`GIT_WORK_TREE=linux-7.2.8` 报 "0 dirty" 是假象 ✗ ——
   仓库真正的工作树是 `git/chopin-linux-7.2.8/`（`linux-7.2.8` 只是**编译树**，两者靠 cp 同步）。

**修法**：
- `audit-repo.py --sync`**：一条命令复制所有缺失/过旧文件并先刷新 manifest（新文件不在 manifest
  里就是不可见的 ✗，所以顺序很重要）✓；已用"删掉仓库那份 → 工具报缺失 → `--sync` 自动恢复"
  端到端测过 ✓；
- `verify-drm.sh` 改用**工具自己的退出码**（0 = 仓库能复现工作树）✓ —— 覆盖缺失、过旧、
  manifest 过期三种情况 ✓；同样端到端测过（删除后 verify 失败，`--sync` 后通过）✓。


### 0.21 面板驱动接上 LM36273（v2eightyninth）

**实测确认**（`dmesg lm36273`）：

```
[1.291641] lm36273_probe: i2c_lcd_bias          <- 大小写不敏感匹配命中 DT 节点
[1.292221] lm36273: chip at 0x11, REV=0x12      <- 寄存器真的读回来了
```

**寄存器能读通**是唯一能失败的判据 ⇒ 总线 `i2c6@11F01000`、地址 `0x11`、芯片型号全部确认 ✓。

**改了三处**：

| 位置 | 原来 | 现在 |
|---|---|---|
| `lcm_prepare` | `ktz8863a_bl_bias_conf()` / `ktz8863a_bias_enable(1)` | `lm36273_bl_bias_conf()` / `lm36273_bias_enable(1, 1)` |
| `lcm_prepare` | （无） | **`lm36273_brightness_set(1023)`** ← 见下 |
| `lcm_setbacklight_control` | `ktz8863a_brightness_set(level)`，**且仅当 level > 2047** | `lm36273_brightness_set(level)`，**每一档都送** |

**为什么必须显式设初始亮度**：原厂靠面板节点的 `backlight` phandle 让 backlight class 去设 ✓，
而我们的面板节点没有这个属性 ✗ ⇒ 没人会设 ✓；而 `lm36273_brightness_set()` **只在电平从 0 上升
时才打开背光** ✓ ⇒ 芯片停在 0 就一直是黑的 ✗。开机那次 120Hz 模式切换会走
unprepare/prepare ⇒ **不设就会黑屏** ✓。所以设 1023（中间值，明显可见，也是验证信号 ✓）。

**刻意只做了一半**：**没有**把 unprepare 接到 `lm36273_brightness_set(0)` 和
`lm36273_bias_enable(0, 3)` ✓ —— 那会关背光并关掉 AVDD/AVEE ✗，而开机模式切换会走
unprepare ⇒ 一旦重开失败就是黑屏 ✗。**打开 + 可调**先落地，「关闭」留到做 suspend/resume 时
一起处理（和 wl2866d 那条 guard 同样的取舍）。

**面板驱动里仍有 12 处 `ktz8863a` 调用**（unprepare / HBM 等路径）—— 接的是从不 probe 的芯片
⇒ **空操作** ✓。这一点由最近几次开机**实测**证明（屏幕一直正常 ✓），不是推断。

**未做**：`bias` GPIO（原厂注释写 `GPIO126/bias0`）—— 编号未经核实 ✓，而 i2c 侧的
`bias_enable(1, 1)` 才是功能性的使能 ✓，所以先不动一个可能驱动错引脚的 GPIO ✓。


### 0.22 **P1 完成：面板全部供电 + 背光已由内核掌控并实测生效**

`dmesg lm36273`（v2eightyninth）：

```
[ 1.275956] lm36273_probe: i2c_lcd_bias
[ 1.276528] lm36273: chip at 0x11, REV=0x12
[12.729450] lm36273_bl_bias_conf backlight and bias setting
[12.730258] lm36273_bias_enable enable = 1              <- AVDD/AVEE 使能
[12.731953] lm36273_brightness_set: bl level = 1023, tmp_bl = 1693, hbm level = 0
[12.733000] lm36273_brightness_set: enable level = 1023 <- 背光从 0 打开
```

**判读依据**：`enable level = %d` 只在 `level > 0 && g_lm36273_led.level == 0` 的分支打印
⇒ 芯片原本停在 0 ✓、这次真的被打开了 ✓。而这整段发生在 12.73s —— 正是开机那次 120Hz
模式切换走 unprepare/prepare 之后 ✓，说明**面板每次模式切换都会重新使能供电和背光** ✓。

| 供电 | 芯片 | 状态 |
|---|---|---|
| VDDI | （本板无） | — |
| **DVDD 1.2V** | WL2866D | ✅ `use=1 open=1`（§0.18） |
| **AVDD / AVEE** | **LM36273** | ✅ `bias_enable enable = 1` |
| **背光** | **LM36273** | ✅ 可写（`enable level = 1023`） |

**⇒ P1「内核接管面板供电」完成。** 余下的是**用户空间接口**：面板节点没有 `backlight`
phandle ⇒ 目前**没有** `/sys/class/backlight/*` ⇒ 亮度能写但**用户改不了** ✓。下一步应在
LM36273 驱动里注册一个 backlight 设备（`update_status` → `lm36273_brightness_set`），
这样 `brightnessctl` / `systemd-backlight` 等标准工具就能直接调 ✓。

**仍未做**（留给 suspend/resume）：unprepare 侧的关闭路径
（`lm36273_brightness_set(0)` + `lm36273_bias_enable(0, 3)`）与 `bias` GPIO ✓。


### 0.23 亮度可由用户空间调节（v2ninetysecond）

**LM36273 驱动注册了一个 backlight 设备**：

```
/sys/class/backlight/lm36273/{brightness,max_brightness,actual_brightness,...}
max_brightness = 2047，默认 1023
```

**刻意不在面板节点加 `backlight` phandle**：面板驱动里已有原厂那段
`ctx->backlight->props.power = FB_BLANK_POWERDOWN` ✓ —— 一旦把 phandle 接上，
**每次 unprepare 都会关背光** ✗，而开机那次 120Hz 模式切换正是走 unprepare ⇒ 黑屏风险 ✗。
现在 `ctx->backlight` 是 NULL ⇒ 那些块是空操作 ✓，用户空间直接用
`/sys/class/backlight/lm36273/` ✓。

**亮度跨模式切换保持**：面板 prepare 会调 `lm36273_bl_bias_conf()` 重写芯片寄存器 ✗，
所以驱动的 `lm36273_bias_enable(1, ...)` 在使能 bias 之后**无条件重放当前电平** ✓
（并把写缓存 `lm36273_last_tmp_bl` 提升为模块状态，否则缓存会导致漏写 ✗）。
⇒ 用户在任意时刻设的亮度，切换刷新率后仍然保持 ✓。

**`init` 新增 `w <路径> <值>`**：这个极简 shell **没有重定向** ✗，
`echo 300 > /sys/...` 打不出来 ⇒ 没有它就无法从控制台驱动任何控制 ✓。


### 0.24 又一次静默失败：管道吞掉退出码，过期 ramdisk 被打包

给 `init` 加 `w` 命令时，`write_file` 调用定义在它**后面**的 `cat_file` ✗ ⇒ **编译失败** ✓。
但 `mkinitramfs.sh` 其实**有** `set -e`、也确实返回了非零 ✓ —— 而我的命令写的是
`mkinitramfs.sh 2>&1 | tail -1` ✗ ⇒ **管道的退出码是 `tail` 的（0）** ✗，我又没开 `pipefail`
⇒ 失败被吞掉 ✓ ⇒ **构建继续，用旧的 `initramfs.cpio.gz` 打了包** ✗，镜像里根本没有新命令 ✓
（用 `strings` 查证 = 0 ✓）。

**三处修**：

1. `init.c`：补上 `cat_file` 的前置声明 ✓；
2. **`build-images.sh` 自己重新生成 initramfs** ✓（`set -e` ⇒ 编译失败就停 ✓）——
   在此之前它**只检查 ramdisk 是否存在** ✗，而过期的和新鲜的**看起来完全一样** ✗；
3. **我的命令一律 `set -o pipefail`** ✓ —— 这是同一个根因的第三次出现（前两次是
   `MISSING FROM THE REPO` 的正则 ✗、和 `GIT_WORK_TREE` 指错树 ✗）：
   **用管道/字符串判断成败，而不是用退出码。**


### 0.25 **亮度调节完成并实测验证**（v2ninetysecond）

设备实测（`ls /sys/class/backlight/` 与六次写入）：

```
OF_FULLNAME       = /i2c6@11F01000/i2c_lcd_bias@11
OF_COMPATIBLE_0   = mediatek,i2c_lcd_bias
max_brightness    = 2047        brightness = 1023        type = raw
w .../brightness 300   =>  300
w .../brightness 2047  =>  2047
w .../brightness 1023  =>  1023
w .../brightness 30    =>  30
w .../brightness 3     =>  3
```

**写入全部回读一致 ✓，且屏幕亮度实际变化 ✓** —— 整条链路打通：
用户空间 → backlight class → LM36273 → 芯片寄存器 → 屏幕 ✓。

`OF_FULLNAME` 独立印证了 §0.19 的结论（节点是 `i2c6@11F01000/i2c_lcd_bias@11`，
compatible 是小写的 `mediatek,i2c_lcd_bias` ✓ —— 而原厂驱动写的大写形式靠
`of_compat_cmp` = `strcasecmp` 匹配上 ✓）。

**⇒ 至此外观层面的可用性齐了**：面板供电（DVDD + AVDD/AVEE）由内核掌控 ✓、背光可控 ✓、
120Hz 可切 ✓、控制台在屏幕上 ✓。

**下一步：suspend/resume**（完全未测）。它需要补上目前刻意留白的部分：
unprepare 的关闭路径（`lm36273_brightness_set(0)` + `lm36273_bias_enable(0, 3)`）、
DVDD/VDDI 的关闭、以及 `bias` GPIO ✓。


### 0.26 suspend/resume 预检（尚未测试）

**已确认具备的条件：**

| 项 | 结论 |
|---|---|
| `CONFIG_PM` / `PM_SLEEP` / `SUSPEND` / `PM_SLEEP_SMP` / `SUSPEND_FREEZER` | **全部 y** ✓ |
| 原厂 DT 的挂起方式 | `suspend-method = "s2idle"` ✓ |
| DRM 注册了系统钩子 | `.suspend = mtk_drm_sys_suspend` / `.resume = mtk_drm_sys_resume` ✓ |
| `wk_lock` | **已分配** ✓ —— CRTC 初始化时 `wakeup_source_register(NULL, "disp_crtc0_wakelock")` ✓ |
| 面板的 disable/unprepare 调用链 | `mtk_dsi.c:3092 drm_panel_disable` / `3107 drm_panel_unprepare` ✓ —— **真实存在** ✓ |

**`mtk_drm_sys_suspend` 的行为**：先看 `mtk_crtc->wk_lock->active` ✓，
**false 就直接 `goto OUT`**（什么都不做）✓ ⇒ 面板不会睡 ⇒ 那条路径是"平凡成功" ✓。
只有为 true 时才走 `drm_atomic_helper_suspend` ✓（那才会 unprepare 面板 ✓）。

**尚未确定**：电源键是否被标为 `wakeup-source` ✗ —— DT 里有 `pwrkey`/`homekey`
中断 ✓ 和 `mtk-kpd` 配置 ✓，但**没有找到 `wakeup-source` 属性** ✓ ⇒
**s2idle 之后能否唤醒我无法保证** ✗ ⇒ 那个测试**可能需要长按电源键硬复位** ✓。

**因此第一步先做可恢复的那个（见 §0.27）。**


### 0.27 下一步：用 fb blank 触发面板 teardown/restore（完全可恢复）

`/sys/class/drm/card0-DSI-1/dpms` 是**只读**的 ✗（`DEVICE_ATTR_RO(dpms)` ✓），
但 `drm_fb_helper_blank()` 确实会设 DPMS ✓：

```c
	case FB_BLANK_UNBLANK:  drm_fb_helper_dpms(info, DRM_MODE_DPMS_ON);
	case FB_BLANK_NORMAL:   drm_fb_helper_dpms(info, DRM_MODE_DPMS_STANDBY);
	/* POWERDOWN -> DPMS_OFF */
```

并且 `CONFIG_DRM_FBDEV_EMULATION=y` ✓、`CONFIG_FB=y` ✓、`FRAMEBUFFER_CONSOLE=y` ✓。

⇒ `w /sys/class/graphics/fb0/blank 4`（DPMS_OFF ⇒ 面板 disable + unprepare + DDIC 睡眠 ✓）
再 `w /sys/class/graphics/fb0/blank 0`（⇒ prepare ⇒ `bl_bias_conf` + `bias_enable(1,1)` + 重放电平 ✓）。

**它走的是与系统挂起同一套面板代码 ✓，但系统不挂起 ⇒ 随时可重启恢复 ✓。**
判据：`dmesg lm36273` 里出现**第二次** `bl_bias_conf` + `bias_enable` ✓，且屏幕恢复 ✓。


### 0.28 **面板 teardown/restore 循环已实测通过**（suspend 的前提）

设备实测（`w /sys/class/graphics/fb0/blank 4` 再 `0`）：

```
[ 1.288390] lm36273_probe: i2c_lcd_bias
[ 1.288969] lm36273: chip at 0x11, REV=0x12
[12.661001] lm36273_bl_bias_conf backlight and bias setting     <- 首次 prepare
[12.661809] lm36273_bias_enable enable = 1
[47.630017] lm36273_brightness_set: bl level = 300  ...        <- 用户调亮度（6 次）
[107.185038] lm36273_brightness_set: bl level = 1023
[324.258869] lm36273_bl_bias_conf backlight and bias setting     <- 【第二次】prepare ✓
[324.258954] lm36273_bias_enable enable = 1                      <- AVDD/AVEE 重新使能 ✓
```

**⇒ `blank 4` 真的走到了面板的 disable + unprepare**（推 `lcm_suspend_setting` ✓），
**`blank 0` 真的走到了 prepare**（推 `lcm_init_setting` + `bl_bias_conf` + `bias_enable` ✓），
**且屏幕熄灭后正常亮起** ✓ —— 与日志逐条对应 ✓。

**⇒ 面板能被完整关掉并安全唤醒，这是 s2idle 测试的前提，现已成立。**

**一处我自己的可观测性缺口** ✗：`lm36273_bias_enable()` 里的**电平重放是静默的** ✓
（直接写寄存器、无打印 ✓）⇒ **"亮度在 prepare 后保持"这个说法在日志里看不到证据** ✓。
只能靠**肉眼**判断（熄灭前与亮起后的亮度是否一致 ✓）。下次有其它改动时应在重放处加一行
`pr_info`，否则 suspend 测试同样无法用日志判定重放是否发生 ✓。


### 0.29 下一步：s2idle 测试

`w /sys/power/state freeze`。已知条件见 §0.26。两种可能结果都能辨别：

| 结果 | 含义 |
|---|---|
| 挂起/唤醒正常，且日志**没有**第二次 `bl_bias_conf` | `wk_lock->active` 为 false ⇒ `goto OUT` ✓ 面板未被动过 ⇒ "平凡成功" ✓（s2idle 在跑，但没测到面板路径） |
| 挂起/唤醒正常，日志**有**第二次 `bl_bias_conf` | `drm_atomic_helper_suspend/resume` 真的跑了 ⇒ 面板随系统挂起并恢复 ✓ |

**风险**：电源键是否为 `wakeup-source` 未确认 ✗ ⇒ 最坏情况**长按电源键硬复位** ✓。
initramfs 环境无数据可失 ✓。


### 0.30 **s2idle 挂起/恢复成立**，但读数通道被挂起自己打断了

设备实测：

```
cat /sys/power/state     -> freeze mem disk
cat /sys/power/mem_sleep -> s2idle [deep]        （mem 的默认是 deep，用 freeze 才对）
w /sys/power/state freeze -> 串口先「脱机」再「连接」
cat /proc/uptime         -> 503.90 4122.84        <- 连续运行，不是重启
```

**⇒ `freeze`（s2idle）挂起并成功恢复** ✓ —— `uptime` 连续 ✓ 是最硬的判据（**没有重启** ✓）。

**但 `dmesg lm36273` 之后一行都打不出来** ✗，根因有两个，都在 `init` 里：

1. `usb_fd = open("/dev/ttyGS0", ...)` **只在启动时做一次** ✗，之后从不重开 ⇒
   **挂起复位了 USB gadget，那个 fd 就成了死 fd** ✗ ⇒ 之后所有输出都发不出去 ✓
   （写入失败只 `break`，fd 还留着 ✗）；
2. `usb_only = 1` 期间（命令执行时 ✓）**连 fd 1（`/dev/console` = tty0 = 屏幕）也不写** ✗
   ⇒ gadget 一死，**既看不到、也不进 ring** ✗。

**⇒ 后果：那次挂起期间的日志（面板有没有被 unprepare/prepare）已经无法取回** ✗ ——
内核消息本身还在 ring 里 ✓，但没有任何可用的通道把它读出来 ✓，而刷机必然重启 ✓。
**这是我这一侧的损失，记在这里。**

**修法**（v2ninetythird）：镜像写失败时**重开 `/dev/ttyGS0`**（单次尝试，不阻塞 ✓）；
`usb_only` 时若镜像**整体失败**，**回落到 fd 1** ✓ —— `/dev/console` 就是 `tty0`（屏幕 ✓），
⇒ **即使 USB 断了也能在屏幕上读到日志** ✓。

**顺带补上 §0.28 记下的可观测性缺口**：`lm36273_bias_enable()` 里的电平重放现在会打印
`lm36273: re-applied level N after bias enable` ✓ ⇒ suspend 测试可以直接用日志判定
"亮度是否被重放" ✓（之前只能靠肉眼 ✓）。


### 0.31 **suspend/resume 完整验证通过**（v2ninetythird）

```
[12.697149] lm36273_bl_bias_conf backlight and bias setting   <- 开机首次 prepare
[12.697959] lm36273_bias_enable enable = 1
[12.699728] lm36273: re-applied level 1023 after bias enable
[30.409646] lm36273_bl_bias_conf backlight and bias setting   <- 挂起恢复时【第二次】✓
[30.409731] lm36273_bias_enable enable = 1
[30.410967] lm36273: re-applied level 1023 after bias enable  <- 亮度被重放 ✓
```

一次挂起证实三件事：

1. **USB 镜像修复生效** ✓ —— 挂起**之后** `dmesg` 能读了（上一轮完全读不到 ✗）；
2. **面板真的随系统挂起并恢复** ✓ —— 30.41s 的第二次 `bl_bias_conf` 说明
   `mtk_drm_sys_suspend` → `drm_atomic_helper_suspend` **真的执行了** ✓
   （即 `wk_lock->active` 为 true ✓，是**非平凡**路径 ✓，不是 §0.30 里记的 `goto OUT`）；
3. **亮度跨挂起保持** ✓ —— `re-applied level 1023` ✓（§0.28 的盲点已由新打印关闭 ✓）。

**⇒ s2idle 的完整路径（含面板 teardown/restore 与电平重放）成立。**

**仍然是缺口的两点**（都属于"省电"而非"可用性"）：

- **挂起期间供电没有关** ✗ —— §0.21 起刻意没接 unprepare 的关闭路径
  （`lm36273_brightness_set(0)` + `lm36273_bias_enable(0, 3)` ✓、DVDD 关闭 ✓、
  `bias` GPIO ✓）⇒ **挂起功耗高于最优** ✓。现在恢复路径已被证明可用 ✓，
  接这条路径的风险从"可能黑屏且无证据"降到"可测" ✓。
- **`mem`（deep）未测** ✗ —— `mem_sleep` 的默认是 `[deep]` ✓，而原厂 DT 明写
  `suspend-method = "s2idle"` ✓ ⇒ s2idle 才是设计目标 ✓，`deep` 非必需 ✓。

**接关闭路径的建议顺序**：先在 **fb blank**（§0.27，完全可恢复 ✓）上验证关闭+唤醒 ✓，
再上系统挂起 ✓ —— 这样万一芯片唤不醒，损失只是一次重启 ✓。


### 0.32 挂起时的关闭路径（省电）

原厂的分工（`panel-k10a-36-02-0a-dsc-vdo.c`）：

```c
lcm_disable:    if (ctx->backlight) props.power = FB_BLANK_POWERDOWN;   /* 走 backlight class */
lcm_unprepare:  push_table(lcm_suspend_setting);   /* DDIC 睡眠 */
                reset_gpio = 0;
                bias_pos = 0;                      /* bias GPIO 0 */
                lm36273_brightness_set(0);         /* 背光关 */
                lm36273_bias_enable(0, 3);         /* AVDD/AVEE 关 */
                lcm_panel_dvdd_disable(dev);       /* DVDD 关 */
                usleep(4~5ms);
                lcm_panel_vm18_disable(dev);       /* VDDI（本板无）*/
```

**核对后发现两件事：**

1. **DVDD 其实早就在关了** ✓ —— `lcm_unprepare` 里已有 `lcm_panel_vddi_disable` ✓，而那条"第一次 disable 拒绝"的
   guard 在**开机那次模式切换**时就被消耗掉了 ✓ ⇒ 后续的 unprepare（含挂起）会真的关掉 DVDD ✓。
   我先前以为它还挡着，是错的。
2. **真正缺的只有两句** ✗ —— 背光与 AVSS/AVEE 的关闭：
   `ktz8863a_brightness_off()` / `ktz8863a_bias_enable(0)` 接的是**从不 probe** 的芯片 ✗
   ⇒ 换成 `lm36273_brightness_set(0)` / `lm36273_bias_enable(0, 3)` ✓。

**并且给面板节点补上了 `backlight` phandle** ✓（`&{/i2c6@11F01000/i2c_lcd_bias@11}` ✓）——
**这是关键**：`lm36273_brightness_set(0)` 会把芯片驱动记住的电平清成 0 ✗，
而 `lcm_prepare()` 里那段"重新断言背光"（`props.power = FB_BLANK_UNBLANK` +
`backlight_update_status()` ✓）能从 **`props.brightness`**（用户设定值 ✓，0 写入不影响它 ✓）恢复 ✓。
没有这个 phandle，`ctx->backlight` 是 NULL ⇒ 那段是空操作 ⇒ **挂起后屏幕不会亮** ✗。

**⇒ 这也修正了我 §0.21 的判断** ✓：当时"不加 phandle"是对的（那时**不想要**关闭 ✓），
现在**想要**关闭了，phandle 就从"危险"变成"必需" ✓。

**校验**：DTB 里 `backlight` = `0x271` → `i2c_lcd_bias@11` ✓、`dvdd-supply` = `0x1f3` → `wl2866d1@1` ✓
（dtc 会拒绝未解析的路径 ✓，但我确认了它指向**对的**节点 ✓）。

**未做**：`bias` GPIO（原厂注释写 `GPIO126/bias0` ✓，编号未经核实 ✗）—— i2c 侧的
`bias_enable(0, 3)` 才是功能性关闭 ✓，先不驱动一个可能错的引脚 ✓。

**测试顺序**（保持可恢复 ✓）：先在 **fb blank** 上验证关闭 + 唤醒 ✓ —— 万一唤不醒，损失只是一次重启 ✓；
通过后再上系统挂起 ✓。


### 0.33 **关闭路径与电平保持经实测验证**（v2ninetyfourth）

```
[36.797261] bl level = 300                      <- 用户设定
[46.705749] bl level = 0 / close                <- blank 4：背光关 ✓
[46.850854] bias_enable enable = 0              <- AVDD/AVEE 关 ✓
[64.391588] bl_bias_conf backlight and bias setting   <- blank 0：重新配置 ✓
[64.391667] bias_enable enable = 1              <- AVDD/AVEE 开 ✓
[64.535051] bl level = 300                      <- 【恢复 300，不是默认 1023】✓
[64.535130] enable level = 300                  <- 背光在 300 上重开 ✓
```

**⇒ 关闭真的发生 ✓、恢复完整 ✓、且恢复的是用户设定的 300** ✓ —— 这正是 §0.32 里
`backlight` phandle + `lcm_prepare()` 重断言块的设计目标 ✓（没有 phandle 就会回落到 1023 ✓）。

**同时记下一个我核对时纠正的错误** ✓：DVDD 的关闭**早已就位**（`lcm_unprepare` 里的
`lcm_panel_vddi_disable` ✓），那条 guard 在**开机那次模式切换**时就被消耗掉了 ✓ ——
我先前以为它还挡着 ✓。（详见 §0.32 ✓）

**补上可观测性** ✓（v2ninetyfifth）：WL2866D 驱动的 enable/disable 现在各自打印
`%s: on` / `%s: off` ✓ —— 在此之前 **DVDD 是否真的关掉只能靠读代码推断** ✗，
而"读代码推断"正是我把那条 guard 判断错的原因 ✓。


### 0.34 下一步：系统挂起（省电的真正验证）

fb blank 已验证关闭 + 唤醒 ✓ ⇒ 现在可以做 `w /sys/power/state freeze` ✓。

**判据**（这次日志会说话 ✓）：

| 期望出现 | 含义 |
|---|---|
| `lm36273_bias_enable enable = 0` + `%s: off`（dvdd2） | 挂起时供电与背光真的切断 ✓ |
| 恢复后 `bl_bias_conf` + `bias_enable = 1` + `%s: on` + `bl level = <用户值>` | 完整恢复且亮度保持 ✓ |

**风险**：电源键是否为 `wakeup-source` 仍未确认 ✗ ⇒ 最坏情况长按电源键硬复位 ✓。
（上一次挂起测试既已成功恢复 ✓，这个风险实际很低 ✓。）


### 0.35 **挂起省电闭环：供电 / AVDD/AVEE / 背光全部真的切断并恢复**（v2ninetyfifth）

设备实测 —— `dmesg wl2866d`：

```
[11.860029] wl2866d-dvdd2: on                        <- 面板 probe 时打开
[12.620357] init phase: keeping wl2866d-dvdd2 enabled <- guard 触发（开机那次模式切换）
[21.047886] wl2866d-dvdd2: off                       <- 挂起真的切断了 DVDD ✓
[21.152381] wl2866d-dvdd2: on                        <- 恢复又打开了 ✓
```

`dmesg lm36273`：

```
[14.279918] bl level = 300                    <- 用户设定
[20.901844] bl level = 0 / close              <- 挂起：背光关 ✓
[21.042656] bias_enable enable = 0            <- 挂起：AVDD/AVEE 关 ✓
[21.154412] bl_bias_conf ...                  <- 恢复：重新配置 ✓
[21.154489] bias_enable enable = 1            <- 恢复：AVDD/AVEE 开 ✓
[21.298863] bl level = 300                    <- 恢复：亮度回到 300 ✓
[21.298942] enable level = 300                <- 恢复：背光在 300 上重开 ✓
```

现象与日志一致：**自动黑屏 → 亮起** ✓。

**⇒ 供电（DVDD）✓、AVDD/AVEE ✓、背光 ✓ 三者在挂起时全部真的切断、恢复时全部真的回来，
且亮度保持用户设定值** ✓ —— §0.32 接的关闭路径与 §0.33 验证的电平保持共同构成闭环 ✓。

**这段日志同时确认了 §0.32 的更正** ✓：那条 guard 确实在**开机时**（12.62s）就消耗掉了 ✓，
所以挂起（21.05s）时能走到真正的关闭 ✓。**这是"读日志"而不是"读代码"才得到的确证** ✓。


---

# 附录 B：早期阶段快照（v2fiftyfirst 时期）

> 下面是本文档早期的顶层章节（当时的"结论摘要 / 里程碑 / 文件地图 / 流程 / 镜像清单"）。
> **它们描述的是早期状态，部分结论已经过时** —— 例如"唯一剩余隐患是 120 Hz porches"
> 和"面板 probe 返回 -12"都已经解决。保留是为了记录当时的判断与流程。
> **当前状态请看开头的"接手先读"。**
>
> 附录 B 原本的文档头（现已被"接手先读"取代，保留于此处）：
>
> > 写于 v2 第五十一版（`chopinport=v2fiftyfirst`）。本文档面向"新会话接手"：把当前进度、剩余故障点、
> > 精确落点、构建流程、镜像清单与所有已踩过的坑一次性交代清楚。

## 1. 结论摘要（一句话）

**v2（vendor `mediatek_v2` 栈）已经在 chopin 上完整驱动这块面板，而且可以从提交源码复现。**
12 个显示组件全部 bound（含 DSC wrap）、CRTC 与 connector 建立、`/dev/dri/card0` 存在、
DSI 时钟与 MIPI PLL 在跑、DSC 寄存器与 LK 黄金值逐字节吻合、
`fbcon → mediatekdrmfb` 正在扫描输出，屏幕正常显示控制台；从零重建的镜像刷机后同样上屏。

剩下的**唯一实质性隐患**是 **§0.11**：connector 上报 120 Hz，但模式里的 porches
没有到达 DSI（`HFP_WC` 恒为 `0x5a0`），即**时序是 LK 遗留的、驱动尚未接管** ——
显示无碍，但重新初始化（休眠唤醒/模式切换/ESD 恢复）会暴露。
其它收尾项见 §0.4/§0.12（原厂初始化序列、背光、删插桩、port 仓库整理）。

---

## 2. 已验证达成的里程碑（均有日志证据）

| # | 里程碑 | 证据（日志片段） |
|---|---|---|
| 1 | v2 整个栈编译/链接/加载/注册 | `CHOPIN-DRM: registering mediatek-drm/ddp/…/mtk-dsi`，`mtk_drm_init done` |
| 2 | MIPI-TX PHY 绑定 | `mediatek-mipi-tx 11e50000.mipi_tx_config: CHOPIN-MIPITX: regs ok / refclk ok / phy ok` |
| 3 | SMI common + larb0/larb1 绑定 | `mtk-smi-common 14002000.disp_smi_common: mtk-smi-common probed, commid=0`；`CHOPIN-DBG: larb probe returning 0 (larbid=0/1)` |
| 4 | **12 个显示组件全部 bound** | `bound 14000000/14001000.disp_ovl, 14003000.disp_rdma, 14006000.disp_wdma, 14007000.disp_color, 14008000.disp_ccorr, 14009000.disp_aal, 1400a000.disp_gamma, 1400b000.disp_dither, 1400d000.disp_postmask, 1400e000.dsi` |
| 5 | master 注册成功 | `CHOPIN-STAGE probe: comps collected, master_add next` → `CHOPIN-STAGE: master_add ret=0` |
| 6 | **CRTC 创建成功** | `[drm] Initialized mediatek 0.0.0 for mediatek-drm.7.auto on minor 0`（已无 `Not creating crtc …`） |
| 7 | connector 已使能 | `status: connected` / `enabled: enabled` / `dpms: On` |
| 8 | **DRM 设备存在** | `ls /dev/dri` → `card0` |
| 9 | DSI 时钟 + MIPI PLL 运行 | `mm_dsi0_mm_clk=1`、`mm_dsi0_intf_clk=1`、`mipi_tx0_pll` 引用计数 1 且绑定到 `1400e000.dsi` |
| 10 | 显示设备只归 DRM | simplefb 已关闭（`# CONFIG_FB_SIMPLE is not set`） |
| 11 | DSI 容错（面板未注册也照常注册） | `mtk-dsi 1400e000.dsi: panel/bridge not up yet, continuing` + `bound 1400e000.dsi` |
| 12 | mi_disp 胶水正常 | `mtk_dsi_create_conn_enc init mtk ealry resume resources`、`[mi_disp:mi_disp_procfs_init] [info]create procfs mipi_rw_prim success!` |

**v2fiftieth 时尚未达成**（v2fiftyfirst 已修掉前两项，第三项仍未解决）：

- ~~面板驱动注册成功~~ ⇒ **v2fiftyfirst 已修**（§0.1/§0.2），待刷机确认；
- ~~connector 只有 generic modes（1024x768…）~~ ⇒ 面板注册后应出现 1080×**2460**（本版仍是 l16 参数）；
- **DSC 编码器没有进管线**（§0.3）⇒ 这是现在挡在"真正点亮"前面的那一项。

---

## 3. ~~唯一剩余故障点：面板 probe 返回 `-ENOMEM (-12)`~~ —— 已在 v2fiftyfirst 修复

> **结论：根因是漏掉的花括号，见 §0.1。** 本节保留为历史记录（它准确描述了当时的观测，
> 但 §3.2 的三条"排除"结论是**基于错误前提**的：那些插桩日志打不出来恰恰是因为代码在
> 它们之前就无条件 `return` 了，而不是因为"走不到那里"）。§3.3 的插桩建议**不需要再做**。

### 3.1 现象（v2fiftieth 实测）

```
<3> mtk-dsi 1400e000.dsi: CHOPIN-DSIDBG: probe enter
<3> l16_36_02_0b_dsc_vdo,lcm 1400e000.dsi.0: CHOPIN-PANELDBG: probe enter
<3> l16_36_02_0b_dsc_vdo,lcm 1400e000.dsi.0: probe with driver l16_36_02_0b_dsc_vdo,lcm failed with error -12
<6> mtk-dsi 1400e000.dsi: panel/bridge not up yet, continuing
<3> CHOPIN-DBG[dsi_probe] panel=0000000000000000 ext=0000000000000000
```

### 3.2 已经排除的范围（重要，别重复走）

| 已排除 | 依据 |
|---|---|
| 不是 DT 匹配问题 | `probe enter` 打出来了 ⇒ `of_device_id` 命中 ✓ |
| 不是面板驱动自己的 GPI/资源报错 | 驱动内 `dev_err("…cannot get reset-gpios…")` 一条都没出现 |
| 不是 `return -ENOMEM` 在 probe 里 | 我在 probe 的 7 个 `return` 前插的 `return at line N` 一条都没打出来 |
| **不是 `devm_kzalloc(sizeof(struct lcm))`** | 新加的 `ctx=… sizeof(lcm)=…` 日志**没有打印** ⇒ 根本没走到那行 |
| 不是 mi_disp / DSI 侧 | `mi_disp_procfs_init success`、DSI 后续注册正常 |

⇒ **故障点被夹在 `lcm_probe()` 里 `probe enter` 与 `devm_kzalloc` 之间的 3 行 of_graph 代码**，
或来自 mipi-dsi 核心对 probe 的封装。

### 3.3 下一步（30 秒的改动，一次刷机定死）

在 `linux-7.2.8/drivers/gpu/drm/panel/panel-k10a-bringup.c` 的 `lcm_probe()` 开头附近，
按顺序插入：

```c
dev_err(dev, "PANELDBG: after of_get_parent\n");
endpoint = of_graph_get_next_endpoint(dsi_node, NULL);
dev_err(dev, "PANELDBG: after get_next_endpoint endpoint=%pOF\n", endpoint);
remote_node = of_graph_get_remote_port_parent(endpoint);
dev_err(dev, "PANELDBG: after get_remote_port_parent remote=%pOF\n", remote_node);
```

判读：
- 第一条都不出 ⇒ 失败在 `of_get_parent` 之前（看是不是函数入口的另一段代码 / mipi-dsi 封装）；
- 出到第二条为止 ⇒ `of_graph_get_next_endpoint` 相关；
- 三条都出、仍 -12 ⇒ 来自 `drm_panel_init`/`drm_panel_add` 之前的其它调用 ⇒ 继续往下插桩。

### 3.4 修好之后的下一步：换 k10a 真面板参数

当前"占位面板"用的是参考树 **l16** 的驱动与参数：

| 项目 | 现状（占位） | 应改为（k10a 真实值） |
|---|---|---|
| compatible | `l16_36_02_0b_dsc_vdo,lcm` | 可保持（驱动是我们自己的），但参数要换 |
| 分辨率 | 1080×**2460** ✗ | **1080×2400** ✓ |
| 初始化序列 | l16 的 | 原厂 `n10p/vendor/kernel_mt6893-chopin_kvm/drivers/gpu/drm/panel/panel-k10a-36-02-0a-dsc-vdo.c` |
| DSC PPS | l16 的 | 原厂 k10a 的 + 与 `port/DSC-REFERENCE.md`（LK 实测黄金值）对账 |

参考模板：`port/mt6895-linux/drivers/gpu/drm/panel/panel-l16-36-02-0b-dsc-vdo.c`。

---

## 4. 关键文件地图

### 4.1 内核侧（工作树 `linux-7.2.8/`，镜像源 `git/chopin-linux-7.2.8/`）

| 文件 | 作用 / 本轮改动 |
|---|---|
| `arch/arm64/kernel/chopin-board.dts` | **实时 DT**（内嵌板级树）。显示相关：dispsys `compatible = "mediatek,mt6885-mmsys","mediatek,mt6893-disp","mediatek,mt6873-mmsys","mediatek,mt6893-mmsys","syscon"`；`dispsys_num = <1>`；`clock-num`；`smi_larb0/1`、`disp_smi_common`；10 个显示组件各自 `mediatek,larb` + `mediatek,smi-id`；`disp_wdma@14006000` 已启用；23 个 MT6895-only 节点 disabled；`panel@0` → `l16_36_02_0b_dsc_vdo,lcm` + `status = "okay"` |
| `arch/arm64/configs/chopin_defconfig` | `CONFIG_DRM_MEDIATEK_V2=y`、`CONFIG_MTK_MML=y`、`CONFIG_DRM_PANEL_K10A_36_02_0A=y`、`CONFIG_BACKLIGHT_CLASS_DEVICE=y`、`# CONFIG_PHY_MTK_MIPI_DSI is not set`、`# CONFIG_MTK_SMI is not set`、`# CONFIG_FB_SIMPLE is not set`；cmdline 末尾 `chopinport=v2fiftieth`（每版递增，用于确认刷的是哪一版） |
| `drivers/gpu/drm/mediatek/mediatek_v2/` | MT6895 移植来的 vendor 栈（133 文件，逐文件与参考树一致 + 我们的 Makefile flags）。本轮改动：`mtk_drm_drv.c`（MT6885 pipe 表删 `DMDP_AAL0/AAL1`；top-clk 数量按节点实际裁剪；master_add 返回值日志）、`mtk_drm_crtc.c`（`first_enable` 无 mode 时跳过）、`mtk_dsi.c`（**面板缺失不再 defer**，`panel/bridge not up yet, continuing`） |
| `drivers/memory/mtk-smi-downstream.c` | 下游 SMI（common + larb），驱动名改 `mtk-smi-common-legacy` / `mtk-smi-larb-legacy`；larb 的 `component_add` defer 被吞掉；`larb probe returning %d` 诊断日志 |
| `drivers/memory/mtk-smi.c` | `mtk_smi_init()` 在 `CONFIG_DRM_MEDIATEK_V2` 下直接 `return 0`（让下游 SMI 拥有名字） |
| `drivers/soc/mediatek/mtk-mmsys.c` | 用 `platform_device_alloc` + `device_set_node(of_find_compatible_node(…,"mediatek,mt6893-disp"))` + `platform_device_add_resources` 创建 v2 master |
| `drivers/gpu/drm/panel/panel-k10a-bringup.c` | **占位面板**（源自 l16 驱动），已插桩 `CHOPIN-PANELDBG`。v2fiftyfirst：修掉两处漏花括号（-12 根因）、`leden`/`dvdd`/`cam` 改 optional、`vrf18` 缺失时清空指针。**参数仍是 l16 的**（1080×2460），下一版换 k10a |
| `drivers/gpu/drm/panel/leds-ktz8863a.c`, `android_bl_map.h` | 面板背光及其依赖头 |
| `drivers/gpu/drm/panel/Makefile` | 追加了 v2/mi_disp 与厂商目录的 include 路径 + 两个 obj |
| `drivers/gpu/drm/mediatek/Makefile` / `Kconfig` | 采用参考树版本（按 `CONFIG_DRM_MEDIATEK_V2` 选择 v2 栈 vs 主线 mediatek-drm） |

### 4.2 port 侧（文档 / 工具 / 产物）

| 路径 | 内容 |
|---|---|
| `port/V2-IMPORT.md` | 导入过程的运行日志（历史） |
| `port/DSC-REFERENCE.md` | LK 实测 DSC 黄金值（PS_SEL=5、PS_WC=1080、SIZE_CON=0x09600168、路由 0xf50=8/0xf54=3/0xf80=1、mutex DSC0=BIT(29) 等） |
| `port/CLEANUP-PLAN.md` | **出图后**要做的 git 历史整理计划（tag `chopin-v2-working` 后重写为约 7 个逻辑提交） |
| `port/DELIVERY.md`、`port/NEXT-DRM.md`、`port/HANDOFF.md`、`port/SUMMARY.md` | 早期（主线 DRM 阶段）的交付/计划文档，部分已过时 |
| `port/tools/build-images.sh` | 打包 `port/out/boot-stockdt.img`（stock DT + 我们的内核 + initramfs） |
| `port/tools/verify-drm.sh` | 本地自检（**每次发版要更新其中硬编码的镜像名**；必须输出 "all local checks passed"） |
| `port/tools/import-mediatek-v2.sh` / `v2-copy-from-mt6895.sh` / `modernize-v2.py` / `v2-stage-headers.sh` | v2 导入工具链 |
| `port/dist/` | 121 个历史镜像 + `SHA256SUMS` |

---

## 5. 每一轮的构建 / 打包 / 提交流程（照抄即可）

```bash
cd /home/kily/project/mt6893

# 1) 改代码后递增版本号
sed -i 's/chopinport=OLD/chopinport=NEW/' linux-7.2.8/arch/arm64/configs/chopin_defconfig

# 2) 构建（必须 build=0）
cd linux-7.2.8
make O=/home/kily/project/mt6893/build/chopin ARCH=arm64 \
     CROSS_COMPILE=aarch64-linux-gnu- chopin_defconfig
make O=/home/kily/project/mt6893/build/chopin ARCH=arm64 \
     CROSS_COMPILE=aarch64-linux-gnu- -j16 Image > ../port/audit/v2-build.log 2>&1
echo "build=$?"          # 必须是 0，再打包！

# 3) 打包 + 校验和
cd ../port && ./tools/build-images.sh 2>&1 | grep 'boot-stockdt.img:'
cd ..
cp port/out/boot-stockdt.img port/dist/chopin-<NAME>-stockdt.img
sha256sum port/dist/chopin-<NAME>-stockdt.img | sed 's|port/dist/||' | tee -a port/dist/SHA256SUMS

# 4) 同步进 git 镜像仓库并提交
cp -a linux-7.2.8/<改动的文件或目录> git/chopin-linux-7.2.8/<对应路径>
git -C git/chopin-linux-7.2.8 add -A && git -C git/chopin-linux-7.2.8 commit -m "…"
git -C port add -A && git -C port commit -m "port: chopin-<NAME>-stockdt.img (…)"

# 5) 更新 port/tools/verify-drm.sh 里的镜像名，然后
./port/tools/verify-drm.sh      # 必须 "all local checks passed"
```

> 注意：`/tmp` 不可写 ⇒ 日志放 `port/audit/`。打包前务必确认 `build=0`，
> 否则会出现"把旧 `port/out/boot-stockdt.img` 换个名字发布"的事故（曾发生过一次）。

---

## 6. 镜像清单与回退

| 分支 | 用途 | 镜像 | sha256 |
|---|---|---|---|
| **dev** | **v2seventieth —— 补 `MMQOS_SUPPORT`/`MMDVFS_SUPPORT`（针对 OVL underflow 的修复，待刷机）** | `port/dist/chopin-v2seventieth-stockdt.img` | `1660563251a1d346af6020841922dcd7a1439c4a43783b349da4f2246b09c8d3` |
| dev | v2sixtyninth —— 换成原厂 k10a 初始化序列（**无行为变化，不必刷**） | `port/dist/chopin-v2sixtyninth-stockdt.img` | `a077a1dc77e911970c1e478645635bcf577cd862c979c68bef5a211bfe6b063c` |
| dev | v2sixtyeighth —— EOT + `vfp_low_power` 修正（仍花屏） | `port/dist/chopin-v2sixtyeighth-stockdt.img` | `7d07a8967e68403aa749f64ba385334a64fa439eb71a4b57f2bb902110e3d995` |
| dev | v2sixtyseventh —— DDIC 帧率映射 + adopt 两半一起改（仍花屏） | `port/dist/chopin-v2sixtyseventh-stockdt.img` | `a2e5ee2fb5fbd6a44d6b9d2d3c56b17fca153d39659259fb93ce9d48e9147e6c` |
| dev | v2sixtysixth —— master + 日志，画面可用（已知good） | `port/dist/chopin-v2sixtysixth-stockdt.img` | `38ef4f9878d4135153054d940948b9cd0cbe7a67474c69fc4ffb142bea20c04e` |
| dev | v2sixtyfourth —— 四个埋点，已定出 §0.11 的根因（画面可用） | `port/dist/chopin-v2sixtyfourth-stockdt.img` | `40858ef4a182e6abf0b36bcba148e82ba305054dcbb2bbcd4f2ec9245a32dedd` |
| — | v2sixtyfifth —— **别刷**：补 DSI 时序的实验，帧率确实翻倍但**花屏**（见 §0.11.3） | `port/dist/chopin-v2sixtyfifth-stockdt.img` | `1c19ebe6f204317213effd63004429b3d76cefc0bdee39bfec3c106090667cbe` |
| **master** | **v2sixtythird —— 已刷机验证上屏**（dmesg 过滤器修好，能搜整个 4 MiB 环） | `port/dist/chopin-v2sixtythird-stockdt.img` | `f8520af5d5fc6c751808aacca1cb2f720309c1f91c2b164f1fd5dd157cbe14e5` |
| dev→master | v2sixtysecond（120 Hz 设为 preferred；144 Hz 占位已删） | `port/dist/chopin-v2sixtysecond-stockdt.img` | `44945d99a49ce807121262f52f55fd46b250185cae59ee25f3c64611bd9e4a0b` |
| master | v2sixtyfirst（derive mode clock；vrefresh 从 81 修正为 60） | `port/dist/chopin-v2sixtyfirst-stockdt.img` | `2fe9b05ae166f3d9071c0232d6f15b6403abd10ecd1373f5b5b5413321b8f338` |
| — | v2fiftysixth（单 pipe + drm 工具重排） | `port/dist/chopin-v2fiftysixth-stockdt.img` | `8490fce9b2dad590f42b46cc62deb4a673aca2ebedd15e610305f3d5c43a5b96` |
| — | v2fiftyfifth（k10a 真参数 + 厂商 DSC PPS，`is_dual_pipe=1` 卡死，**别刷**） | `port/dist/chopin-v2fiftyfifth-stockdt.img` | `a1c2af243cf5c9153fb37140e654c9d402cd4f6f30f2309b4bd45962495fbaa7` |
| — | v2fiftyfourth（**通路打通**：connector 出现 1080×2400×7） | `port/dist/chopin-v2fiftyfourth-stockdt.img` | `12d6caa7d760d4d086a4fbd771ad3bc323a26a1a6222d170e4d3cf837381b7f2` |
| — | v2fiftythird（面板被放到 port@1 ⇒ 被当成 slave，**别刷**） | `port/dist/chopin-v2fiftythird-stockdt.img` | `0f9d6a353705120776265729ba965a8f81a1ecbe1ce6b46dc5fc4d1430cafd1f` |
| — | v2fiftysecond（DSI 找错面板 ⇒ 卡死，**别刷**） | `port/dist/chopin-v2fiftysecond-stockdt.img` | `694aae3bd591413c668307bb0e4d50ba774d665ceff25c196e9248c9396e524a` |
| — | v2fiftyfirst（面板注册成功，但 `drm` 会卡死，**别刷**） | `port/dist/chopin-v2fiftyfirst-stockdt.img` | `04fe5a9ae69479963eb980267e78cc98d354c6dd31c4c79b246fb62303808b27` |
| — | v2fiftieth（面板 probe -12，已知状态） | `port/dist/chopin-v2fiftieth-stockdt.img` | `115e31995b56bdff384566201e6a91ca00c6f2fe5de854c36309969a0059b219` |
| — | v2fortyfifth（CRTC 建立 + connector enabled + card0，**未上屏**） | `port/dist/chopin-v2fortyfifth-stockdt.img` | `405d1da3670757eda59768dc89c2d13126cc3847299067dc457455600d037e97` |
| — | ringfix88-livemirror（LK 流水线 + adopt + 帧镜像，**非 DRM**，可用回退） | `port/dist/chopin-ringfix88-livemirror-stockdt.img` | `3b1132038717390f118e063a79dfbdab9bce1554fe2bee482f42c574eb207b5e` |

`port/dist/*.img` 被 `.gitignore` 忽略（只跟踪 `SHA256SUMS`），镜像本体只在本地磁盘。

> **v2sixtyfourth（dev）要看的**：刷机后 `dmesg CHOPIN-DSI` 应给出四条链路证据 ——
> `[mode_set] in: clock=331912 ... => vm(hfp=20 ...)`（模式确实带 20 像素 HFP 进来）、
> `[calc] #N hop=? vm(hfp=?) ... => hfp_byte=?`（算出来是 48 还是 1440）、
> `[write] wrote HSA/HBP/HFP=... ; read back HSA/HBP/HFP=0x?/0x?/0x?`（写进去的与读回来的是否一致）、
> `[hopping] mipi_hopping_sta <= ?`（若为 1，`t_hfp` 会改取 `dyn->hfp` 而不是 `vm->hfront_porch`）。
> 这四条足以判定 §0.11 的根因。

刷机：

```bash
fastboot flash boot_a port/dist/<镜像名>.img
fastboot reboot
```

机内自检命令（initramfs 里的自定义命令；**`dmesg` 只接受一个过滤词，不能用管道**）：

```bash
chopin# dmesg chopinport        # 确认刷的是哪一版（cmdline 里的 tag）
chopin# dmesg CHOPIN            # 我们加的里程碑日志
chopin# dmesg CHOPIN-PANELDBG   # 面板 probe 插桩
chopin# dmesg larb / defer / probe / smi / dsi
chopin# drm                     # connector / atomic / 时钟 / 中断 / 寄存器 总览
chopin# ls /dev/dri             # card0
```

---

## 7. 本阶段已解决的坑（避免新会话重复）

1. **驱动名冲突**：`mediatek-mipi-tx`（⇒ 关 `CONFIG_PHY_MTK_MIPI_DSI`）、`mtk-smi-common`/`mtk-smi-larb`（⇒ 改名 `-legacy`，最终在 v2 下跳过主线 SMI 注册）。
2. **v2 master 需要 fwnode**：`of_device_get_match_data` 为 NULL 会 oops ⇒ 用 `of_find_compatible_node("mediatek,mt6893-disp")` + `device_set_node` + 资源传递。
3. **`dispsys_num` / ioremap**：需要 `dispsys_num = <1>` 与 `platform_device_add_resources`；`-EBUSY 0x14116000` ⇒ v2 栈改用 `devm_ioremap()`。
4. **下游 SMI 的时钟名**：larb 需要 `"apb"`/`"smi"`（`"gals"` 可缺）；common 需要 `"apb"`/`"smi"`/`"async"`/`"gals0"`/`"gals1"`。**必须读驱动源码，不要猜属性**（这一点曾浪费 3 次刷机）。
5. **`mediatek,larb` + `mediatek,smi-id`**：v2 的 `mtk_drm_ddp_comp.c:641` 读 `mediatek,smi-id`，缺了会 `read smi-id failed:-22`。注意 `need larb device` 只是**诊断 warn**（同文件 :653），不是错误。
6. **`component_add` 的 -EPROBE_DEFER 互等**：v2 master 在自己 probe 末尾才 `component_master_add_with_match()`；组件/larb 的 `component_add` 会 defer ⇒ 互相等 ⇒ 超时。已在 larb 与 25 个组件驱动里吞掉该 defer。
7. **CRTC 组件表**：`mt6885_mtk_ddp_main` 里的 `DMDP_AAL0/AAL1` 本板没有 ⇒ 会 `Not creating crtc 0`。**必须在数组里删掉**；运行时 `continue` 跳过会崩在 `mtk_drm_crtc_create`。
8. **`disp_wdma@14006000`（comp 65）**：本板有 v2 驱动 ⇒ 在 DT 里启用即可（不能靠"删除"）。
9. **`first_enable` 在无 mode 时会卡死**：面板注册晚于 first_enable ⇒ `display size error(0x0)` + 控制台卡住 ⇒ 已在 `mtk_drm_crtc_first_enable()` 加"无 mode 直接返回"。
10. **top-clk 报错非致命**：`mtk_drm_get_top_clk get 0 clk failed` → `top_clk_num=-1`，而 `mtk_drm_top_clk_prepare_enable()` 开头 `<=0 return` ⇒ 不是阻塞项（已按节点实际时钟数裁剪）。
11. **面板的 of_graph 检查**：原驱动要求 endpoint remote 指回面板自身，我们的 DT 指向 DSI ⇒ 会 `return -ENODEV`；已放宽为继续。
12. **simplefb vs DRM**：`Console: switching to colour dummy device` 是 DRM 接管 fbdev 的正常现象；已按用户建议关闭 `CONFIG_FB_SIMPLE`，让显示只归 DRM。
13. **插桩时最容易自伤的一类改动**：给 `if (!x) return -Exxx;` 插日志时**一定补大括号**。
    v2fiftieth 就是在这上面卡住的（§0.1）：漏掉大括号后 `return` 变成无条件执行，
    而插进去的日志因为被 `if` 守住反而不打，于是"日志没打"被误读成"没走到那里"，
    把排查方向整个带偏到 of_graph / mipi-dsi 封装上。**日志打不出来时，先确认那行是不是死代码。**
14. **DT 属性"存在"≠"能解析"**：`panel@0` 的 `reset-gpios = <&{/pinctrl} 42 …>` 一度被怀疑
    因为 `&{/pinctrl}` 不是 GPIO 控制器而无效。**直接反编译编译产物验证**
    （`dtc -I dtb -O dts build/chopin/arch/arm64/kernel/chopin-board.dtb`）：
    该节点确实带 `gpio-controller` / `#gpio-cells = <0x02>`（来自 stock 基树，overlay 只改
    compatible/reg，不做替换），所以它能解析。**别靠读 dts 源码猜，读编译出来的 dtb。**
15. **`regulator_get()` 失败后一定要把指针清掉**：本板 `CONFIG_REGULATOR_DUMMY` 未开，
    缺失 supply 返回的是 `ERR_PTR(-ENODEV)` 而不是 NULL。任何"先 init、后无条件使用"的
    静态指针（这里是 `disp_vddi`）都会在第一次 modeset 时把它喂给
    `regulator_set_voltage()` → 解引用崩溃。

---

## 8. 建议的接手顺序（新会话第一小时）

1. **先刷 `chopin-v2fortyfifth-stockdt.img` 或 `ringfix88`**，确保手上有可用状态；
2. ~~按 §3.3 给 `lcm_probe()` 的 3 行 of_graph 插桩 ⇒ 一次刷机定死面板 -12~~ **已完成，不需要再做**（§0.1）；
3. ~~修掉面板注册~~ **已修**（v2fiftyfirst）。刷它确认：`drm_panel_add done`、DSI 不再 `panel/bridge not up yet`、
   `dmesg defer` 不再列 `1400e000.dsi`、connector 出现面板模式（本版是 1080×**2460** 的 l16 参数）；
4. ~~下一版（v2fiftysecond）：把 DSC 接进主通路 + 换 k10a 真参数~~
   **DSC 那一半已完成**（v2fiftysecond，且比原设想简单：只改 DT compatible，见 §0.5）：
   ~~给 `mtk_disp_dsc.c` 的 `of_match` 与 `mtk_drm_drv.c` 的组件匹配表加 `mt6893-disp-dsc`~~
   —— **不要那样做**，`mediatek,mt6893-disp-dsc` 本来就是个不存在的名字；
   ~~让 `mt6885_mtk_ddp_main[]` 带上 `DDP_COMPONENT_DSC0`~~ —— **不需要**，
   `mtk_drm_crtc.c` 按 `MMSYS_MT6885` 在运行时插入。
   剩下的是 **k10a 真参数**（§0.4 的五项 + 原厂初始化序列 + DSC PPS，与 `port/DSC-REFERENCE.md` 对账）；
5. 出图后执行 `port/CLEANUP-PLAN.md`（打 tag `chopin-v2-working`，把这一长串迭代提交整理成约 7 个逻辑提交，并校验 tree 一致），同时清理临时插桩（`CHOPIN-DBG`/`CHOPIN-PANELDBG`/`CHOPIN-COMPDBG`/`CHOPIN-STAGE`/`CHOPIN-OVLDBG`/`CHOPIN-DSIDBG`）与 `mtk_smi_stub.c` 残留。

---

## 9. 当前 git 状态

- 内核镜像仓库 `git/chopin-linux-7.2.8`：分支约定见 §0.14 ——
  **`master`** 是已刷机验证的干净历史，**`dev`** 放待验证的阶段性改动。
- `master`（`6aa05ae`）：**历史已整理**（见 §0.12）。
  `master` 是从 `99df977^`（`5baaae9`）重做的 **10 个逻辑提交**，覆盖整个 DRM 工作，
  最新 `6aa05ae arm64: configs: chopin_defconfig for the v2 display stack`。
  更早历史从上游基线 `2ca60ae Linux 7.2.8 (clean upstream baseline)` 算起，全库 35 个提交。
- **不变量**：`git -C git/chopin-linux-7.2.8 status` 干净，且用
  `GIT_DIR=<repo>/.git GIT_WORK_TREE=linux-7.2.8 git status --porcelain` 对全部
  95k 文件比对，**0 处差异** —— 提交的源码就是我们构建的源码。
- **回退点**：tag `chopin-v2-working`（`21d1ba4`，整理前旧历史）。与 `master` 只差
  那个被清掉的杂物文件 `drivers/memory/chopin_defconfig`。
- port 仓库：见其最新提交（镜像发布 + 本文档 + 工具）。
- `verify-drm.sh` all local checks passed。

## 10. 工具状态（接手时先看一眼）

- `port/tools/verify-drm.sh` **全绿**：`all local checks passed`。
  `port/tools/audit-repo.py` 的目录展开缺陷已修（检查面 83 → 404 个文件），
  `MANIFEST.txt` 已按 404 项重新生成。
- `dmesg` 是 initramfs 里的自定义命令，**只接受一个过滤词、不能用管道**（`dmesg | grep` 会失败）。
- `/tmp` 不可写，日志一律放 `port/audit/`（本轮日志：`v2fiftyfirst-build.log`、`v2fiftyfirst-defconfig.log`）。
