# chopin（Redmi Note 10 Pro / MT6893）Mu-Silicium UEFI 移植

> 面向下一个会话。参考实现：`Mu-Silicium/Silicon/MediaTek/MT6789Pkg`（POCO M5 / stone）与
> `MT6768Pkg`（Redmi 9 / lancelot）；硬件参数证据见
> `port/audit/mt6893-uefi-hardware-params.md`，Mu-Silicium 代码解剖见
> `port/audit/mu-silicium-mtk-port-anatomy.md`。

---

## 0. 状态一句话

在 `Mu-Silicium/` 里新增了 **MT6893 硅层包 + chopin 平台包**，可构建出 UEFI FD 与
可刷写的 Android boot 镜像。**2026-10-16 上机验证：`LK → GZ → UEFI(EL1) → SEC → DXE`
整条链打通，framebuffer 控制台可见完整启动日志**。UFS 存储驱动已写好并进入 FV，
ACPI 与 MT6360 电源仍是缺口（第 5 节）。

### 上机已确认（硬证据）

| 结论 | 证据 |
| --- | --- |
| FD 被 LK 解压到 `0x40080000` 并从 FD 基址执行（首指令 `b +0x1000` 进 FV） | 屏幕出现 `Project Silicium for Xiaomi Redmi Note 10 Pro`（照片 `work/device/uefi-2026-10-16-firstboot-msdc-clk-assert.jpg`） |
| UEFI 在原厂 GZ 链下以 EL1 运行，SEC/DXE 正常 | `Loading DxeCore at 0x00479F000`、HOB list、Memory Allocation 列表 |
| 内存映射被接受（无重叠、`DXE Heap` 找到） | 分配一路到 `0x47EF0000`（落在 60 MiB DXE Heap 内） |
| 帧缓冲控制台可用（写在 LK 留下的 `0x78E0C000`） | 屏幕上的固件版本、逐个驱动加载日志、断言文本 |
| APRIORI 顺序符合预期（Clock → PmicWrapper → MT6359 → DiskIo → MsdcDxe） | 照片里的 `Loading driver at …` 序列 |
| MT6359 的 pwrap（WACS2 `0xC80…`）能读通 | `MT6359PmicDxe` 的 DEPEX 依赖 wrapper 协议，它被成功 dispatch |
| `bootopt`/DTBO/gzip 三个启动条件都满足 | GZ 没有报 32 位 instruction abort，直接进 payload |
| **BDS 起来了**：DXE 全量 dispatch 完成、图形 logo 上屏、BDS 开始加载应用 | 照片 `work/device/uefi-2026-10-16-bds-ufs-cache-assert.jpg`：`Project Silicium` Si logo + `Loading driver … BootManagerMenuApp.efi` |
| **UFS 前端流程走到了数据传输** | 该断言来自 `UfsHcDxe` 调用的 `ArmCacheMaintenanceLib`（见 3.7），说明 HCE、时钟、M-PHY、ref clock、UIC/链路至少已经跑到第一个 `Flush()` |
| 第三次启动（r3）：过了缓存断言，改成 **CPU 异常**（`ASSERT [ArmCpuDxe] CpuExceptionHandlerLib.c(438)`） | 视频 `work/device/uefi-video-2026-10-16/`（抽帧）；异常 dump 走 ConOut，屏幕上只剩断言行，见 3.8 |
| 第四次启动（r4）：**不再崩溃**，BDS 跑完 load-option dump、倒计时 `Wait (2) >>>`，最后停在 `ConIn in first WaitForKey in Lazy ConIn mode`（等按键） | 视频 `work/device/uefi-video-2026-10-16-r4/`（抽帧 `f010`/`rf010` 可读到整屏日志） |
| UFS 是否真的枚举出磁盘：**仍未证明**。dump 里的 `Internal Storage` 是 `MsBootOptionsLib.c:246` 注册的**静态**启动项，不代表发现了块设备 | 同上 + 源码 |
| 第五次启动（r5）：**UFS 时钟/协议都起来了**（日志：`INFRA_AO_UFS`/`UNIPRO_SYSCLK` 182 MHz、`TOP_UFS_SEL` 父 `TOP_MAINPLL_D6_D2`、`UFS: controller protocol installed`），但约 23 s 后崩在 `Data abort: Translation fault, second level`，**FAR = 0** | 视频 `work/device/uefi-video-2026-10-16-r5/`，抽帧 `f004`（UFS 日志）/`f022`、`f023`（异常 dump） |
| 根因（离线定位）：EDK2 的 UFS PRDT 数据地址写成 `addr >> 2`、UTRD 的 UCD 基址写成 `addr >> 7`，而 MTK 原厂驱动与主线 Linux 都写**原始 64 位地址** → 控制器往 `addr/4`、`addr/128` 做 DMA，读写全错并踩内存 | `Resources/MuPatches/Ufs-Dma-Addresses.patch`（新，13 个 hunk），见 3.9 |
| 第六次启动（r7，含 DMA 修正）：**跑到 BDS 不再崩**，屏幕在「Si logo + BDS 菜单」和「调试日志」之间来回闪（按一下键重画一次） | 视频 `work/device/uefi-video-2026-10-16-r7/`，抽帧 `g025`（日志）/`g006`、`g008`（黑屏） |
| 绿块从没出现过的真正原因：framebuffer 平面**带 alpha 混合**，标记颜色 alpha=0x00 全透明；控制台白字是 `0xFFFFFFFF` 才可见。主线移植当时用的是 `0xff00ff00` | 见 3.11 |
| UFS 是否枚举出磁盘：**仍未证明**。日志末尾那行 `… not found, Status = EFI_NOT_FOUND` 更像 `MsBootPolicyLib` 没找到可引导介质（chopin 的 UFS 上没有 ESP），不能当作"没有块设备"的证据 | r8 新增 BlockIo 计数行来直接回答，见 3.11 |
| 第七次启动（r8，alpha 修正）：**标记终于可见**。逐帧统计（连通域 + 长宽比过滤反光）：启动早期 1 个绿块，**约 14 s 时整行 8 个绿块**（= r8 编号的 stage 8），黄块始终 0 | 视频 `work/device/uefi-video-2026-10-16-r8/`：`s029.png`（整行 8 绿）、`s040.png`（4 蓝 + 4 绿，被 BDS 清屏/角度遮住） |
| 第八次启动（r10 照片）：`st=10 blk=0 HCE=0x1`；**`OCS0..3 = 0f,0f,0f,0f`**；**UIC 各层错误码全 0**；`IS` 里 UCCS(bit10) 已置位 | 照片 `work/device/uefi-r10-photo/zoom-top.png` |
| 由以上可得：**UIC 命令层全通（链路起来了），但 UTP 传输请求从未被控制器完成**（0x0f 就是驱动写进去的初值）→ 没有块设备、没有可引导介质 | 见 3.13 |
| 第九次启动（r12 照片）：`HCS=0x0000000f`（DP+UCRDY=1 设备在位、UIC 就绪）、`RSR=0x1`（传输列表在跑）、`DBR=0`（门铃被取走了）、`CLR=0`、`UBA=0x47060000`（基址正常）、`OCS` 仍 `0f` → **主机侧全部正确，请求被接受却永远不完成**，属数据通路问题 | 照片归档在 `work/device/uefi-r10-photo/`（同名目录） |
| 与厂商 `ufs_mtk_pre_link()` 对照又查出两处缺陷：① 我的 DME 命令 `Arg1` 编码错了（应为 `MIB<<16`，我传了裸 MIB → 一直在读写"属性 0"）；② 漏了 **`PA_LOCAL_TX_LCC_ENABLE=0`** 和**关闭 "deep stall"**（`VS_SAVEPOWERCONTROL` bit6）——deep stall 正是"UIC 能过、UTP 永不完成"的成因 | r13 修复，见 3.14 |
| **结论修正**：整行 8 绿说明 `UfsPassThruDxe` 确实被启动、链路起来、并且完成过完整的 Map→Flush→Unmap（一次传输），但 `BlockIo` 仍为 0 → 卡在设备/单元描述符到 SCSI 设备创建之间。BDS 菜单会清屏，那几条一次性报错读不到 | r9/r10：早期钩子标记、CAP/VER、主动 Connect、**每秒重写的寄存器状态行**，见 3.12 |

**上机暴露并已修掉的一个真问题**：MsdcImplLib 沿用了 MT6789 的时钟名 `TOP_MSDC50_0`，
而 MT6893 的主线时钟树里只有 `TOP_MSDC50_0_SEL`，于是 DEBUG 构建在
`ASSERT_EFI_ERROR` 处停住整机。修法见 3.6 节。

---

## 1. 交付物（全部为新增，MediaTekPkg 只动了 3 行死代码）

| 路径（相对 `Mu-Silicium/`） | 作用 | 依据 |
| --- | --- | --- |
| `Silicon/MediaTek/MT6893Pkg/MT6893Pkg.dsc.inc` | SoC PCD（GIC/核数/OEM TableId/SMBIOS）+ 库绑定 | MT6789Pkg.dsc.inc |
| `…/Library/GpioImplLib/` | pinctrl 寄存器布局 + `MaxPin=226` | mainline `pinctrl-mtk-mt6893.h`（227 个 pin，最高序号 226） |
| `…/Library/ClockImplLib/` | 244 条时钟表（Fixed 4 / PLL 17 / MuxGate 66 / Factors 83 / Gate 74），从 mainline `clk-mt6893-*.c` 机械派生并与厂商 `clk-mt6885.c` 交叉核对 | 文件头列了 168 条未映射时钟及原因 |
| `…/Library/PmicWrapperImplLib/` | MT6359 的 pwrap/WACS2 寄存器（**第三套布局**） | mainline `mtk-pmic-wrap.c` 的 `mt6873_regs` |
| `…/Library/MsdcImplLib/` | MSDC0/1 时钟、GPIO（MSDC0: 183-194；MSDC1: 51-56）、上电流程 | mainline pin table；电源部分目前是 no-op |
| `…/Library/PlatformSecLib/` | 关看门狗（`toprgu@10007000` 写 `0x22000000`）+ **EL2→EL1 下沉** | `mtk_wdt.c`；`MT6768Pkg` 的 `PlatformHelper.S` |
| `…/Drivers/MT6359PmicDxe/` | MT6359 电源/Home 键（`TOPSTATUS=0x2a`，bit1/bit3）；regulator 暂返回 `EFI_UNSUPPORTED` | mainline `mtk-pmic-keys.c` 的 `mt6359_regs` |
| `Platforms/Xiaomi/chopinPkg/{chopin.dsc,chopin.fdf,chopin.dec,DeviceBuild.py}` | 平台构建 | lancelot/stone 平台包 |
| `…/Include/{APRIORI,DXE}.inc` | FV 里各模块的顺序 | lancelot，替换为 MT6359 |
| `…/Library/MemoryMapLib/` | **31 个 LK 保留区 + MMIO 区**（跨模块硬约定，见第 2 节） | mainline chopin dts 的 `mblock-*` |
| `…/Library/KeypadDeviceLib/` | 音量键走 **KPD `kp@10010000`**，电源键走 MT6359 | vendor `hal_kpd.h` |
| `Resources/Configs/chopin.toml` | boot.img 布局（与厂商逐字段一致） | `work/boot_a_head.bin` + `port/tools/build-images.sh` |
| `Resources/DTBs/chopin.dtb` | **原厂 DTB（DTBO 容器）**，复制自 `port/out/stock-wrapped.dtb` | LK 的 dtbo overlay 需要基树里 60 个符号 |
| `…/Drivers/UfsHcDxe/{UfsHcDxe.c,UfsPlatform.c,UfsHcDxe.h,.inf}` | **UFS 平台驱动**：装 `EDKII_UFS_HOST_CONTROLLER_PROTOCOL`（MMIO/DMA）+ `EDKII_UFS_HC_PLATFORM_PROTOCOL`（MTK 复位/时钟/M-PHY） | mainline `ufs-mediatek.c`、`phy-mtk-ufs.c`、vendor `ufs-mtk-platform.h` |
| `port/tools/verify-chopin-uefi-img.py` | 校验产物头字段/gzip 头，`--fix-gzip` 可在页数不变时就地重压 | 本文件第 6 节 |
| `port/tools/check-chopin-uefi-memory-map.py` | 离线查内存映射重叠 / 缺失区域名 | 本文件第 6.2 节 |

对上游的改动（都在 `Mu-Silicium/` 内，都是为了「构建出来的镜像真的能被 LK 启动」）：

1. `build_uefi.py` 的 gzip 分支：`gzip.open()` 会把输出文件名写进 gzip 头（FNAME），
   而 **LK 的 inflate 不认 FNAME**（实测报 `decompress kernel image fail!!!`，见
   `port/README.md` 的同类坑）。改成 `GzipFile(filename="", mtime=0)` 并把头部规范化成
   `FLG=0 / MTIME=0 / OS=10`（与原厂镜像一致）。
2. `Silicon/MediaTek/MediaTekPkg/Drivers/MsdcDxe/MsdcDxe.c`：`MsdcGetNextSlot()` /
   `MsdcGetSlotNumber()` 里删掉两个「赋值后从未使用」的局部变量 `Private`——
   clang 19 在 `-Werror` 下会因此中断构建。行为不变，同时让 lancelot/stone 也能用新 clang 构建。
3. （正常现象）`Mu_Basecore` 子模块会显示为 modified：`build_uefi.py` 在构建前后会
   apply/remove `Resources/MuPatches/*.patch`，构建完保持 apply 状态。

---

## 2. 关键硬件参数（写代码时被引用到的那些）

| 项 | 值 | 证据 |
| --- | --- | --- |
| DRAM | `0x40000000` + 8 GiB（2 rank × 4 GiB） | `…lpm-memory.dtsi:444`，LK 日志 `n10p/ex.txt:11905` |
| LK / ATF / GZ | `0x48200000+0x600000` / `0x48C00000+0x200000` / `0x7EE00000+0x1000000` | `n10p/ex.txt:8836`、NoGZ 手册、`mblock-7/4` |
| 内核加载地址 | `0x40080000`（= UEFI FD base） | boot header `kernel_addr`、`boot linux-64 … pc=0x40080000` |
| tags / ramdisk / dtb | `0x47C80000` / `0x51100000` / `0x41F78000` | `work/boot_a_head.bin` 逐字段 |
| GIC | Dist `0x0C000000`、Redist `0x0C040000` | `…infra-pinctrl.dtsi:332` |
| 时钟控制器 | topckgen `0x10000000`、infracfg_ao `0x10001000`、apmixedsys `0x1000C000` | mainline dtsi |
| GPIO | 基址 `0x10005000`；DIR `+0x000` / DOUT `+0x100` / DIN `+0x200` / MODE `+0x300`，步距 `0x10` | 与 MT6789 布局一字不差 → 只改 `MaxPin` |
| PMIC | MT6359 挂 pwrap `0x10026000`；MT6360 挂 i2c5；MT6315×3 挂 SPMI `0x10027000` | `n10p/ex.txt:446-453` |
| 按键 | KPD `kp@10010000`：`KP_MEM1(0x04)` bit0=音量上、bit1=音量下；电源键 = MT6359 PWRKEY | `kpd-hw-init-map=<0x73 0x72 …>`、`hal_kpd.h` |
| 显示 | **LK 已点亮**，存活帧 `0x78E0C000`，1080×2400，stride 4352（=1088px×4）→ `PcdFrameBufferWidth=1088` | mainline simple-framebuffer 节点 + 实机日志 |
| 看门狗 | toprgu `0x10007000`，默认 31 秒；`MODE(0x00)` 写 `0x22000000` 关闭 | `mtk_wdt.c` |
| 存储 | **UFS**：ufshci `0x11270000`、mphy `0x11FA0000`；SD 走 MSDC1 `0x11240000` | 8 条独立证据见硬件报告 §7 |

`MemoryMapLib` 里的区域名是跨模块硬约定，写错一个就是 `CpuDeadLoop()` 或 DXE 起不来：
`"DXE Heap"`（Sec 必需）、`"Display Reserved"`（SimpleFb + 帧缓冲控制台）、
`"MSDC-%u"`/`"MSDC Top-%u"`、`"WatchDog Timer"`、`"Pinctrl"`、`"PMIC Wrapper"`、
`"Keypad"`、`"AP Mixed"/"Top Ck Gen"/"Infra Cfg Ao"`。

---

## 3. 启动链分析与移植决策

### 3.1 链路

```
preloader → LK（AArch32/Thumb-2，0x48200000）→ GenieZone（EL2 hypervisor）
          → payload（boot.img 的 kernel）= 我们的 UEFI FD，EL1h
```

GZ 的日志 `boot linux-64 via EL2, pc=0x40080000, x0=0x47c80000, mode=0x3c5`：
`0x3c5` = **EL1h**，即 payload 由 GZ 以 EL1 启动（GZ 自己待在 EL2）。打了
`kvm/mtk-mod-tee-nogz` 的 NoGZ ATF 补丁后是 `0x3c9` = **EL2h**。

### 3.2 决策 1：不做 BootShim，FD 直接当 kernel

* LK 用 boot.img 头的 `kernel_addr`（= `0x40080000`）解压并跳转，**不看** ARM64 Image 头：
  在 `lk.bin` 里全文搜索 **不存在** `"ARM\x64"`（`0x644d5241`）字面量，也没有
  `text_offset` 字符串；而 `"64Bit"/"32Bit"` 与 `"64 bit kernel can't boot at"`
  说明 32/64 位只由 `bootopt=` 决定（`port/README.md:490-517` 的实验已证明）。
* 因此 `Resources/Configs/chopin.toml` 里 **不设 `[boot_shim]`**，
  `kernel_compression = "gzip"`（LK 自带 gunzip）——这正是 lancelot 的做法。
* gzip 头必须是 `FLG=0`（不能带 FNAME），否则 LK 报 `decompress kernel image fail!!!`。
  Python 的 `gzip.open()` 默认不带 FNAME，`verify-chopin-uefi-img.py` 会检查。

### 3.3 决策 2：栈放在 FD 之后，而不是 `0x40000000`

LK 把 `0x40000000+0x80000` 作为 `mblock[0]` 保留（`n10p/ex.txt:8840`），所以
0x40000000 这 512 KiB 不归我们用。chopin 的布局是：

| 区域 | 地址 | 大小 |
| --- | --- | --- |
| UEFI FD | `0x40080000` | `0x200000` |
| UEFI Stack | `0x40280000` | `0x40000` |
| DXE Heap | `0x402C0000` | `0x7D40000`（到 `0x48000000` 为止，避开 LK/ATF） |

### 3.4 决策 3：异常级别

`PlatformHelper.S` 抄 **MT6768** 而不是 MT6789：`EL1_OR_EL2` 判断，
在 EL1 直接 `ret`（原厂 GZ 路径），在 EL2 才下沉到 EL1（NoGZ 路径）。
MT6789 版是无条件 `ret`，一旦上 NoGZ 补丁就会让 SEC 在 EL2 跑「以为自己在 EL1」的代码。

> ⚠️ 取舍：下沉到 EL1 之后，UEFI 交给 OS 的也是 EL1，**KVM/EL2 的手交就没了**。
> 若将来要「UEFI 在 EL2、把 OS 用 EL2h 交出去」（`kvm/` 那条路线的目标），
> 需要把这段下沉改成条件化或去掉，并重新验证 DXE 在 EL2 下的行为。

### 3.5 决策 4：boot.img 必须带原厂 DTB

LK 会把 `dtbo` 分区的覆盖层套在 boot.img 的基树上，需要基树里有 **60 个符号**，
否则 `ufdt_overlay_do_fixups()` 失败、**LK 直接不启动内核**。所以
`Resources/DTBs/chopin.dtb` 用的是原厂打包树（DTBO 容器，201848 字节），
不是我们自己编的 `mt6893-xiaomi-chopin.dtb`。

### 3.6 上机教训：时钟名不能照抄别的 SoC，也不要 ASSERT

第一次上机时 MsdcImplLib 沿用了 MT6789 的 `TOP_MSDC50_0` / `TOP_MSDC30_1`，
但 MT6893 的主线时钟树里根本不存在这两个节点——MSDC 的源时钟是带门控的
`TOP_MSDC50_0_SEL` / `TOP_MSDC30_1_SEL`（`clk-mt6893-topckgen.c` 的
`MUX_GATE_CLR_SET_UPD`），再往下才是 `INFRA_AO_MSDC0_SRC` → `INFRA_AO_MSDC0`。
名字对不上时 ClockDxe 返回 `EFI_INVALID_PARAMETER`，而库里紧跟的
`ASSERT_EFI_ERROR` 在 DEBUG 构建里直接把整机停在 DXE。两条改动：

* `MsdcImplLib` 改用真实存在的 `*_SEL` 名字（宏集中在文件顶部）；
* 这个库里所有「取不到时钟/协议」的分支都改成 **只打日志不 ASSERT**——
  一个名字写错不应该让整台机器起不来。同样的理由，构造函数的三个
  `LocateProtocol` 断言也去掉了。

顺带记一条可复用的检查：`port/tools` 之外，可以用一条 python 交叉核对
「FV 里所有驱动 `GetId()` 引用的时钟名」是否都在 `ClockImplLib` 里
（本次核对结果：MT6893 侧 0 缺失）。

### 3.7 上机教训：AArch64 上 `CacheMaintenanceLib` 的「整缓存」入口是空实现

第二次上机 BDS 已经起来（Si logo + 开始加载 `BootManagerMenuApp.efi`），
最后停在：

```
ASSERT [UfsHcDxe] ArmPkg/Library/ArmCacheMaintenanceLib/ArmCacheMaintenanceLib.c : ((BOOLEAN)(0==1))
```

原因是 UFS 的 DMA 路径：MTK 的 UFS 总线主设备**不是 cache coherent**（设备树里没有
`dma-coherent`），EDK2 把 DMA 的缓存维护交给平台驱动，而我在 `Flush()` 里调了
`WriteBackInvalidateDataCache()`——这个「整缓存」入口在 AArch64 的
`ArmCacheMaintenanceLib` 里就是 `ASSERT (FALSE)`。该库**只有 range 版本是真实现**
（`WriteBackDataCacheRange` = DC CVAC、`InvalidateDataCacheRange` = DC IVAC、
`WriteBackInvalidateDataCacheRange` = DC CIVAC；`InvalidateDataCache`/
`WriteBackDataCache`/`WriteBackInvalidateDataCache`/`InvalidateInstructionCache`
四个整缓存入口全部 `ASSERT(FALSE)`）。

于是改成「自己记住设备能碰到的缓冲区，全部用 range 操作」：

* `AllocateBuffer()` 把每个 4 KiB 对齐的缓冲区记进私有数据（UTRL/UTRD/命令描述符）；
* `Flush()` 对这些缓冲区逐个 `WriteBackInvalidateDataCacheRange()`——
  CIVAC 双向都安全（脏的描述符会被写回，设备写回的数据会被丢弃以便 CPU 重读）；
* `Map()`/`Unmap()` 为每个映射分配一条记录：`BusMasterRead` 先 CVAC（设备要读），
  `BusMasterWrite` 先 IVAC，`Unmap` 时对 `BusMasterWrite` 再 IVAC 一次
  （传输期间 CPU 可能又把那些行填回缓存），然后释放记录。

给别的 MTK/非一致平台移植时的通用教训：**不要在平台驱动里用整缓存入口**，
要么用 range 版本并自己跟踪缓冲区，要么自己写 DC ISW/CISW 循环。

### 3.8 第三次启动：CPU 异常 + 「日志看不见」的三个原因

r3 的启动视频里，BDS 已经起来、UFS 也跑过了第一段 DMA 路径，最后死在
`ASSERT [ArmCpuDxe] CpuExceptionHandlerLib.c(438)` —— 也就是 `ArmCpuDxe` 的默认
异常处理器在打印完寄存器 dump 后 `ASSERT(FALSE)`。**光知道"有异常"不够，还需要
ESR/ELR/FAR**，但这台机器既没有串口、屏幕上的日志又几乎看不见。查下来有三个原因：

1. **`PcdDebugPrintErrorLevel = 0x8007EE0F` 把 `DEBUG_INFO` 过滤掉了**
   （第 6 位为 0），所以平台驱动里所有 `DEBUG_INFO` 的步骤日志根本没输出。
   已在 `chopin.dsc` 里改成 `0x8007EE4F`（只多打开 INFO）。
2. **缓存维护里我用了纯 `InvalidateDataCacheRange`（DC IVAC）**：它会**丢掉**同一条
   缓存行上属于其它对象的脏数据——DMA 缓冲区只要和堆元数据共行，就是一次静默的
   堆破坏，随后在任意地方炸成数据异常。已全部改成 `WriteBackInvalidateDataCacheRange`
   （DC CIVAC，干净+失效，双向都安全）。
3. **控制台在 BDS 之后只剩最后几行**：BDS/logo 会重画 framebuffer，而
   `FrameBufferSerialPortLib` 是另一套光标，于是异常 dump 基本被盖掉。
   对策是**不依赖控制台**：`UfsHcDxe` 现在直接往两块候选 framebuffer
   （`0x78E0C000` / `0x77A20000`，主线移植验证过这两块都能被面板扫到）里画
   **阶段标记**——第 1..7 个绿色方块对应：

   | 标记 | 含义 |
   | --- | --- |
   | 1 | `InitUfsHc` 完成（时钟开、协议装好） |
   | 2 | PreHce（UFSHCI 复位 + CLK_CG + AHIT=0） |
   | 3 | PostHce（M-PHY 上电 + ref clock 握手 + 600 µs） |
   | 4 | PreLinkStartup（UniPro power-down 检查） |
   | 5 | 第一次 `Map()`（开始有 DMA 传输） |
   | 6 | 第一次 `Flush()`（缓存维护） |
   | 7 | 第一次 `Unmap()`（一次传输走完） |

   标记是直接写 DRAM 的，崩溃后仍留在屏上；**有几个方块 = 走到第几步**。
   整行在 Stage 0 时会先清空，所以每次启动都从零开始数。

> 同类平台驱动移植的通用教训：**不要用整缓存入口**（AArch64 上是空实现），
> **不要用裸 IVAC 去失效"别人也可能有脏数据"的缓冲区**，以及在没有串口的设备上
> **把关键进度画到屏幕上**比指望控制台可靠得多。

**r4 上机结果：异常消失，BDS 正常停在等按键。** 也就是说 3.8 里那三处修正（尤其
IVAC→CIVAC）解决了 CPU 异常。但视频里**一个绿色标记都没有**——帧扫描（按纯绿像素
检测）结果为 0：标记画在 DXE 驱动初始化阶段，而 BDS 之后会**整屏重画**（boot logo +
控制台），把它们盖掉了。于是 r5 把标记改成：

* 位置挪到面板内部显眼处（`x=100, y=560`，8 个 32×32 槽位，带黑色底条），不再躲在
  圆角里；
* 由一个 **1 秒周期定时器持续重画**（`MtkUfsMarkerTick`），所以 BDS 之后无论谁重画屏幕，
  下一次 tick 都会把标记盖回去；
* 语义扩到 8 个（顺序即发生顺序，**第 5 个 = `PostLinkStartup`，出现即证明 UFS 链路起来了**）：

  | 槽位 | 含义 |
  | --- | --- |
  | 1 | `InitUfsHc` 完成（时钟开、协议装好） |
  | 2 | PreHce（UFSHCI 复位 + CLK_CG + AHIT=0） |
  | 3 | PostHce（M-PHY 上电 + ref clock 握手 + 600 µs） |
  | 4 | PreLinkStartup（UniPro power-down 检查） |
  | 5 | `PostLinkStartup` —— **链路建立成功** |
  | 6 | 第一次 `Map()`（开始 DMA） |
  | 7 | 第一次 `Flush()` |
  | 8 | 第一次 `Unmap()`（一次传输完整走完） |

### 3.9 UFS 真正跑不起来的原因：EDK2 把 DMA 地址编码写错了

r5 上机后有两个现象：UFS 的时钟和协议栈都正常初始化（日志见 §0 第五行），可
约 23 秒后整机崩在 `Data abort: Translation fault, second level`，`FAR = 0`
（即写空指针）。这类「随机位置空指针」是**内存被踩**的典型症状。顺着 UFS 的 DMA
路径对照第三方实现，找到两处同类 bug：

| 位置 | EDK2（错误） | MTK 原厂 `kernel_mt6893-chopin_kvm` | 主线 Linux 7.2.8 |
| --- | --- | --- | --- |
| PRDT 数据地址 `UTP_TR_PRD.DW0` | `DbAddr = addr >> 2`（字段声明成 `Rsvd1:2; DbAddr:30`） | `base_addr = lower_32_bits(dma_address)`（原始） | `prd->addr = cpu_to_le64(sg->dma_address)`（原始） |
| UTRD 的 UCD 基址 `UTP_TRD.DW4` | `UcdBa = addr >> 7`（字段声明成 `Rsvd6:7; UcdBa:25`） | `command_desc_base_addr_lo = lower_32_bits(addr)`（原始） | `__le64 command_desc_base_addr`（原始） |
| UTRL/UTRD 基址寄存器 | 原始值 ✓ | 原始值 ✓ | 原始值 ✓ |

两个参考实现互相独立却一致，因此结论明确：**MT6893 的 UFS 控制器要的是原始字节
地址**。EDK2 那两处位移让控制器把数据写到 `addr/4`、把命令描述符读到 `addr/128`
的位置——既读不到正确数据，又把无关内存覆盖掉，于是 BDS 阶段随机崩溃。

修法（`Resources/MuPatches/Ufs-Dma-Addresses.patch`，已挂进 `build_uefi.py` 的
patch 列表，`git apply` 正向可应用、反向校验通过）：

* `MdePkg/Include/IndustryStandard/UfsHci.h`：`UTP_TR_PRD.DW0` 与 `UTP_TRD.DW4`
  改成完整的 32 位字段（低位不再当保留位），并在注释里写明依据；
* `UfsPassThruDxe/UfsPassThruHci.c`：PRDT 一处、UCD 基址三处改为写原始地址；
* `UfsBlockIoPei/UfsHci.c`：同样三处写入 + 三处**读回**（`<< 7`）一起改，保持自洽。

> 附带说明为什么这个 bug 能潜伏这么久：Mu-Silicium 里没有任何平台真正用过
> `UfsPassThruDxe`（Qualcomm 用预编译 UFSDxe、Samsung 用自家 UfsDxe），所以这段
> 通用代码从没在真实硬件上跑过。**这也是本次移植第一个「不是我们写错、而是上游
> 从未被验证」的缺陷。**

### 3.10 阶段标记为什么在屏上看不见（待确认）

r4/r5 的绿色方块一次都没出现在视频里（对全部帧做纯绿像素扫描：0 命中），
而同一块 framebuffer（`Display Reserved` = `0x78E0C000`）上的控制台文字却看得见。
当前判断是**位置被控制台文字压住**（`y=560` 落在满屏日志的范围内）而不是写不进去，
所以 r6/r7 把标记行移到面板最底部（`y=2336`），并给定时器创建和每个阶段加了
`DEBUG_ERROR` 日志；r7 上机后二选一验证：要么看到底部一行方块，要么在日志里
读到 `UFS: marker timer started (Success)` 与 `UFS: stage N`。

### 3.11 屏幕上的三条线索：alpha 混合、双控制台抢屏、以及"到底有没有磁盘"

r7 上机后不再崩溃（说明 3.9 的 DMA 地址修正起作用了），但屏幕表现为**交替**：
一会儿是 Si logo + BDS 菜单，一会儿是一屏调试日志，按键时重新画一次。原因有两个：

1. **两套控制台共用一块 framebuffer**：`GraphicsConsoleDxe`（GOP，BDS 菜单/logo 走它）
   和 `FrameBufferSerialPortLib`（DEBUG 输出走它）各自维护光标、互不知情，谁后画谁覆盖。
   这不是 bug，是移植期缺少"统一控制台"的必然现象；要看干净日志就得让 BDS 菜单
   别把屏清掉（例如下次出诊断版时把 `BootManagerMenuApp` 从 FV 里去掉）。
2. **标记一直是透明的**：bootloader 把 framebuffer 平面配置成 **alpha 混合**，我写的
   颜色是 `0x0000FF00`（alpha=0x00）→ 完全透明；控制台文字 `0xFFFFFFFF` 才看得见。
   主线内核移植当初用的是 `0xff00ff00`/`0xffff0000`（alpha=0xFF），所以那份代码
   看得见。**教训：这块屏上写像素必须带 `0xFF` alpha。** r8 已把所有标记色改成不透明。

另外 r8 给标记加了**第二行**，直接回答"UFS 有没有枚举出磁盘"：

| 行 | 颜色 | 含义 |
| --- | --- | --- |
| 第一行（`y=2336`） | 绿=成功 / 红=失败 / 暗蓝=未到达 | UFS 驱动走到第几步（1..8，第 5 个 = 链路建立成功） |
| 第二行（`y=2280`） | **黄色块数 = `BlockIo` 协议句柄数** | chopin 的 SD/eMMC 检测不到卡（日志里 `Failed to detect card`），所以**黄块非零就等于 UFS 出了一个块设备** |

同一份计数也会在变化时打进日志：`UFS: BlockIo handles = N`，这样即使标记又被
BDS 覆盖，也能从日志读到。

### 3.12 把「UFS 走到哪一步」变成可量化的：1 绿 / 0 黄说明了什么

r8 的 alpha 修正让标记第一次可见。逐帧统计（连通域 + 长宽比过滤反光条纹）得到的
序列是：**启动早期 1 个绿块 → 约 14 s 时整行 8 个绿块（满了）**，而第二行黄块始终为 0。

> 我最初只扫了“绿像素最多的簇”、又拿最早期的一帧下结论，误判成“只到 stage 1”。
> 正确读法见 `work/device/uefi-video-2026-10-16-r8/s029.png`（整行 8 绿）与
> `s040.png`（4 蓝 + 4 绿，是 BDS 清屏/拍摄角度遮掉了上半行，不是阶段倒退）。

含义：

* **跑到 stage 8（r8 满行）= 走完 PreHce → PostHce → PreLinkStartup →
  PostLinkStartup → 第一次 Map → Flush → Unmap**：控制器被启动、M-PHY 上电与
  ref clock 握手完成、**链路建立成功，并且至少完成了一次完整传输**（DMA 地址修正生效）；
* **0 个 `BlockIo` 句柄** → 仍然没有盘。失败点在链路之后、SCSI 设备出现之前：
  设备/单元描述符读取、电源模式/属性读写，或 SCSI PassThru 安装这几步之一。

`Start()` 的顺序是（`UfsPassThru.c`）：`OpenProtocol` → `GetUfsHcMmioBar`（我们的
实现）→ `AllocateCopyPool` → `LocateProtocol(UFS_HC_PLATFORM)` →
`GetUfsHcInfo`（读 `VER(0x08)`/`CAP(0x00)` 后调 `OverrideHcInfo`）→
`UfsControllerInit`（其第一步 `UfsEnableHostController` 里才触发 PreHce）。
所以嫌疑只有两个：**BDS 没连接这个句柄**，或者 **`GetUfsHcInfo` 阶段失败**。

r9 就是为这两个嫌疑设计的（标记槽位扩到 12，阶段重编号）：

| 标记 | 含义 |
| --- | --- |
| 1 | `UfsHcDxe` 初始化完成、协议装好 |
| **2** | **`GetUfsHcMmioBar` 被调用 = `UfsPassThruDxe` 的 `Start()` 真的进来了** |
| **3** | **`OverrideHcInfo` 被调用 = `VER`/`CAP` 读成功、控制器 MMIO 正常** |
| 4 / 5 | PreHce / PostHce（M-PHY 上电、ref clock 握手） |
| 6 / 7 | PreLinkStartup / **PostLinkStartup（链路建立成功）** |
| 8 / 9 / 10 | 第一次 Map / Flush / Unmap（一次传输走完；r8 里是 6/7/8，满行 8 块即到这一步） |

同时 r9 还会：把 `VER`/`CAP`（含 Nutrs/Nutmrs）用 `DEBUG_ERROR` 打出来；在
`UfsHcDxe` 里**主动 `ConnectController` (Recursive=TRUE)** 并打印返回值——这样既排除
“BDS 没连接”的可能，也让 UFS 的初始化时机变得确定（不必等 BDS 的
`EfiBootManagerConnectAll`）。

r10 再解决“报错被 BDS 清屏冲掉”的问题——**每秒重写一条寄存器状态行**：

```
UFS poll: st=<阶段> blk=<BlockIo 数> HCE=0x.. HCS=0x.. IS=0x..
          UEC[PA=.. DL=.. N=.. T=.. DME=..] OCS0..3=xx,xx,xx,xx
```

* `HCS(0x30)` 的 DP 位 = 设备在位；
* `UECPA/UECDL/UECN/UECT/UECDME(0x38..0x48)` = UIC 各层错误码，**直接指出链路在哪一层出错**；
* `OCS0..3` 取自 UTRL（`0x50` 指向的描述符数组）每个 UTRD 的 OCS 字段：`0` 表示该传输
  成功，非 0 就是 UFS 的错误码（`OCS_INVALID_COMMAND`、`OCS_ABORTED` 等）；
* 前 4 次 `Map()` 还会打 `UFS: map#n op=.. len=.. host=.. dev=..`，用来核对 DMA 地址
  确实是恒等映射（3.9 那两处修正的现场证据）。

> 读标记的方法：**先看第一行绿块数**——2 就是"没被连接"，3 就是"MMIO 正常但
> 更早就失败"，4~10 就照着阶段表往下查。红色表示失败，块数对应
> `MtkUfsSetFailure` 的步骤号（1 UFS 区域、2 UFS PHY 区域、3 Infra Cfg AO 区域、
> 4 时钟协议、5 使能时钟、6 安装协议）。

### 3.13 卡点定位到 UTP 层：UIC 通、传输不完成

r10 的常驻状态行把范围缩到了很小（照片 `work/device/uefi-r10-photo/zoom-top.png`）：

```
UFS poll: st=10 blk=0 HCE=0x1 HCS=0x.. IS=0x405
          UEC[PA=00000000 DL=00000000 N=00000000 T=00000000 DME=00000000]
          OCS0..3=0f,0f,0f,0f
```

读法：

* `HCE=0x1`：控制器已使能；
* `IS` 有 **UCCS(bit10)** 与 bit2，且 **UIC 各层错误码（PA/DL/N/T/DME）全 0** →
  **UIC（DME）命令这一路是通的**，链路启动也确实成功了（否则
  `UfsControllerInit` 会直接返回错误，根本走不到 stage 7 之后）；
* **`OCS0..3 = 0x0f`**：`0x0f` 是驱动在发请求前写进 UTRD 的初值
  （`UFS_HC_TRD_OCS_INIT_VALUE`）。也就是说**控制器从未把前四条传输请求标记完成**，
  驱动只能超时返回，于是 `Map/Flush/Unmap`（stage 8/9/10）都发生过、但一条数据都没读出来。

> 注意 `Map/Flush/Unmap` 是在退出路径里无条件调用的，所以"满行绿"只证明**请求发出去了**，
> 不证明它成功——成功与否写在 `OCS` 里。这一点我上一轮说重了，已在 3.12 更正。

于是嫌疑收敛为"UTP 请求为什么没人处理"，按可能性排序：

1. `UTRLRSR(0x60)` 没有真正置 1（传输列表没跑起来）；
2. `UTRLDBR(0x58)` 门铃位还挂着（控制器根本没取走请求）；
3. `UTRLBA(0x50)` 与驱动实际分配的 UTRD 缓冲区不一致（基址编程不对）；
4. 以上都正常 → 那是 UniPro/设备侧的数据通路没配好（例如 UniPro 没真正退出
   low-power、或设备没被换到 FAST 电源模式）。

r12 把 1~3 全部搬进了状态行（`RSR`/`DBR`/`CLR`/`UBA`，并可与日志里
`UFS: map#1/#2 ...` 打印的地址直接对照），另外补上 `HCS`（bit0 = 设备在位）和
`IS`，一次拍照即可判定属于哪一类。

### 3.14 UTP 层卡点的两个真因：DME 参数编码 + 漏掉的 deep stall

r12 的照片把主机侧排除干净（`RSR=1`/`DBR=0`/`UBA` 正常/`HCS` 里 DP 与 UCRDY 都是 1），
只剩"请求被控制器接受后永远不完成"这一种解释。回厂商驱动逐句对照
`ufs_mtk_pre_link()`（它就在 PreLinkStartup 这个时机跑），发现两处实打实的问题：

**① DME 命令的 `Arg1` 编码是错的。** UFSHCI 规定 DME_GET/DME_SET 的 UICCMDARG1 =
`(MIB attribute << 16) | MIB selector`（Linux：`UIC_ARG_MIB_SEL`；`UIC_ARG_MIB(x)` =
`x << 16`）。我原来的代码传的是**裸 MIB**（`0xD0A8`），等于去访问 attribute 0 /
selector 0xD0A8——也就是说那条"清除 UniPro power-down"从来没作用在正确的属性上。

**② 漏了厂商 pre-link 的两个关键动作**（这也是"UIC 通、数据传输不通"的经典成因）：

| 厂商动作 | MIB | 作用 |
| --- | --- | --- |
| `ufs_mtk_unipro_set_lpm(false)` | `VS_UNIPROPOWERDOWNCONTROL 0xD0A8` = 0 | UniPro 退出 power-down |
| `ufshcd_disable_host_tx_lcc()` | `PA_LOCAL_TX_LCC_ENABLE 0x155E` = 0 | **关掉 TX 链路配置变更**，否则链路一直在重配、不承载 UTP 流量 |
| 关闭 deep stall | `VS_SAVEPOWERCONTROL 0xD0A6` bit6 = 0 | **deep stall 会在链路一空闲就把 UniPro 停住**——正是"UIC(DME) 能过、UTP 请求被取走却永不完成" |
| （post-link）`ufs_mtk_cfg_unipro_cg(true)` | `0xD0A6` bit0/2/3 = 1、`VS_DEBUGCLOCKENABLE 0xD0A1` bit5 = 0 | 链路起来后开 UniPro 时钟门控 |

r13 把这一整套按厂商顺序补齐，并且：

* 每个 MIB 的改动都打成一行日志（`UFS: deep stall cleared 0x0d0a6: 0x... -> 0x...`），
  这样即使标记被遮住，也能从日志确认修的是不是那条；
* **DME 失败不再让整个 UFS 初始化失败**（只记录），避免一个 MIB 读不通就把后面的
  流程全挡住——这比厂商实现更保守，因为我们的目标是先把 UTP 打通。

**r13 上机结果：`OCS` 仍是 `0f`、仍无黄块**，说明这两处修正还不足以打通 UTP（也可能
MIB 写入本身没生效）。于是 r14 把"链路真实状态"也读成状态行——UIC(DME) 是当前唯一
还能用的通道：

```
UFS UCD=<slot0 的 UCD 地址> CMD=<驱动构造的命令 UPIU 首 4 字节> RESP=<响应区首 4 字节>
UFS DME LCC=%u SPC=%08x UPD=%u
UFS DME PWR=%u TXG=%u RXG=%u TL=%u RL=%u
```

* `LCC=0`、`SPC` bit6=0：确认 r13 的 pre-link 写入是否真的落到 UniPro；
* `PWR`：1=FAST、2=SLOW(PWM-G1)、3/4=对应 auto 模式；`TXG/RXG` = 收发 gear；
* **`TL`/`RL` = `PA_ACTIVETXDATALANES`/`PA_ACTIVERXDATALANES`：如果是 0，就等于
  "控制通路能走、数据通路根本不存在"**——那 UTP 卡死就完全解释得通，修法是补上电源
  模式/通道数协商（厂商的 `ufs_mtk_change_pwr_mode` / 主线的 `ufshcd_change_power_mode`）；
* `RESP=0` 表示设备对这条请求**从未作出任何响应**。

> 安全注意：UCD 地址会先做 DRAM 范围检查（`0x40000000..0x80000000`）再读，
> 避免拿到野指针直接读 MMIO 触发异常。

**r14 上机直接崩了（我的错，值得记录）**：dump 里 `ESR ISS=0x21` = **Alignment fault**。
原因是我在**定时器里调用 `DriverInterface->UfsExecUicCommand`**——那个接口指针指向
`UfsPassThruDxe` 的 `Private` 结构，而它的 `Start()` 失败后会 **FreePool(Private)**，
于是指针变成野指针，里面的 `UfsHcBase` 是垃圾值，MMIO 访问落到非对齐地址 → 对齐异常。
**教训：平台回调拿到的 `DriverInterface` 只在回调期间有效，任何跨时刻（尤其定时器）
的使用都是 use-after-free。**

r15 的修正方式：

* DME 读取全部移回**回调内部**（`PreLinkStartup` / `PostLinkStartup`，接口有效），
  并把结果存进静态变量；定时器只**回放已抓到的数值**，不再访问任何驱动接口；
* 那个 stashed 的接口指针被**彻底删掉**，避免以后再被误用；
* 定时器里的 UTRL/UCD peek 增加 `0x40000000..0x80000000` 范围检查后再读，
  且只读**我们自己的**描述符内存。

**r15 上机（照片可读）：数据通路根本没建立。**

```
UFS DME LCC=0 SPC=00000090 UPD=0        ← r13 的 pre-link 写入确实生效（bit6=0、LCC=0）
UFS DME PWR=0 TXG=0 RXG=0 TL=0 RL=0     ← 电源模式=0、gear=0、活动通道数=0
UFS UCD=.. CMD=00000000 RESP=00000000   ← 设备从未应答
```

对照 Linux：`ufshcd_link_startup()` 在链路起来后立刻把状态标注为
**"PWM-G1, 1-lane, SLOW-AUTO"**（`ufshcd_init_pwr_info()`），也就是说这个时刻
`PA_PWRMODE`/`PA_ACTIVETXDATALANES`/`PA_ACTIVERXDATALANES` **应当已被链路启动写好**。
我们板上读回全是 0——**链路在"链路层"是通的（UIC/DME 能过、`HCS.DP=1`），但一条数据
通道都没激活**，这正是"UTP 请求被控制器取走却永远不完成"的直接原因。

r16 做两件事：

1. **自己发起一次电源模式变更**（顺序照 `ufshcd_change_power_mode`：
   先 `PA_ACTIVETXDATALANES`/`PA_ACTIVERXDATALANES` = 1，再 gear = 0、关 HS 终端，
   最后写 `PA_PWRMODE = SLOW_AUTO(4)`），并把读回值打进状态行；
2. **M-PHY 已被 LK 打开时不再重跑上电序列**（PLL 已上电、未隔离、force 已释放就跳过）——
   避免把 LK 建好的数据通路扰动掉，而 UEFI 又无法用 ATF 的 `UFS_MPHY_RESTORE` 复原。

### 3.16 策略转向：不再推倒重来，而是"原样接管 LK 的控制器"

r16 上机后 `PWR/TL/RL` **仍是 0**（我发起的电源模式变更没生效），而 `UCD=FFFFFFFF`
说明 Start 失败后那块描述符内存已被回收。把这些放回整条时间线看，有一个更简单的
解释一直没被验证：

> **LK 本来就把 UFS 用得好好好的**（我们的 payload 就是它从 UFS 读进来的）。而 UEFI
> 一上来就 **复位控制器 → 重开 HCE → 重跑 M-PHY/refclk → 重发链路启动**，等于把这套
> 已经工作的状态拆掉再重建；重建后的 link 只剩控制通路（UIC/DME 可用、`HCS.DP=1`），
> 数据通道数却是 0——之后再怎么改 `PA_PWRMODE` 也回不来。

EDK2 平台协议 v3 恰好为这种场景提供了两个开关：`SkipHceReenable` 与
`SkipLinkStartup`。r17 用它们做**纯接管**：

| 回调 | r17 行为 |
| --- | --- |
| PreHce | **不复位、不写 CLK_CG/AHIT**，只读回 HCE/XOUFS 并打日志 |
| PostHce | **不碰 M-PHY、不做 refclk 握手**，只读回 `8C`/`0x144` |
| PreLinkStartup | **只读**：打印 LCC/SPC/PWR/TL/RL，不写任何属性 |
| PostLinkStartup | 不做 post-link 时钟门控 |

代码里用 `MTK_UFS_TAKEOVER_FROM_LK` 一个宏控制"接管 / 从零初始化"两条路线，
切换回来只需改这一个宏。这是本轮**唯一的变量**，结果很干净：

* 若 r17 出盘（`blk>0`/黄块出现/`OCS` 变 `00`）→ 说明硬件通路本来就在，UEFI 该做的是
  "最小侵入接管"，后续再逐步加回必要的初始化；
* 若 r17 仍然 `OCS=0f` → 说明"重建链路"和"接管现场"两条路都不通，问题不在初始化顺序，
  而要回到控制器/设备层面的假设（例如 UTP 引擎需要 LK 没启用的某个条件）。

### 3.17 两条新线索：接管模式下"数据通路是活的"，以及 expdb 日志通道

**（一）r17（原样接管）的视频给出了关键差异**

```
UFS st=10 blk=0 HCE=00000001
UFS DME LCC=0 SPC=00000090 UPD=0
UFS DME PWR=3 TXG=0 RXG=0 TL=1 RL=1   ← FAST-AUTO、每方向 1 条活动通道
UFS UBA=47060000 UCD=FFFFFFFF          ← 控制器用的不是我们填的描述符表
OCS 0F 0F 0F 0F                        ← 传输仍未完成
```

对比 r13–r16（先复位再重建）时读到的 `TL=0 RL=0`：**"不要推倒重来"是对的**——只要原样
接管，UniPro 的数据通路就是活的。剩下 `OCS=0f` 且 `UCD=FFFFFFFF` 指向一个很具体的
UFSHCI 硬规则：

> `UTRLBA`/`UTMRLBA` **只能在对应的 run-stop 寄存器为 0 时写入**。LK 把两个列表都留着
> 运行（`RSR=1`），于是通用驱动写基址被硬件忽略，控制器继续走 LK 的旧描述符表 →
> 我们的 UTRD/UCD 永远不被执行。

**（二）expdb：把"拍屏猜日志"换成"读完整日志"**

adb 可用时（设备跑 Android/主线内核），可以拉出 expdb 分区：LK 在每次启动早期会把
pl/lk 日志环（0x7FFBF000）转存到该分区最后 2 MiB，`port/tools/parse-expdb.py` 能把它
拆成一次启动一个 session。**实测已拉到 12 次启动的完整 preloader+LK 日志**，里面就有
LK 自己跑通的 UFS 现场：

```
[UFS] ufs_aio_init_mphy: ufs res0 value=0x801c5003
[UFS] UFS version 3.10            ← 控制器是 UFSHCI 3.1
[UFS] ufs id: KLUDG4UHDC-B0E1     ← Samsung 器件
[UFS] info: RX HS-G4, setting is 4
[UFS] info: TX HS-G4, setting is 4
[UFS] info: HS-G4-2, 1 lane       ← LK 最终把链路跑在 HS-G4 / 1 lane（FAST）
```

于是 r18 顺手把 UEFI 侧的日志也接进同一个环：新增
`chopinPkg/Library/LogStoreSerialPortLib`（替换 `SerialPortLib`），它按内核已验证的
协议（`logstore_set_length()`：写 ring 后刷新 SRAM 头 `0x4c/0x20/0x5c` 与环内 `0x18`
四个长度字段）把 **所有 `DEBUG()` 输出**写进去。这样下一次启动 LK 会把它转存进 expdb，
**连异常 dump 都能完整读回**，不必再拍照。代价是屏幕上不再有 DEBUG 文本（只剩标记方块
和菜单）；把 DSC 里那一行换回 `FrameBufferSerialPortLib` 即可恢复。

**r18 的两处修改**：

1. 接管模式下，在通用驱动重编程之前**先把两个 UTP 列表停掉**
   （`UTRLRSR(0x60)=0`、`UTMRLRSR(0x80)=0`），让基址写入真正生效；
2. `LogStoreSerialPortLib` + 内存映射新增 `SRAM Log Header (0x11DF00)`，
   UEFI 日志落进 expdb 通道。

读取方式（设备已连 adb 时）：

```sh
tools/pull-expdb.sh                     # 或
adb exec-out 'su -c "cat /dev/block/by-name/expdb"' > work/expdb/pull.img
python3 tools/parse-expdb.py work/expdb/pull.img --session <最新> -n 20000
```

### 3.18 r18 黑屏的教训：日志环只能在 DXE 之后写

r18 把 `SerialPortLib` 换成了写日志环的实现，上机结果：**第一屏之后直接黑屏、连绿块都没有**。
用 expdb 一查就清楚了：

* 拉回来的 dump 里 **完全没有 UEFI 的痕迹**（`grep -a "UFS:\|ASSERT\|DXE"` 全空）；
* 新增的 session #12 只有 7 KB、且只有 preloader 内容（用户长按电源重启那一次）。

原因：`SerialPortLib` 在 **SEC/PEI 就会被调用**，而环（0x7FFBF000）与 SRAM 头
（0x11DF00）是 **PEI 的内存映射**才建立的——`SEC` 里第一次 `DEBUG()` 就写到了未映射地址，
数据异常直接把机器打死，所以环里一个字都没留下。

**r19 的修正**：把"写环"从全局 `SerialPortLib` 移到 **UFS 驱动内部（DXE，映射已完整）**：

* `SerialPortLib` 恢复为 `FrameBufferSerialPortLib` → **屏幕日志回来**（便于即时观察）；
* 驱动里新增 `MtkUfsTrace()`：同一条日志既打屏、也追加进日志环，并按内核验证过的协议
  刷新 SRAM 头长度字段。目前接进去的是**全部 poll 行 + stage/失败行**——也就是我最需要的
  那几行；
* r18 的 UFS 修正（先停 UTRL/UTMRL 再让驱动写基址）保留。

于是 r19 的调试闭环变成：

```
刷 r19 → 启动（屏幕照旧能看到 poll 行）→ 长按电源重启
      → LK 把环转存进 expdb → 设备回到 Android/adb
      → adb 拉 expdb → parse-expdb.py --session N   ← 即使屏幕读不清也有完整数据
```

> 顺带记录一条**硬件参考数据**（来自 expdb 里 LK 自己的 UFS 日志）：
> 控制器 **UFSHCI 3.10**、器件 **Samsung KLUDG4UHDC-B0E1**，LK 最终把链路跑在
> **HS-G4 / 1 lane（FAST）**，M-PHY 初始化时 `ufs res0 = 0x801c5003`。

### 3.19 r19 卡在 MMU：新增区域必须 4 KiB 对齐

r19 上机第一屏就停住，屏幕写着：

```
UpdateRegionMapping RegionStart: 0x11DF00 or RegionLength: 0x1000 are not page aligned!
Failed to Configure MMU! Status = Invalid Parameter
```

我给 SRAM 日志头加的内存区域起点写成了 `0x11DF00`（SRAM 头的真实地址），但
`ArmConfigureMmu()` 要求每个区域的起点和长度都按 4 KiB 对齐。改成覆盖同一页的
`0x11D000`（长度 0x1000，不与 `0x11E10000` 的 `MSDC Top-1` 冲突）后，
`tools/check-chopin-uefi-memory-map.py` 重新通过。

**教训**：MemoryMapLib 里任何新条目的 `Address`/`Length` 都必须是 4 KiB 的整数倍，
"设备寄存器在某个非对齐地址"要靠"覆盖它的那一页"来表达，而不是直接写寄存器地址。

（附：这次 r19 死得比 DXE 更早，所以 expdb 里同样没有 UEFI 痕迹——环写入要在
MMU 建好、进入 DXE 之后才会发生，符合 3.18 的结论。）

### 3.20 一个重要的自我纠错：`OCS`/`UCD` 要在传输刚结束时读

r20 上机结果与 r17 一致（`st=10 blk=0`、`TL=RL=1`、`HCS=0xf`、`IS=0x401`、
`RSR=1`、`DBR=0`），说明"先停表再写基址"并没有改变结局。回头看才发现自己一直在读
**已经失效的数据**：

* `UfsPassThruDxe` 的 `Start()` 失败时会 `FreePool` 掉 UTRL/UCD 那些缓冲区；
* 我的 poll 是**事后**（BDS 空闲时）才去读 `UTRLBA` 指向的描述符 → 内存早已被回收/复用，
  所以 `OCS=0f`、`UCD=FFFFFFFF` 是**垃圾值**，不代表真实完成状态。

**有效的观测点**是：阶段标记、`blk`（BlockIo 数量）、以及控制器寄存器
（`HCS`/`IS`/`RSR`/`DBR`）。r21 因此补了两条关键测量：

| 新增 | 位置 | 为什么 |
| --- | --- | --- |
| `UFS xfer#n UBA=.. DBR=.. OCS xx xx xx xx IS=..` | **`Unmap` 钩子内部**（驱动每次传输的退出路径） | 此刻描述符**还有效**，读到的是真实 OCS：`00` = 成功、其它 = UFS 错误码 |
| `UFS ring off_lk=.. len=.. cur=.. sram_res=..` | poll | 回读 SRAM 日志头的长度字段，判断"写环"到底有没有生效 |

**expdb 复核**：三次拉取的"最后 2 MiB"哈希都不同 → LK 每次确实在转存；但转存区里
**没有**我们写的文本，且原厂内核不提供 `/dev/mem`（`devmem` 报 `No such file`），
无法从 Android 直接验证 SRAM 头。所以先靠上面那条 `UFS ring` 回读自证。

### 3.21 参考 MT6995/MT6895 的成功移植：两个真正的传输层修复（并纠正我之前的误判）

把 `https://github.com/mytiantian001/Mu-Silicium` 拉到 `work/refs/mu-mt6995`（290 MB）后
发现它带了一整套 **MTK 自己的 UFS 驱动**：
`Silicon/MediaTek/MT6895Pkg/Drivers/{MtkUfsDxe, MtkUfsPassThruDxe}`（MT6895 与 MT6893
是同代近亲）。它带来两个决定性结论：

**（一）我之前的 "DMA 地址编码" 补丁是错的，已撤销**

他们的 `UfsPassThruHci.c` 用的正是上游编码：

```c
Prdt[..].DbAddr = (UINT32)RShiftU64 ((UINT64)(UINTN)Remaining, 2);   // dword 单位
Trd->UcdBa      = (UINT32)RShiftU64 ((UINT64)CmdDescPhyAddr, 7);      // 128B 单位
```

也就是说硬件就是要这种"带位移的字段"编码；我在 3.9 里"改成原始地址"其实把传输打断了
（`Resources/MuPatches/Ufs-Dma-Addresses.patch` 已删除，三个文件 `git checkout` 还原）。

**（二）他们注释里记录的两个修复，正好对上我们的症状**（原文带 "Pearl:" 前缀）：

| 修复 | 原因（他们的注释） |
| --- | --- |
| **DM/query 命令 `Trd->Dd = UfsNoData`** | 设备管理命令不搬 PRDT 数据（flag/属性在 UPIU 事务字段里、描述符在 UPIU 数据段），必须用"无数据"方向；上游用数据包方向，会让控制器**期待一个数据阶段**，于是命令以 **PHY adapter / data link 错误失败，或干脆永不完成** |
| **敲门铃前强制 `UTRLRSR=1` + `MemoryFence()`** | 从 bootloader 继承的控制器状态不可靠（列表可能停着，停着的列表永远不会取 UTRD）；描述符在 non-cacheable 内存里，敲门铃前必须确保写入已到内存 |

这两个修复已做成 `Resources/MuPatches/Ufs-Mtk-Transfer-Fixes.patch`（2 hunk，已挂进
`build_uefi.py`）。r22 = 撤销错误补丁 + 这两个修复。

**（三）他们的平台层是"完整重初始化"**（不是我们 r17 试的"接管"）：
refclk 先起 → **通过 SMC 复位 UFS 设备**（`ufs_mtk_device_reset`）→ 每次 HCE 前开 CLK_CG →
关 auto-hibernate → HCE 0→1 循环 → **DME_RESET + DME_ENABLE** → UniPro 退出低功耗 →
关 host TX LCC → 链路启动。如果 r22 仍不成功，下一步就按这个顺序把我们的回调补齐
（包括 SMC 设备复位——他们的代码会检查 SMC 返回值，遇到 `SMC_UNK` 会明确报错）。

### 3.22 照 MT6895 成功案例补齐初始化序列（r23）

r22 上机后状态**比之前更早**：poll 行显示 `st=6`（PreLinkStartup），而且按键触发重连后
标记从"少 2 格"变成"少 6 格"——同一个启动里 UFS 栈被初始化两次、第二次失败得更早。
这正好印证参考实现里的注释：**从 bootloader 继承的控制器状态是不可靠的**。

对照 `work/refs/mu-mt6995/Silicon/MediaTek/MT6895Pkg/Drivers/MtkUfsDxe`，我们一直缺这
几步（他们的代码注释逐条说明）：

| 步骤 | 为什么必须 |
| --- | --- |
| **refclk 在 HCE 循环之前起** | 参考时钟默认被门控，链路没它起不来 |
| **通过 SMC 复位 UFS 设备**（`MTK_SIP_UFS_CONTROL=0xC2000276`, `UFS_MTK_SIP_DEVICE_RESET=0x2`；RST_n 低、HCE 置 0、≥10 µs、释放、15 ms 等待） | mainline 在 `ufshcd_init()` 里总做这一步；设备留在异常状态时，链路能起但 UTP 永远不通 |
| **每次 HCE 前开 CLK_CG**（`XOUFS_CTRL bit7`） | HCE 循环会清掉它；不开会触发 ERR_CHK 让控制器无法拉起链路 |
| **HCE=0 并轮询到真的读到 0** | 通用驱动只是写 HCE 就往下走 |
| **PostHce 等 ~600 µs** | 控制器自己的初始化（HCE 1→0→1）需要时间，通用驱动会读到过期的 1 |
| **不要自己发 DME_RESET/DME_ENABLE** | 那只适用于 `UFSHCI_QUIRK_BROKEN_HCE`（如 Exynos）；这个控制器在 HCE=0 时 UIC 引擎不完成命令 |
| **完全不要碰 M-PHY** | 依赖 bootloader 已初始化的 PHY（恢复设置需要 ATF 的 `UFS_MPHY_RESTORE`，UEFI 拿不到） |

r23 把 `MTK_UFS_TAKEOVER_FROM_LK` 置 0（回到完整重初始化），并按上表重写了 PreHce/
PostHce，同时保留 r22 的两个传输层修复（DM 命令 `Dd=UfsNoData`、敲门铃前强制
`UTRLRSR=1` + `MemoryFence()`）。

如果 r23 仍不出盘，下一步就是把参考的 `MtkUfsDxe` + `MtkUfsPassThruDxe` **整套移植**
过来（它们是可直接替换的驱动，只需改 MMIO 基址与时钟 ID），而不是继续逐条猜。

### 3.23 真正的根因线索：LK 在交接前把 UFS **关掉**了

r23（完整重初始化）上机后：`HCS=0x00000008` → **只有 UCRDY，DP（设备在位）=0**，
`RSR=0`、`UBA=0`、`TL=RL=0`，`st=6` —— 链路这次根本没起来（停 PreLinkStartup 之前的
link startup）。

回头翻 LK 的 expdb 日志（`work/expdb/sess11b.txt`）看到一行关键内容：

```
[UFS] notify power-off success
```

**LK 在跳转到 payload 之前把 UFS 关掉了。** 这一下把之前所有矛盾都解释通了：

| 观察 | 解释 |
| --- | --- |
| 接管模式（r17–r22）`DP=1`、UIC/DME 全通，却**一条 UTP 传输都不完成** | 链路层还活着，但**设备已被 power-off**，UTP 请求发出去没人应答 |
| 完整重初始化但**跳过 M-PHY**（r23）`DP=0` | 设备复位把链路拆了，而 M-PHY 没上电 → 链路起不来 |
| 早期 r13–r16 有 M-PHY 上电、`DP=1`，但 `TL=RL=0` | 那时缺 refclk/设备复位/CLK_CG 的顺序，数据通路没建起来 |

于是正确配方应当是三件事**同时**具备：

1. **用 SMC 把设备唤醒**（`MTK_SIP_UFS_CONTROL=0xC2000276` / `UFS_MTK_SIP_DEVICE_RESET=0x2`，
   RST_n 脉冲 + 足够长的恢复时间）；
2. **给 M-PHY 上电**（已是"开着就跳过"的幂等实现，绝不复位 PHY）；
3. **两个传输层修复**（DM 命令 `Dd=UfsNoData`、敲门铃前强制 `UTRLRSR=1` + `MemoryFence()`）。

r24 = r23 的序列 + (2) M-PHY 上电 + SMC 返回码日志 + 复位后等待延长到 200 ms。

### 3.24 r24 仍无链路；并把初始化结果回放到 poll 行（r25）

r24（SMC 设备复位 + M-PHY 上电 + 传输层修复）上机结果与 r23 相同：
`HCS=0x8`（只有 UCRDY，**DP=0**）、`RSR=0`、`UBA=0`、`TL=RL=0`、`st=6` —— 链路仍没起来。

**expdb 通道复核**：这次完整拉到 40 MiB（此前 1 MB 是传输被截断；另外原厂 Android 上
`/data/local/tmp` 对 root 也不可写，只能用 `adb exec-out 'su -c "cat …"'` 这条路）。
但整份 dump 里**依然没有我们的行**（`UFS: device reset`/`UFS st=`/`UFS init` 全部命中 0），
说明 LK 的转存只覆盖 preloader/LK 那段。写到 0x20000 的环内容不在其中，需要再逆几轮
才能当稳定通道用，**先挂起**。

于是 r25 把初始化结果也做成"每秒回放"的一行，这样一张照片就能看到关键返回值：

```
UFS init SMC=<assert>/<deassert> mphy8C=<...> HCS=<...> XOUFS=<...>
```

判读：SMC 返回 `0` 或非负 = SIP 成功；返回 `0xFFFFFFFF`（SMCCC "not supported"）说明
ATF 没有实现 `0xC2000276`，那"设备复位"这条路在 chopin 上就是死路，需要换唤醒方式。

### 3.25 决定性对比：我们自己把一条**通着的链路**打断了

r25 的视频里抓到同一启动的两个时刻：

```
（早期，UfsHcDxe 刚跑完）  UFS st=1 blk=0 HCE=00000001
                          UFS HCS=0000000f     ← bit0 DP=1，链路是通的！

（最终，完整重初始化之后）  UFS st=6 blk=0
                          UFS HCS=00000008     ← 只剩 UCRDY，DP=0，链路被打断了
```

结合 LK 日志里的 `[UFS] notify power-off success`，模型就清楚了：

> **LK 交接时链路是通的，但设备处于 power-off/休眠状态。** 我们 r23/r24 的"完整重初始化"
> （SMC 设备复位 + HCE 循环）把这条已经工作的链路拆掉了 —— 这是**方向性错误**。

r26 因此：**恢复接管模式**（`MTK_UFS_TAKEOVER_FROM_LK=1`，不复位、不碰 M-PHY、不重发
链路启动），只在 PreLinkStartup 时发一条 **`DME_HIBERNATE_EXIT`（UIC opcode 0x18）**
把设备从休眠里叫醒——这是**在不破坏链路的前提下**唯一还能用的唤醒手段。

同时 poll 新增一行，一张照片即可看到全部关键值：

```
UFS init SMC=<assert>/<deassert> mphy8C=<...> HCS=<...> XOUFS=<...>
```

### 3.26 r26（接管 + hibernate exit）结果与下一步决策

r26 上机后（视频放大帧 `work/device/uefi-video-2026-10-17-r26/`）：

```
UFS init SMC=.../... mphy8C=... HCS=0x0000000F XOUFS=0x00000000
UFS DME  LCC=0 SPC=00000090 UPD=0
UFS DME  ... TXG=0 RXG=0 TL=1 RL=1
UFS UBA=... OCS 0F 0F 0F 0F
```

即：**链路在位（DP=1）、数据通道在（TL=RL=1）、UIC/DME 全通，`OCS` 仍是 `0f`** ——
`DME_HIBERNATE_EXIT` 没有让 UTP 传输开始完成。另外 `XOUFS_CTRL` 读回 **0**，说明
接管模式下 CLK_CG（bit7）不是我们置的（LK 的值也没留下），这一点与参考实现有出入。

到这里，"逐条补丁"这条路已经**平台化**（连续 r17→r26 都停在同一个现象）。结论：

> **下一步不再逐条猜，而是把参考仓库里已经验证的 `MT6895Pkg/Drivers/MtkUfsDxe` +
> `MtkUfsPassThruDxe` 整套移植到 `MT6893Pkg`**：它们是可直接替换的驱动（平台胶水 +
> 修好的传输实现），只需改 MMIO 基址、时钟 ID 和 SMC 号；这比继续在通用驱动上
> 打补丁更可能一次成型。

另一种取舍是把 UFS 暂时挂起，先交付目标里**不依赖 UFS** 的部分（ACPI 表骨架、MT6359
regulator、按键行为、显示双帧选择、启动链集成文档），之后再带着参考驱动回来打 UFS。

### 3.27 整套移植 MT6895 的 UFS 驱动（r27）

按参考实现的路子，把他们的两个驱动整体搬进 `MT6893Pkg`：

| 新增（来自 `work/refs/mu-mt6995`） | 作用 | 替换掉 |
| --- | --- | --- |
| `MT6893Pkg/Drivers/MtkUfsDxe/` | 平台驱动：init 时置 `XOUFS_CTRL` 的 CLK_CG(bit7)、请求 refclk、安装 `EDKII_UFS_HOST_CONTROLLER_PROTOCOL` + `EDKII_UFS_HC_PLATFORM_PROTOCOL` + DevicePath；平台回调按参考顺序（PreHce: refclk→SMC 设备复位→CLK_CG→HCE=0；PostHce: 600 µs；PreLinkStartup: UniPro 退 LPM / TX LCC=0 / 关 deep stall） | **我们的 `UfsHcDxe`（已从 FV 移除）** |
| `MT6893Pkg/Drivers/MtkUfsPassThruDxe/` | 传输驱动（参考修好的 fork：DM 命令 `Dd=UfsNoData`、敲门铃前强制 `UTRLRSR=1`+`MemoryFence()`，并新增 `UfsDeviceConfig` 协议） | **MdeModulePkg `UfsPassThruDxe`（已从 FV 移除）** |

配套改动：
* `MediaTekPkg.dec` 新增 `gMediaTekPkgTokenSpaceGuid.PcdUfsHostControllerBase`（token `0x0000E101`）；
* `chopin.dsc` 设 `PcdUfsHostControllerBase|0x11270000`，`[Components]` 换成两个新驱动；
* `Include/APRIORI.inc` 与 `Include/DXE.inc` 同步替换（FDF 通过 `!include` 引用这两个文件）。

**验证**：构建日志中 `UfsHcDxe` 出现 **0 次**，`MtkUfsDxe`/`UfsPassThruDxe` 各出现 6 次
（编译/链接/拷贝），说明 FV 里已只有新驱动；`ScsiBusDxe`/`ScsiDiskDxe` 保留不变。

> 与之前最大的行为差别：新驱动在**驱动 init 阶段**就把 CLK_CG 置上并请求 refclk——
> 而 r26 的接管模式读回 `XOUFS_CTRL=0`（从未置位）。参考实现的注释说这一步用来
> "bypass abnormal ERR_CHK，防止 host hang"，很可能正是 UTP 始终不完成的关键。

### 3.28 **UFS 打通了**（r27 = 整套移植参考驱动）

r27 上机后的日志（视频放大帧 `work/device/uefi-video-2026-10-17-r27/hit.png`）：

```
MtkUfsDxe: AllocateBuffer(0x46E3A000, 1 pages) normal-nc: Success
MtkUfsPassThru: ring slot 0 UTRLRSR=0x1 HCS=0xf      ← 传输真的在跑（HCS 里 DP+UCRDY 都在）
InstallProtocolInterface: CE345171-BA0B-11D2-8E4F-00A0C969723B   ← BlockIo
BlockSize : 4096
LastBlock : 10A7FF                                    ← ScsiDisk 读到了真实容量
```

也就是说：**链路在位、UTP 请求完成、BlockIo/ScsiDisk 枚举出多个 LUN**（帧里能看到
`LastBlock` 为 `3FFFF` / `6E7FF` / `10A9FFF` / `FF` 等多个值）。之前 26 轮卡住的问题，
根源就是**上游通用 UFS 实现 + 我们自研平台胶水的组合**；换成 MT6895 移植验证过的
`MtkUfsDxe` + `MtkUfsPassThruDxe`（并在 init 阶段就置 `XOUFS_CTRL` 的 CLK_CG、请求
refclk）后一次通过。

唯一一条报错是 `UfsExecScsiCmds() fails with Target Failure (LUN 128, CDB 00 00 00 00 00 00)`
——LUN 128 (0x80) 是 UFS 的**设备管理 well-known LUN**，不少器件会拒绝这条 SCSI 命令，
属预期内、可忽略。

**结论**：UEFI 侧的存储栈（UFS → ScsiBus → ScsiDisk → BlockIo）已经可用，接下来才是
"从 UFS 启动内核"的启动链集成。

### 3.29 按键后的现象：BDS 在反复扫描 UFS（正常），日志把菜单刷没了

按键后的视频里，日志成串重复：

```
MtkUfsDxe: AllocateBuffer(0x46E5B000, 1 pages) normal-nc: Success
MtkUfsPassThru: ring slot 0 UTRLRSR=0x1 HCS=0xf
BlockSize : 4096
LastBlock : 7FF
```

这是 **BDS 每次刷新启动项都会重新检测介质**（ScsiDisk 的 `DetectMedia`）——盘在位、可读，
属正常行为。问题只在于 `PcdDebugPrintErrorLevel` 打开了 `DEBUG_INFO`，控制台被刷满、
看不到菜单。

r28 把该 PCD 恢复成 SiliciumPkg 默认（`0x8007EE0F`，只打 WARN/ERROR）——
UFS 已工作，不再需要 INFO 级别的驱动日志；屏幕应当能以较干净的状态显示
boot manager 与启动项。

### 3.30 从 UFS 启动 Linux 的方案（进行中）

UFS 打通后，"从 UFS 启动"就变成纯软件问题。chopin 上没有 ESP（`MsBootPolicyLib` 报
`EFI_NOT_FOUND` 的原因），所以按主线移植的经验自己加载内核：

**阶段 1（本轮实现）**：新增 `Platforms/Xiaomi/chopinPkg/Drivers/LinuxBootDxe`
（`DXE_DRIVER`，`[Depex] gEfiBlockIoProtocolGuid`，因此会在 ScsiDisk 产生 BlockIo 之后
才被 dispatch）：

* 枚举所有 `BlockIo` 句柄 → 读 MBR/GPT → 按名字找 `boot_a`/`boot`/`boot_b`；
* 解析 Android boot image 头（magic `ANDROID!`、`header_version`、`page_size`、
  kernel/ramdisk/dtb 的 size+addr、cmdline）；
* 把 kernel/dtb/ramdisk 读进 `AllocatePages` 得到的内存（kernel 需 2 MiB 对齐），
  检查 ARM64 `Image` 头（0x38 处 `ARM\\x64`）与是否 gzip；
* **只报告、不跳转**，用 `DEBUG_ERROR` 打印全部结果（`DEBUG_INFO` 已关闭）。

**阶段 2（下一步）**：真正交接——`ExitBootServices` 之后 `ArmDisableCachesAndMmu`
（`MdePkg/Library/ArmLib/AArch64/AArch64Support.S` 里已有），`x0=dtb` 后跳到内核入口。

**内核载荷的存放**（已验证可行）：

| 项 | 值 |
| --- | --- |
| 原始 `Image` | 44.4 MB（`boot_a` 只有 16 MiB，且里面是我们的 UEFI，放不下） |
| 压缩方式 | 树里**没有** zlib/inflate，但自带 `LzmaCustomDecompressLib`（`LzmaUefiDecompress`） |
| LZMA 压缩后 | **11.1 MB** ✓ 可放进 16 MiB 的 `boot_b` |
| 已生成的载荷 | `work/Image-chopin-raw`、`work/Image-chopin-raw.lzma` |

计划：把内核放到 **`boot_b`**（a/b 槽的备用槽），加载器扫分区找 boot image/raw Image，
阶段 2 用 LZMA 解压后交接。

### 3.31 r29：自己的启动菜单 + 内核加载器（阶段 1）

**（一）菜单终于能出现了**

r28 的照片显示 `...Booting Internal Storage` + `MsBootPolicy.efi`——说明
`BootManagerMenuApp`（`PcdBootManagerMenuFile` 指向的那个应用）**画完 Si logo 就退出了**，
菜单列表从未出现，BDS 于是直接按启动顺序去启动 "Internal Storage"。根因是
`Resources/MuPatches/Boot-Manager.patch` 让它通过 HII 构建条目列表，在这个平台上初始化
不成功。

r29 用自己写的极简菜单替代它：`Platforms/Xiaomi/chopinPkg/Applications/BootMenuApp/`

* 纯 `ConOut` 文本绘制，**不依赖 HII / 字符串包 / GOP 逐像素绘制**；
* 列出 BDS 已知的**全部启动项** + 一个 `Power Off`；
* **任意按键 = 切到下一个条目**，**5 秒无按键 = 启动选中项**——这样只有音量键也能完整操作；
* 把 `PcdBootManagerMenuFile` 指向它（GUID 字节 `31 9C 7A 5B …`），并把
  `PcdPlatformBootTimeOut` 从 3 s 提到 **10 s**。

**（二）`LinuxBootDxe`（阶段 1）**

`Drivers/LinuxBootDxe`（`DXE_DRIVER`，`[Depex] gEfiBlockIoProtocolGuid`，1286 行）：

* 枚举 BlockIo → MBR/GPT（LBA 尺寸在 {BlockSize,512,4096} 里探测，表项严格校验）
  → 按名字找 `boot_a`（回退 `boot`/`boot_b`）；
* 读 8 KiB 解析 Android boot image（v0/v1/v2：page_size、kernel/ramdisk/second/dtb 的
  size+addr、cmdline）；
* 把 kernel（2 MiB 对齐）/ramdisk/dtb 读进内存，报告 ARM64 `Image` magic（0x38）与是否
  gzip；**不跳转**，全部结果用 `DEBUG_ERROR` 打到屏幕上。

**（三）已知的下一步**

当前 `Mu-chopin.img` 作为 boot image 打包时 `append_dtb=false`，所以加载器会报
"zero sized DTB"；真正启动 Linux 前需要把 DTB 一起放进 `boot_b` 的镜像里
（`append_dtb=true`），或者由加载器去读 `dtbo_a`/`dtb` 分区。阶段 2 再做
`ExitBootServices` → `ArmDisableCachesAndMmu` → `x0=dtb` 的交接。

### 3.32 r30：菜单必须排在启动顺序第一，否则永远看不到

r29 上机：**有 Si logo、随后黑屏、没有菜单**。查 `PlatformBootManagerLib` 后明白了两件事：

* **Si logo 是 `PlatformBootManagerAfterConsole()` 画的**（`DisplayBootGraphic(BG_SYSTEM_LOGO)`），
  与那个菜单程序无关；
* 之后 BDS **按启动顺序自动启动第一个选项**——`MsBootOptionsLib` 里第一个注册的是
  "Internal Storage"（一个扫描介质的策略应用），所以屏幕黑掉、卡在扫描里，
  根本没有机会显示菜单。`PcdBootManagerMenuFile` 只决定"平台要启动哪个菜单应用"，
  并不会把它排在启动顺序前面。

r30 的修法：在 `MsBootOptionsLibRegisterDefaultBootOptions()` 的**最前面**注册菜单本身：

```c
RegisterFvBootOption (FixedPcdGetPtr (PcdBootManagerMenuFile), L"Boot Menu", (UINTN)-1, LOAD_OPTION_ACTIVE, NULL, 0);
```

于是启动顺序变成 **Boot Menu → Internal Storage → USB Storage …**，BDS 一上来就进菜单，
菜单里再选"Linux/存储/Power Off"。配合 r29 的 `BootMenuApp`（任意键切换、5 s 无按键启动
选中项），单靠音量键就能完整操作。

### 3.33 r31：菜单被启动了但黑屏——加"分步日志"定位

r30 的截图证明 **`BootMenuApp` 确实被 BDS 启动了**（`...Booting BootMenuApp` +
`BootMenuApp.efi`），说明"注册成启动顺序第一项"是对的；但屏幕随即黑掉，说明应用在
**画第一行之前**就出问题了（最可能是 `gST->ConIn` 仍处于懒连接状态，或 ConOut 的
写没有落到 framebuffer）。

r31 给 `BootMenuApp` 加了**分步 `DEBUG_ERROR` 日志 + 控制台空指针保护**：

```
BootMenuApp: entry ConIn=%p ConOut=%p
BootMenuApp: %u load option(s)
BootMenuApp: %u menu item(s)
BootMenuApp: drawing the menu now
BootMenuApp: menu drawn, waiting for input
```

这样下一张照片就能判定卡在哪一步：

| 屏幕现象 | 结论 / 下一步 |
| --- | --- |
| 一行都没有 | 应用在入口前就没跑起来（FV 里应用损坏？）→ 换直接 GOP 绘制 |
| `entry ConIn=00000000` | **ConIn 懒连接是黑屏原因** → 在应用里显式 `ConnectController` 控制台 |
| 走到 `menu drawn` 但屏幕仍黑 | ConOut→GraphicsConsole 的写没落到这块 framebuffer → 改成直接画 GOP |
| 出现菜单 | 成功，继续做阶段 2（加载器跳转） |

### 3.34 r32：菜单改用调试控制台绘制（ConOut 在这台机器上不出字）

r31 的截图很关键——应用**跑到 `drawing the menu now` 就停在那里**，之前的三行也都正常：

```
BootMenuApp: entry ConIn=0x... ConOut=0x...
BootMenuApp: 4 load option(s)
BootMenuApp: 5 menu item(s)
BootMenuApp: drawing the menu now          ← 到这里之后屏幕上什么都没有
```

说明 **`EFI_SIMPLE_TEXT_OUTPUT` 的写根本不显示在这块屏上**：屏幕上所有可见文字都来自
`FrameBufferSerialPortLib`（DEBUG 通道直接写 framebuffer），而 Si logo 是
`DisplayBootGraphic()` 用 **GOP 直接画**的——也就是说这个平台的 `ConOut` 里并没有
GraphicsConsole 这个设备，菜单用 ConOut 画等于画到看不见的地方。

r32 因此把菜单**改用 DEBUG 通道打印**（`MenuPrint` → `DEBUG((DEBUG_ERROR, ...))`），
并且**每次按键后追加一行选择状态**（不做清屏，因为清屏也走 ConOut）：

```
BootMenuApp: ================ chopin boot menu ================
BootMenuApp: ==> [0] Boot Menu
BootMenuApp:     [1] Internal Storage
...
BootMenuApp: selected: [0] Boot Menu
```

这份输出**一定可见**（其它所有驱动日志都是这么显示出来的）。等键盘/超时逻辑验证后，
再决定要不要换成 GOP 直接绘制以获得"可高亮刷新的真菜单"。

### 3.35 r33：菜单按键轮询 + 彻底消除刷屏

r32 菜单已经可见，但有两个问题：**屏幕一直重复打印**、**按键无反应**。

* 刷屏的根因：原先用 `gBS->WaitForEvent (2, {ConIn.WaitForKey, Timer})`，而这个平台的
  ConIn 是**懒连接**的（BDS 那句 `Connect ConIn in first WaitForKey in Lazy ConIn mode`），
  事件一旦无效就立刻返回错误 → 我的代码 `break` 出去 → 应用**返回** → BDS 按启动顺序
  又把它当启动项跑一遍 → 菜单块无限重复。
* 按键无反应：同样源于没有真正触发/使用 ConIn 的输入路径。

r33 改成"**轮询 + 永不返回**"：

```
入口先做一次 ReadKeyStroke（触发懒连接，忽略结果）
MenuDraw() 打印整块菜单一次
循环（每 30 ms）：
    ReadKeyStroke 成功 → 选中项 +1，追加一行 "selected [N] ..."（不清屏）
    连续 8 s 无按键 → MenuBoot() 启动选中项一次，随后打印菜单并等按键
```

要点：
* 应用**永不返回**（不会退出后又被 BDS 重新启动）→ 刷屏消失；
* 每次按键只追加**一行**（不做整屏重绘，因为清屏同样走 ConOut 不可见）；
* 启动失败后**阻塞等按键**，不会自动重试成环；
* 音量上/下、电源键都通过 ConIn 的 `ReadKeyStroke` 读取（任意键 = 下一个），
  单键设备也能完整操作。

### 3.36 r34：按键去抖（菜单可用）

r33 的现象：菜单只打印一次 ✓、按键确实被读到 ✓，但**一次按键会连读多次**，`selected` 一路
递增刷屏（照片里能看到 0→1→2→3→0 的快速循环）。

原因：EDK2 的 `SimpleTextIn` **没有按键重复**机制，而平台的 keypad 在按住/释放过程中会
持续把同一个键投递进队列，所以每 30 ms 轮询都会"读到一次"。

r34 加了 **400 ms 去抖**：距上次接受按键不足 400 ms 就丢弃该键。

```
SinceKey += POLL_INTERVAL_US;
读到键 && SinceKey >= 400 ms → 接受（选中 +1，打印一行），SinceKey = 0
```

这样一次物理按键只走一步，长按也不会连跳；8 s 无按键仍然自动启动选中项。

### 3.37 r35：把原始键码打出来（按键/幻影事件定位）

r34 反馈：**不按键也会自动刷屏**（每 400 ms 一条 `selected`），按键要"特定组合"才生效。
说明平台的 keypad 把大量**非真实按键**投进了 ConIn 队列（按住/幻影事件），
而我的去抖只是把它们放慢了，没有过滤。

r35 做两件事：

1. 去抖加长到 **800 ms**；
2. **每接受一个键就把原始键码打出来**：

```
BootMenuApp: key sc=0017 uc=0000 -> selected [1] Internal Storage ...
```

这样下一张照片就能看清到底收到什么（`ScanCode` / `UnicodeChar`），据此只接受真实的
音量/电源键码，其余一律丢弃——甚至可能发现是 PMIC 的某个键状态被误读成"一直按下"。

### 3.38 r36：按键边沿检测（过滤幻影键）

r35 仍然"不按键就自动刷屏"→ 说明 ConIn 队列里有一个**恒定的幻影键**在反复投递。
r36 加**边沿检测**：只有键码与**上一次接受的不同**才处理，同一个键的连续重复一律丢弃。

```c
if (HaveLastKey && Key.ScanCode == LastKey.ScanCode && Key.UnicodeChar == LastKey.UnicodeChar) continue;
LastKey = Key; HaveLastKey = TRUE;
```

这样恒定幻影键被彻底过滤（屏幕不再自动滚），而真实的音量/电源键（键码不同）仍然
每次都能推进菜单；同时保留 800 ms 去抖与 `key sc=.. uc=..` 的原始键码打印，
万一还有第二类幻影键也能立刻从照片上看出来。

### 3.39 r37：幻影键是"上/下交替流"，改为"静默后才接受"

r36 的视频把键码打出来了：

```
BootMenuApp: stale key 0001
BootMenuApp: key sc=0002 uc=0000 -> selected [1] ...
BootMenuApp: key sc=0001 uc=0000 -> selected [2] ...
BootMenuApp: key sc=0002 uc=0000 -> selected [3] ...
（0002/0001 交替，永不停止）
```

即 `ScanCode 1/2`（标准 EFI 的 Up/Down）在**无限交替投递**——这是平台 keypad 驱动
（`KeypadDeviceLib` + `KeypadDxe`，读 PMIC/GPIO）本身的缺陷，不是菜单逻辑。边沿检测
因此失效（每次键码都"不同"）。

r37 改为**"队列静默才接受"**：只有连续 3 次轮询（约 90 ms）都读不到键之后，才接受下一次
按键；幻影流没有空档，因此被完全过滤——屏幕不再自动滚，`selected` 保持不动。

（r37 上机确认：屏幕**不再自动滚**，只显示 Si logo + 静态菜单文字 ✓。副作用见下。）

> 副作用（诚实说明）：如果幻影流一直不停，真实按键也会被这个过滤器挡住，菜单只能靠
> **8 s 空闲自动启动**。要真正恢复按键，需要在平台层修 keypad：直接读 **MT6359
> `PWRKEY` 状态（reg 0x434 bit0）** 与 **音量键 GPIO**，绕开现在这个重复投递的路径。

### 3.40 阶段 2：从 UFS 启动 Linux（r38 完成）

**载荷已就绪**：`work/boot_b-chopin-test.img`（11 309 056 B，可放进 16 MiB 的 `boot_b`）：

| 字段 | 值 |
| --- | --- |
| magic / header_version / page_size | `ANDROID!` / 2 / 2048 |
| kernel | `Image-chopin-raw.lzma`（LZMA 压缩，11 101 545 B，头字节 `5d 00 00`） |
| dtb | `Resources/DTBs/chopin.dtb`（201 848 B） |
| ramdisk | 4 B 占位（本次只是验证"UEFI → Linux"能跳过去，没有 initramfs） |
| cmdline | `bootopt=64S3,32N2,64N2 console=tty0 earlycon=mtk8250,mmio32,0x11002000,115200 …` |

> 树里没有 zlib/inflate，所以内核用 **LZMA**，由 EDK2 自带的
> `LzmaCustomDecompressLib` 在加载器里解压；`console=tty0` 让内核一启动就把
> 日志打到我们这块 framebuffer 上，跳转成功与否肉眼可辨。

**加载器的阶段 2**（已实现，r38 内含）：

1. 识别 LZMA（`0x5D` 起 + 8 字节解压后长度）→ `LzmaUefiDecompress` 解到 **2 MiB 对齐**的缓冲；
2. 校验 ARM64 `Image` magic（0x38 处 `ARM\x64`）与 `text_offset`（0x08）；
3. 校验 DTB magic（`0xD00DFEED`）；**没有 DTB 就不跳**（当前 boot image 若 `dtb_size=0` 会明确报错退出）；
4. 刷 cache → `ExitBootServices`（失败重试一次）→ `ArmDisableCachesAndMmu` →
   `x0=dtb, x1=x2=x3=0` → `br <entry>`（新增 `LinuxBootJump.S`）。

**载荷已用裸 FDT 重建**（包装 DTB 解出：offset 64，FDT magic `d00dfeed`，totalsize 201784；
其 `/chosen/bootargs` 自带 `console=tty0`，所以内核日志会直接打到面板上）：

| 项 | 值 |
| --- | --- |
| 文件 | `work/boot_b-chopin-test.img` |
| 大小 / sha256 | 11 309 056 B / `00a5e54eda4ddb31066a103a5e96ee0ce26c17fc19521e357a73a3a445769552` |
| 跳转条件 | LZMA 成功、`Image` magic `0x644D5241`、`text_offset` 合理、DTB 存在且 magic `0xD00DFEED`、`ExitBootServices` 两次内成功 |

**实现要点**（`LinuxBootDxe` + 新增 `LinuxBootJump.S`）：

* `LzmaCustomDecompressLib` 是 `LIBRARY_CLASS = NULL`，只能在 DSC 里用 `NULL|<inf>` 强制实例化；
  其构造函数会注册 guided-section handler，因此该模块同时把 `ExtractGuidedSectionLib` 换成 DXE 版
  （平台默认的 PrePi 版会写 HOB，DXE 阶段不安全）；
* `LzmaUefiDecompress` 第 4 个参数是 64 KiB **scratch**，所以先 `LzmaUefiDecompressGetInfo` 再解压；
* 解压目标 2 MiB 对齐，入口 = `KernelBase + text_offset`；
* 交接顺序：刷 cache → 摘要行 → `ExitBootServices`（失败重试一次）→ `ArmDisableCachesAndMmu`
  → `LinuxBootJumpToKernel(x0=DTB, x1=entry)`（汇编里清 x1/x2/x3 后 `br`）。

> 注意：`LinuxBootDxe` 的 depex 是 `BlockIo`，所以它会**在 DXE 派发阶段自动启动 Linux**，
> 菜单/倒计时会被绕过——当前阶段这正合适；以后想用菜单选择启动时再改成按需触发。

**用户侧操作**：

```sh
fastboot flash boot_b /home/kily/project/mt6893/work/boot_b-chopin-test.img
fastboot flash boot   /home/kily/project/mt6893/port/dist/chopin-uefi-DEBUG-ufs-r38.img
```

### 3.41 r39：菜单要先派发掉加载器（否则永远轮不到它）

r38 上机：**菜单正常显示、但完全没有 `LinuxBoot:` 行**——加载器根本没跑。

原因：`LinuxBootDxe` 的 depex 是 `gEfiBlockIoProtocolGuid`，它要等 BDS 在
`EfiBootManagerConnectAll()` 之后调用 `gDS->Dispatch()` 才会被派发；而我们的
`BootMenuApp` 是**启动顺序第一项**、又**永不返回**，于是"菜单先跑、派发永远轮不到"。

r39：`BootMenuApp` 启动时先自己调一次 `gDS->Dispatch()`（`PiDxe.h` + `DxeServicesTableLib`），
把 depex 已满足的驱动（也就是 Linux 加载器）派发掉：

```c
if (gDS != NULL) {
  DEBUG ((DEBUG_ERROR, "BootMenuApp: dispatch -> %r\n", gDS->Dispatch ()));
}
```

如果 `boot_b` 里的载荷有效，加载器会在菜单出现前就完成"读盘 → 解 LZMA → 校验 →
`ExitBootServices` → 跳转"，菜单根本不会出现——这正是我们要的自动启动行为。

### 3.42 r40：加载器改成"菜单里选择的启动项"（按 MT6895 的做法）

r39 上机：`BootMenuApp: dispatch -> Not Found`——派发队列里没有可派发的驱动，加载器
始终没跑（`BlockIo` 的 depex 在那个时刻并不满足，而菜单又是永不返回的启动项）。

r40 按参考实现的常规做法，把加载器从"等 depex 的驱动"改成**平台应用 + 启动项**：

| 改动 | 内容 |
| --- | --- |
| `LinuxBootDxe.inf` | `MODULE_TYPE = UEFI_APPLICATION`，去掉 `[Depex]`，`UefiDriverEntryPoint` → `UefiApplicationEntryPoint` |
| `MsBootOptionsLib.c` | 注册启动项 **"Linux (UFS)"**（GUID `C10F560B-F759-4F72-BA45-D54C5A5B96D8`），排在菜单之后 |

于是菜单里会出现 **`[2] Linux (UFS)`**，选中（或等 8 s 自动）它就会：读 `boot_b` →
解 LZMA → 校验 Image/DTB → `ExitBootServices` → `ArmDisableCachesAndMmu` → 跳转。

> 顺带说明：菜单的幻影键（`sc=0001/0002` 交替）仍在，音量上/下都能推进选择、
> 电源键无反应——这块要等 keypad 的读取路径修好（直接读 MT6359 `PWRKEY`/GPIO）。

### 3.43 r41：禁用菜单，直接启动 Linux

按要求撤掉菜单，让启动顺序变成 **Linux (UFS) → Internal Storage → USB Storage → …**：

* `MsBootOptionsLib.c` 删掉 `Boot Menu` 的注册；
* `Linux (UFS)`（加载器应用，GUID `C10F560B-…`）注册为**第一项**。

`BootMenuApp` 仍留在 FV 里但不再被启动。开机后 BDS 直接运行加载器：
读 `boot_b` → LZMA 解压内核 → 校验 `Image`/DTB → `ExitBootServices` →
`ArmDisableCachesAndMmu` → 跳转。屏幕上应直接出现 `LinuxBoot: ...` 系列行，
随后是内核日志（`console=tty0`）。

### 3.44 r42：把菜单应用从固件里彻底删掉

r41 上机：**菜单又出现了**（里面已经有 `[0] Linux (UFS)` ✓），然后自动去启动
`[1] Internal Storage`。原因是：上一版注册的 `Boot Menu` 启动项**已经存进 NVRAM 的
BootOrder**，而"不再注册"并不会删除已存在的变量——所以它照样被启动。

r42 从根上处理：

* `Include/DXE.inc` 与 `chopin.dsc` 里**不再把 `BootMenuApp` 编进 FV**
  （构建日志里 `BootMenuApp` 出现 0 次，`LinuxBootDxe.efi` 正常编入）；
* 于是 NVRAM 里那条旧的 `Boot Menu` 启动项**找不到文件、加载失败**，BDS 自动跳过，
  直接执行下一项 **`Linux (UFS)`** → 读 `boot_b` → 解压 → 交接。

固件从 833536 B 减小到 823296 B（DEBUG），也印证菜单应用确实不在里面了。

### 3.45 r43：真正只启动 Linux（清掉 NVRAM 里的其它启动项）

r42 的视频显示：菜单确实没了 ✓，但 BDS 依次去启动旧启动项
（`MsBootPolicy.efi` → `Internal Storage` 失败 → 试 USB），**`Linux (UFS)` 排在后面**。

原因：`EfiBootManagerAddLoadOptionVariable()` 把新项**追加**到 NVRAM 的 BootOrder 末尾，
"先注册"并不等于"排在最前"；旧的 "Internal Storage"/"USB Storage" 等项还在 BootOrder 里。

r43：新增 PCD `gMediaTekPkgTokenSpaceGuid.PcdChopinBootLinuxDirectly`（chopin 置 TRUE），
`MsBootOptionsLib` 在注册完成后把**除 `Linux (UFS)` 之外的所有启动项删除**
（`EfiBootManagerDeleteLoadOptionVariable`），于是 BootOrder 里只剩我们的加载器 → BDS 直接
执行它：读 `boot_b` → LZMA 解压 → 校验 → `ExitBootServices` → `ArmDisableCachesAndMmu` → 跳转。

> 该 PCD 默认 FALSE，因此不影响仓库里其它平台；只对 chopin 生效。

### 3.46 找到 `boot_b` 的一个真问题：LZMA 头里的长度是"未知"

按加载器的逻辑在主机侧复解析 `work/boot_b-chopin-test.img`，发现：

```
kernel 头字节 = 5d 00 00        (LZMA-alone ✓)
LZMA 头的解压后长度 = 0xFFFFFFFFFFFFFFFF   ← 未知长度！
```

`xz --format=lzma` 和 Python 的 `lzma.compress(FORMAT_ALONE)` **都把长度写成 -1（未知）**，
因为压缩器是流式的。而：

* 加载器的 LZMA 探测条件是"长度字段在 1 MiB..256 MiB"→ 直接不认为是 LZMA；
* EDK2 的 `LzmaUefiDecompress` 也需要已知的输出长度。

修法：压缩后**把真实长度写回 LZMA 头**（offset 5 的 8 字节 LE），这是安全的——解码器按
长度截断，尾部 end-marker 被忽略：

```
work/Image-chopin-raw.lzma : declared size = 44448256 ✓
work/boot_b-chopin-test.img: 11 309 056 B, FDT magic d00dfeed, sha256 f827e5c5…
```

### 3.47 r43 + 纯 raw Image 载荷（`boot_b` 有 128 MB，不必压缩）

用户确认 `boot_b` 实际有 **128 MB**，于是直接放**未压缩的 arm64 Image**，彻底绕开 LZMA：

| 项 | 值 |
| --- | --- |
| 文件 | `work/boot_b-chopin-test.img` |
| kernel | `work/Image-chopin-raw`（44 448 256 B，0x38 处 `ARM\\x64` ✓） |
| dtb | `work/chopin-plain.dtb`（201 784 B，FDT magic `d00dfeed` ✓） |
| ramdisk | 4 B 占位（无 initramfs，故内核会在找 root 处 panic，属预期） |
| 总大小 / sha256 | 44 656 640 B（42.6 MB）/ `ced5bf5ea1f1a77572f7f38b7cd5133f2eaf15c0fed38a2fb3fe591f3f80665b` |

加载器会走"非 LZMA"分支：直接把 Image 载入（必要时复制到 2 MiB 对齐的缓冲），
读 `text_offset` 得到入口，校验 DTB 后 `ExitBootServices` → `ArmDisableCachesAndMmu` → 跳转。

### 3.48 与 MT6895 参考的功能差距清单（下一步补齐）

把 `work/refs/mu-mt6995/Silicon/MediaTek/MT6895Pkg/Drivers` 与我们的 `MT6893Pkg/Drivers`
对比：

| 参考有 | 我们 | 说明 |
| --- | --- | --- |
| `MtkUfsDxe` | ✅ 已移植 | UFS 平台驱动（CLK_CG/refclk/SMC 复位/回调） |
| `MtkUfsPassThruDxe` | ✅ 已移植 | 修好的传输驱动（`Dd=UfsNoData`、强制 RSR、fence） |
| **`SimpleFbDxe`** | ❌ 缺 | 参考用它提供 **GOP/GraphicsConsole**。这正是我们"`EFI_SIMPLE_TEXT_OUTPUT` 不出字、
只能靠 DEBUG 通道"的根因——补上它，标准 ConOut/菜单/Shell 才能真正显示 |
| **`MtkUsbDxe`** | ❌ 缺 | USB host 模式（存储/键盘/FFU） |
| 平台 `KeypadDeviceLib` | ⚠️ 有但按键幻影 | 参考平台的 keypad 读取路径不同；我们现在的 ConIn 会无限投递 `sc=0001/0002` |
| `MT6359PmicDxe` | ✅ 有 | 但 regulator 表/充电（MT6360）未补 |

**建议补齐顺序**：

1. **`SimpleFbDxe`**（显示/ConOut）——顺带解决菜单与日志"只能靠 DEBUG 通道"的问题；
2. **keypad 读取路径**（参考平台的实现），解决幻影键与电源键；
3. **`MtkUsbDxe`**（USB host）；
4. ACPI 表骨架、MT6359 regulator / MT6360 充电。

### 3.49 r44/r45：GOP 显示打通 + 加载器等 BlockIo

**r44（`SimpleFbDxe` 移植）上机成功**：面板上的文字改由**标准 ConOut** 绘制（用户观察到
"文字居中了"——正是 GOP/GraphicsConsole 的特征）。同时截图暴露了新的问题：

```
[Bds]Booting Linux (UFS)
Loading driver at 0x00046EDD000 ... LinuxBootDxe.efi
LinuxBoot: start (UFS BlockIo -> boot_a -> kernel handoff)
LinuxBoot: FAILED: no BlockIo handle found (Not Found)
[Bds] Unable to boot!
```

即**加载器跑起来了，但那一刻 ScsiDisk 还没发布 BlockIo**（启动项执行得太早）。

**r45 修法**：加载器在查找 BlockIo 前先 `EfiBootManagerConnectAll()`，然后**每 100 ms 重试、最多 30 s**
（给 UFS 链路启动和 ScsiDisk 绑定留足时间），期间打印
`LinuxBoot: no BlockIo yet, connecting everything` / `waiting for BlockIo (N)`。

### 3.50 r45 的突破与 r46 的修正：加载器选错了分区

r45 日志（照片 `work/device/uefi-r45/shot.jpg`）走到了加载器内部，新东西很多：

```
LinuxBoot: no BlockIo yet, connecting everything
LinuxBoot: 1 BlockIo handle(s) present
LinuxBoot: GPT on disk #0, 128 entries (lba_size=4096, array at 0x2000)
LinuxBoot:   entry 31 'boot_a' LBA 170752..203519
LinuxBoot: partition 'boot_a' at LBA 170752 len 32768 blocks
LinuxBoot: partition 'boot_a' at byte 0x29b00000 size 0x8000000 (134217728 bytes)
LinuxBoot: boot image magic=ANDROID! version=2 page=2048
LinuxBoot: kernel size=617208 ... dtb_size=0
```

* **`EfiBootManagerConnectAll` + 轮询起作用了** ✓（BlockIo 出现、GPT 用 `lba_size=4096` 正确解析）；
* **`boot_b` 确实是 128 MB**（`0x8000000`），用户说得对；我早先按 16 MiB 的估算作废；
* 但它**优先选了 `boot_a`**——那是**我们的 UEFI 固件本身**（kernel 617208 = 压缩后的 UEFI，
`dtb_size=0`）→ 之后必然以"没有 DTB"失败。

r46 把分区优先级反过来：**`boot_b`（主）→ `boot_a`（回退）→ `boot`**。这才是 chopin 的实际布局：
`boot_a` 放 UEFI，`boot_b` 放 Linux 载荷。

### 3.51 r46：**交接成功执行**，但内核没有可见输出（fbdev 缺失）

r46 的照片（`work/device/uefi-video-2026-10-17-r46/`）显示加载器完整走完：

```
LinuxBoot: partition 'boot_b' at LBA 350976 len 32768 blocks
LinuxBoot: partition 'boot_b' at byte 0x55B00000 size 0x8000000 (134217728 bytes)
LinuxBoot: kernel size=44448256 addr=0x40080000 ...
LinuxBoot: kernel Image magic ok (0x644D5241), text_offset=0x80000, entry=0x43800000
LinuxBoot: dtb size=201784 first8=d0 0d fe ed 00 03 14 38 at 0x42E6F000
LinuxBoot: handoff: entry=0x43800000 dtb=0x42E6F000 ramdisk=0x46FB3000 kernel=0x43000000 ...
SecurityLock::LockType: SOFTWARE_LOCK ... LockCapsuleInterface
```

`handoff:` 之后紧跟着的是 `ExitBootServices` 的 security-lock 回调 → **ExitBootServices 成功、
MMU/cache 已关、已经跳进内核入口**。但之后屏幕上**没有任何内核输**。

**原因查清了**（`work/cfgtest/.config`，即 chopin 内核配置）：

```
CONFIG_DRM=m                      # 模块：没有 initramfs 就加载不了
# CONFIG_DRM_SIMPLEDRM is not set  # 没有 simple-framebuffer 支持
# CONFIG_FB_SIMPLE is not set      # 也没有旧的 simplefb
CONFIG_FB_EFI=y                   # 只有 efifb —— 但它要求经由 EFI stub 启动
```

也就是说：`console=tty0` **没有可用的 fbdev**；而我们是"手动跳转"，绕过了内核自带的
**EFI stub**，所以 `FB_EFI` 也拿不到 `screen_info` → 内核日志无处可去（`ttyS0` 是 UART，
我们没引出）。

**下一步（r47）**：改成**走内核自带的 EFI stub**，也就是：

1. 把 DTB 作为 EFI 配置表安装（`EFI_DTB_TABLE_GUID`）；
2. `gBS->LoadImage()` 直接加载 arm64 `Image`（它本身就是 PE/EFI 应用，`CONFIG_EFI=y`）+ `StartImage()`；
3. stub 自己会 `ExitBootServices`、把 GOP 转成 `screen_info` → **efifb** → `console=tty0` 就能看到内核日志。

这也是真实 arm64 引导器（GRUB/systemd-boot）的标准做法，比手动跳转更稳。

### 3.52 r47：改走内核自带的 EFI stub（让 efifb/console=tty0 生效）

r46 已证明手动交接能执行（`handoff:` + ExitBootServices 回调），但内核没有可见输出，
因为 chopin 内核里**只有 `CONFIG_FB_EFI=y`**（`simpledrm`/`simplefb` 都没开，`DRM=m` 是模块），
而 `efifb` 只有在**经由内核自带的 EFI stub** 启动时才拿得到 `screen_info`。

r47 因此把交接改成标准 arm64 引导器路线：

1. `gBS->InstallConfigurationTable (EFI_DTB_TABLE_GUID, dtb)` —— 把 DTB 交给 stub；
2. `gBS->LoadImage (TRUE, ImageHandle, NULL, kernel, size, &handle)` —— arm64 `Image`
   本身就是 PE/EFI 应用（`CONFIG_EFI=y`，首字节 `4D 5A`）；
3. `gBS->StartImage (handle, ...)` —— stub 自己执行 `ExitBootServices`、把 GOP 转成
   `screen_info` → **efifb** → `console=tty0` 就能看到内核日志。

`LoadImage` 失败时才回退到原来的"关 MMU 手动跳转"路径（保留 `ExitBootServices` +
`ArmDisableCachesAndMmu` + `LinuxBootJumpToKernel`）。新增日志：
`LinuxBoot: FDT config table -> ...` / `LoadImage(kernel PE) -> ...` / `StartImage -> ...`。

### 3.53 r47 成功：**UEFI → Linux 已经跑起来了** 🎉

r47 上机（截图 `work/device/uefi-r47/stub-log.jpg`、`kernel-running.jpg`）：

```
LinuxBoot: kernel size=44448256 ... header_ok=1 gzip=0
LinuxBoot: cmdline='bootopt=64S3,32N2,64N2'
LinuxBoot: handoff: entry=0x43800000 dtb=0x42E6F000 ...
SecurityLock::LockType: SOFTWARE_LOCK ... LockCapsuleInterface
LinuxBoot: LoadImage(kernel PE) -> Success handle=0x...
LinuxBoot: StartImage -> ... (stub 接管)
```

随后屏幕被**内核接管**并画出了它的启动进度条（用户描述为"Linux 内核测试条"）——
也就是说：**UEFI 通过内核自带的 EFI stub 成功启动了 Linux 内核** ✓✓✓。
这是整条主线（LK → UEFI → Linux）里最关键的一步。

（对比 r46 的手动跳转：同样"跳进去了"但没有任何输出，因为内核只有 `CONFIG_FB_EFI=y`；
r47 走 stub 后 `screen_info` 正确、`efifb` 生效，才有画面。）

**下一步**：内核现在没有根文件系统（`root=/dev/ram` 但 boot image 里 ramdisk 只有 4 字节占位）
→ 会在挂根时 panic。可选：
1. 放一个**最小 initramfs**（busybox 或原厂 recovery ramdisk）到 `boot_b` 的 ramdisk 段；
2. 用原厂 `boot_a` 的 ramdisk（Android 的 initramfs）配合 `androidboot.*` 参数走 Android 启动；
3. 以 `root=/dev/sdaN` 指向某个 ext4 分区（需要内核带相应 fs + 分区识别）。

### 3.54 r48：把 initrd 真正交给内核（`/chosen/linux,initrd-start/end`）

r47 已经能启动内核（看到启动进度），但 **init 没跑**。原因：加载器只把 ramdisk 读进内存，
**没有告诉内核它在哪**——arm64 内核是从 DTB 的 `/chosen/linux,initrd-start` /
`linux,initrd-end`（或 EFI stub 的 `initrd=`）拿 initramfs 的，我们两样都没有。

r48 补齐：

1. **载荷 DTB** 里加了两个占位属性（`dtc` 反编译 → 插入 → 重编译：
   `linux,initrd-start = <0 0x48000000>`、`linux,initrd-end = <0 0x48000000>`）；
2. **加载器在跳转前用实际地址覆盖它们**（新增 `LinuxBootSetInitrd()`：解析 FDT 找到
   `/chosen`，把两个 64 位单元写成 `RamdiskAddress` / `RamdiskAddress + RamdiskSize`），
   并打印 `LinuxBoot: dtb initrd-start -> 0x...`；
3. 载荷里的 ramdisk 换成**你的 init**：`aarch64-linux-gnu-gcc -static` 编译
   `port/initramfs/init.c` → `cpio -H newc | gzip` → 321 264 B。

### 3.55 r49：keypad 幻影键的真因——KPD 矩阵是**低有效**，读反了

参考实现（lancelot/stone 的 `KeypadDeviceLib`）其实也是 PMIC/GPIO 轮询，不是 KPD 驱动；
而我们的 chopin 库本来就在读 KPD 控制器。真因是：

> **MTK KPD 矩阵是 active-low**（`mtk-kpd.c`: `pressed = (new_state & mask) == 0`），
> 而基线代码写成了 `!!(KeypadState & (1 << Pin))` —— **读反了**。空闲时硬件读 `0xFFFF`，
> 于是两个音量键被永久判定为"按下"，过了 500 ms 长按门限后每 100 ms 重复投递
> `SCAN_UP`/`SCAN_DOWN` —— 正是我们看到的幻影上/下流。

r49 修好的内容：

* 按低有效解码（读 `KP_MEM1..5`，`0x04..0x14`）；
* `KP_DEBOUNCE = 0x400`、`KP_EN = 1`（对齐 DT）；
* **安全网**：全零矩阵采样（物理上不可能）视为无按键，并重试一次 + 打一条警告，
  这样即使 KPD 时钟没开也不会再刷屏；
* PMIC 定位改为非致命、每次轮询重试（`KeypadDeviceDxe` 的 depex 是 TRUE，可能早于
  `MT6359PmicDxe` 派发）；
* 顺手修了 `KeySetState/KeyGetState` 的 unicode 位图索引错误。

**新的键码映射**（实测待确认）：

| 按键 | 输出 |
| --- | --- |
| 音量上（点按） | `ScanCode 0x01`（SCAN_UP） |
| 音量下（点按） | `ScanCode 0x02`（SCAN_DOWN） |
| 长按 500 ms | 首次重复，之后每 100 ms |
| 电源（点按） | `UnicodeChar 0x0D`（回车） |
| 电源长按 | `ScanCode 0x17`（ESC） |
| 电源+音量上 / 电源+音量下 | `SCAN_LEFT` / `SCAN_RIGHT` |

KPD 基址仍由 `LocateMemoryRegionByName("Keypad")` → `0x10010000` 得到（无新增 PCD）。

### 3.56 r50：菜单回来了（默认选中 Linux (UFS)，8 s 自动启动）

r42/r43 为了"直接启动 Linux"把菜单从固件里删了，所以 r49 里按键自然没有菜单可开。
keypad 修好之后，r50 把菜单加回来：

* `BootMenuApp` 重新编进 FV（`DXE.inc` / `chopin.dsc`）；
* 启动顺序：**Boot Menu → Linux (UFS) → …**；
* 菜单**不再把自己列进条目**（否则默认选中会是它自己，空闲定时器会重启菜单），
  因此默认选中 `Linux (UFS)`，**8 秒无按键就自动启动它**；
* 按键（音量上/下 = 上/下移动，电源 = 确认）现在由修好的 KPD 路径投递。

效果：正常开机不按键 → 8 秒后进 Linux；按任意键 → 停在菜单里选择。

### 3.57 r51：换回 **Mu/Silicium 原生 boot manager 菜单**

要求使用平台自带的菜单，于是：

* 删掉 `chopin.dsc` 里对 `PcdBootManagerMenuFile` 的覆盖 → 回落到
  `SiliciumPkg.dsc.inc` 的默认值（`{DC 5B C2 EE …}`，即**原生 `BootManagerMenuApp`**）；
* 我们的 `BootMenuApp` 再次**不编进 FV**（源文件保留，仅作备用）；
* `MsBootOptionsLib` 仍然把 `PcdBootManagerMenuFile` 指向的应用注册成**启动顺序第一项**
  （`L"Boot Menu"`），所以 BDS 一上来就进原生菜单。

之所以这次值得再试一遍原生菜单：r29/r30 时它"画完 logo 就退出"，很可能是因为**当时没有
GOP/ConOut**（`EFI_SIMPLE_TEXT_OUTPUT` 写不进面板）。现在 `SimpleFbDxe` 提供了 GOP，
GraphicsConsole 也接回来了，原生菜单应该能正常绘制与响应按键。

### 3.58 对比"LK 直接启动能进 init"的镜像：两处关键差异

用户提供了参考镜像 `chopin-v2onehundredfifth-stockdt.img`（LK 直接启动、能进 init）：
gzip kernel（16 063 597 B）、ramdisk 318 561 B（gzip→783 360 B cpio）、**无内联 DTB**（用 LK 自己的）。

对比后发现两处我们缺的东西：

**（一）cmdline**

```
bootopt=64S3,32N2,64N2 console=tty0 console=ttyS0,921600n8
earlycon=mtk8250,mmio32,0x11002000
clk_ignore_unused regulator_ignore_unused pd_ignore_unused
ignore_loglevel panic=0 printk.devkmsg=on
```

* **`clk_ignore_unused` / `regulator_ignore_unused` / `pd_ignore_unused`** ——没有它们，
  内核启动后期会把"没被引用"的时钟/regulator/电源域关掉，很可能就挂在那里；
* **`earlycon`** ——早期控制台（UART）；
* 注意它**没有 `root=/dev/ram`**（有 initramfs 时内核直接跑 `/init`）。

**（二）initramfs 结构**

参考的 cpio 里有 `dev/ proc/ sys/ tmp/ bin/ init` 七个条目；我们之前只有 `init` 一个文件，
而 init 需要 `/proc`、`/sys`、`/dev` **挂载点**——没有目录就挂不上，自然什么都打不出来。

于是重建 `boot_b` 载荷（**不需要重新刷 UEFI**，只换 `boot_b`）：

* DTB 的 `/chosen/bootargs` 换成上面那条完整命令行（`dtc` 编辑）；
* ramdisk 重新打包成 `init + dev/ proc/ sys/ tmp/ bin/`。

### 3.59 r52：移植 `MtkUsbDxe`（USB host）

参考的 `MT6895Pkg/Drivers/MtkUsbDxe`（581 行）整体搬进 `MT6893Pkg/Drivers/MtkUsbDxe`：

* INF：`MT6895Pkg.dec` → `MediaTekPkg.dec`、`gMediaTekTokenSpaceGuid` → `gMediaTekPkgTokenSpaceGuid`；
* 新增 4 个 PCD（`MediaTekPkg.dec`，token `0xE120..0xE123`），值来自 **chopin 的设备树**：

| PCD | 值 | 来源 |
| --- | --- | --- |
| `PcdXhciBase` | `0x11200000` | `usb_xhci@11200000` |
| `PcdIppcBase` | `0x11203E00` | `ssusb` 的 `reg-names = "ssusb_base","ssusb_ippc","ssusb_sif2"` |
| `PcdU2PhyBase` | `0x11E40000` | `usb0phy@11e40000` |
| `PcdI2cBase` | `0x11D01100` | `fpga_i2c_physical_base`（先按此值，后续可校准） |

* `chopin.dsc` 补上 `NonDiscoverableDeviceRegistrationLib`（该驱动依赖，平台原先没解析），
  并把 `MtkUsbDxe.inf` 加进 `[Components]` 与 `DXE.inc`；
* 构建日志里出现 `MtkUsbDxe.efi`，DEBUG 镜像从 825344 增至 **829440** B。

### 3.60 r53：USB 崩溃 = 寄存器区域没进内存映射

r52 上机直接异常：

```
Synchronous Exception at 0x000047054E84   (MtkUsbDxe)
ESR : EC 0x25 ... Data abort: Translation fault, third level
FAR 0x0000011203E08
```

`FAR = 0x11203E08` 正是 **`PcdIppcBase (0x11203E00) + 8`** ——USB 的 IPPC 寄存器区域
**不在 UEFI 的内存映射里**，第一次访问就数据异常。

r53 在 `MemoryMapLib` 里补两条（4 KiB 对齐，符合 ARM64 页表要求）：

```c
{"USB",      0x11200000, 0x00010000, AddDev, MMAP_IO, UNCACHEABLE, MmIO, NS_DEVICE},
{"USB PHY",  0x11E40000, 0x00010000, AddDev, MMAP_IO, UNCACHEABLE, MmIO, NS_DEVICE},
```

`0x11203E00` 落在 `USB 0x11200000..0x11210000` 内 ✓，U2 PHY 单独映射 ✓。

### 3.61 r54：第二个崩溃 = I2C 基址填错（`0x11D01100` → `0x11280000`）

r53 之后仍然异常，但地址变了：

```
ESR : EC 0x25 ... Data abort: Translation fault, second level
FAR 0x0000000011D0110C          ← PcdI2cBase(0x11D01100) + 0xC
X24 = 0x11D0110C                ← 驱动里的寄存器指针
```

`0x11D01100` 是我之前从 DTS 里误取的 `fpga_i2c_physical_base`（FPGA 地址，不是真 I2C）。
查参考平台（pearl/MT6895）的 PCD：

```
PcdXhciBase  = 0x11200000
PcdIppcBase  = 0x11203E00
PcdU2PhyBase = 0x11E40000
PcdI2cBase   = 0x11280000     ← USB 专用 I2C
```

r54 把 `PcdI2cBase` 改成 **0x11280000**，并把该区域也加进内存映射：
`{"USB I2C", 0x11280000, 0x1000, ...}`。

### 3.62 USB：初始化通了、但**没有 VBUS 供电**（缺 MT6360 OTG 使能）

r54 上机：不再崩溃、原生菜单正常 ✓；但插 OTG hub **指示灯不亮**、U 盘不出现 → **端口没有供电**。

原因在参考驱动的头注释里写得很清楚：

```
- switches the MT6375 Type-C port to host mode over I2C (TCPCI ...)
```

参考（pearl/MT6895）用的是 **MT6375** 做 Type-C 角色切换与 VBUS 使能；**chopin 用的是 MT6360**，
I2C 地址（0x34）与寄存器都不同 → 那段 I2C 序列对 chopin 无害但也无效，端口自然没电。

MT6360 的 OTG 使能要从上游 `drivers/power/supply/mt6360_charger.c` 的寄存器表取
（本仓库的 vendor 内核里只有 `MT6360_MASK_OTG_OC` 这类限流位，没有 `OTG_EN` ✗）。

**下一步（A 方案）**：查上游 `mt6360_charger.c` 拿到 `OTG_EN` 的寄存器/位，
用我们已有的 polled I2C 写 MT6360 → 打开 OTG boost（VBUS 5V）+ 切 host 模式。
这需要网络查询上游源码（本轮预算已用尽）。

### 3.63 r55：MT6360 OTG（VBUS）使能

A 方案落地。MT6360 的 OTG/boost 使能在 **`CHG_CTRL1 (0x11)` bit0 = `OPA_MODE`**（vendor 头
`drivers/misc/mediatek/pmic/mt6360/inc/mt6360_pmu_chg.h` 明确定义），并且 OTG 供电时充电必须关掉
（`CHG_CTRL2 (0x12)` bit0 `CHG_EN`）。

`MtkUsbDxe` 新增 `MtkUsbMt6360OtgOn()`，用驱动里已有的 polled I2C（`MtkI2cRmw`）在 I2C 地址
`0x34` 上做两次读改写：

```
CHG_CTRL1: OPA_MODE=1, HZ_EN=0, FORCE_SLEEP=0     → 打开 OTG boost（VBUS 5V）
CHG_CTRL2: CHG_EN=0                                → OTG 模式下停止充电
```

并在 `MtkUsbDxeInitialize` 一开始就调用（先供电、再注册 xHCI），日志：
`MtkUsbDxe: MT6360 OTG enable (CHG_CTRL1) -> Success` / `charger off (CHG_CTRL2) -> Success`。

### 3.64 r55 上机：MT6360 I2C **超时**（控制器没被初始化）

```
MtkUsbDxe: MT6360 OTG enable (CHG_CTRL1) -> Timeout
MtkUsbDxe: MT6360 OTG failed (Timeout), port will not be powered
```

传输超时（不是 NACK）说明 **MTK I2C 控制器本身没工作**：`0x11280000` 只是寄存器地址，
I2C 还需要

1. **时钟门控**（`ClockImplLib` 里已列出 14 个 `IMP_IIC_WRAP_*_AP_I2Cx` 门 ✓）；
2. **引脚复用**（SDA/SCL 切到 I2C 功能，走 `GpioImplLib`）；
3. 正确的总线基址（chopin 的 DTS 里 `i2cN@` 有 `0x11015000/0x11D00000/0x11D01000/0x11CB0000/0x11D02000/0x11F00000/0x11F01000/0x11D03000` 等，
   MT6360 挂在哪一路需要从 vendor DTS 的 `mt6360_pmu_dts` 归属确认）。

**下一步（A1）**：给 MT6360 所在的那路 I2C 做三件事——开时钟门、配引脚复用、选对基址；
做完后 `CHG_CTRL1/CHG_CTRL2` 的读改写才能真正落到 MT6360，OTG (VBUS) 才有电。

### 3.65 r56：MT6360 在 **i2c5 (`0x11F00000`)**，I2C 平台初始化补齐

子代理把 A1 做完了，关键结论都有证据：

**（一）总线不是 `0x11280000`**

vendor DT（`n10p/vendor-fdt.dts`）里 `i2c5@11F00000` 下正好挂着 MT6360 的四颗 die：
`subpmic@34`（PMU/charger，`mediatek,subpmic` = `mt6360_pmu_i2c.c` 的 of_match）、`usb_type_c@4e`、
`subpmic_pmic@1a`、`subpmic_ldo@64`，还有 `clock-frequency = <0x33e140>`（340 kHz）。
`0x11280000` 在 chopin 上**不是 I2C 控制器**（读回全 0）→ 之前的 `Timeout`。

**（二）时钟 / 引脚 / 时序**

| 项 | 值 |
| --- | --- |
| INFRACFG_AO 门 | `0x100010A4` bit 18（状态 `0x100010AC`） |
| IMP_IIC_WRAP_N 门 | `0x11F02E08` bit 0（状态 `0x11F02E00`） |
| 引脚 | GPIO202 = SCL5、GPIO203 = SDA5，功能 1（mode word `0x10005490`） |
| 模块时钟 | `CLK_TOP_I2C_SEL`，vendor 的 `[I2C]Clk=24960000` → `CLOCK_DIV=0x0404`，SCL ≈ 40 kHz |
| I2C 寄存器 | 补齐 v2 全套（`CONTROL`/`FIFO_ADDR_CLR`/`TRANSFER_LEN_AUX`/`LTIMING`/`SDA_TIMING`…），50 ms 轮询超时 |

**（三）OTG 序列**（先 BC1.2 off、VOBST=5.00 V、OTG_OC=3 A，再 `CHG_CTRL1.OPA_MODE=1`、
`CHG_CTRL2.CHG_EN=0`，最后回读确认）；同时**删掉了原先对 `0x34` 的 MT6375 寄存器写**——在 chopin 上
`0x34` 是 MT6360，那套偏移会写到无关的充电寄存器。

**（四）内存映射**：新增 12 个 4 KiB 区域（I2C-0..7 + Wrap C/E/N/S）。

### 3.66 r56 上机：**8 条 I2C 全部 Timeout** → 控制器本身没在跑

```
MtkUsbDxe: MT6360 probe 0x11f00000 (i2c5) -> Timeout
MtkUsbDxe: MT6360 does not answer on 0x11f00000 (i2c5), scanning the other I2C buses
MtkUsbDxe: MT6360 probe 0x11015000 (i2c0) -> Timeout
MtkUsbDxe: MT6360 probe 0x11D00000 (i2c1) -> Timeout
... (i2c2/3/4/6/7 同样 Timeout)
MtkUsbDxe: MT6360 not found on any I2C bus (Not Found), port will not be powered
```

**总线选错的可能性已经排除**（8 条全扫过）✓——问题是 **MTK I2C 控制器本身没有完成传输**，
而且**时钟门/引脚复用的读回检查没有报 WARN**（说明那两步"看起来"成功了）。

两种可能：

1. **模块时钟没真正跑起来** —— `CLK_TOP_I2C_SEL` 的父时钟/分频由 TOPCKGEN 控制，
   我们只写了 `CLOCK_DIV`，没有走完整的时钟树（TOPCKGEN + INFRACFG + wrap）；
2. **I2C 传输的编程仍有错** —— MTK I2C v2 的 `CONTROL`/`START`/FIFO 时序有坑，
   `Timeout` 也可能是传输根本没启动。

**决定性诊断（下一轮）**：在失败后把寄存器 dump 出来——
`I2C_INTR_STAT`、`I2C_START`、`I2C_CONTROL`、`CLOCK_DIV`，以及两处时钟门的状态位——
就能区分"时钟没开"还是"传输没起来"。

> 注：**Linux 侧完全不需要这些**——DT 交给内核后，`mt6360_charger`/`xhci-mtk` 会自己开 OTG 与枚举；
> 这套 UEFI 侧的 I2C 只为"UEFI 阶段也能用 USB"服务。

### 3.67 r57：I2C 加 push-pull + 超时寄存器 dump

两条改动：

1. **push-pull**：chopin 的 i2c5 在 vendor DT 里带 `mediatek,use-push-pull`，对应的
   控制器位是 `I2C_CONTROL` **bit 7** —— 之前没设，开漏模式下跟 MT6360 的通信可能直接
   起不来；
2. **超时寄存器 dump**（关键诊断）：
   ```
   MtkUsbDxe: i2c 0x11f00000 TIMEOUT stat=… start=… ctrl=… div=… timing=… len=… aux=… transac=…
   ```
   一次就能判断是"**模块时钟没跑**（div 读回 0 / start 没置位）"还是"**编程错了**
   （stat 里有错误位、len/ctrl 不对）"。

### 3.68 r58：补上 TOPCKGEN 的 I2C 模块时钟（寄存器读回全 0 的真因）

r57 的 dump 是决定性的：**8 条 I2C 的寄存器全部读回 0**（包括刚写的 `ctrl`/`div`）——
外设没有模块时钟时读寄存器就是 0，所以不是传输编程错误，而是**时钟没开**。

r58 在 `MtkI2cClockEnable()` 里补上最后一层——**TOPCKGEN**：

| 寄存器 | 值 | 依据 |
| --- | --- | --- |
| `TOP_CLK_CFG_10` (`0x100000B0`) | bits[1:0]=0（父时钟 clk26m）、bit2(pdn)=0 | `clk-mt6877.c`: `MUX_CLR_SET_UPD(CLK_TOP_I2C_SEL, "i2c_sel", …, CLK_CFG_10(0x00B0), …)` |
| `TOP_CLK_CFG_UPDATE1` (`0x10000008`) | bit 9 置位提交 | `TOP_MUX_I2C_SHIFT = 9` |

并把 `TopCkGen (0x10000000, 4 KiB)` 加进内存映射。
日志新增 `MtkUsbDxe: <bus> TOPCKGEN I2C_SEL cfg10=0x…` —— 只要 `div` 之后能读回非 0，
就说明控制器活了；再走 MT6360 探测与 OTG。

### 3.69 A 两轮结束（USB-in-UEFI 挂起）+ r59 转 initramfs

**A 的结论**（r57/r58）：

* r57 dump：8 条 I2C 寄存器全 0 → 缺模块时钟；
* r58 补上 TOPCKGEN `CLK_CFG_10`/`UPDATE1` 后，`cfg10=0x05050201` **可读可写** ✓，
  但 **I2C 控制器寄存器仍然全 0** → 那两层 gate（INFRACFG_AO / imp_iic_wrap）状态位
  虽然"看起来"置上，实际没让控制器活起来——还差 APB/power-domain 那一层，而我们已经
  没有可信的寄存器依据。

按约定 **两轮结束，USB-in-UEFI 挂起**。注意：**Linux 侧的 USB 不依赖这些**
（DT 交给内核后 `mt6360_charger` + `xhci-mtk` 自己完成）。

**转 initramfs**：

内核配置确认支持（`CONFIG_BLK_DEV_INITRD=y`、`CONFIG_RD_GZIP=y`）；问题在于**交接**——
arm64 的 EFI stub 从命令行参数 **`initrd=<addr>,<size>`** 取 initramfs，而 `/chosen/linux,initrd-start/end`
只在"直接跳内核"时生效。r59 因此在 `LoadImage` 之后**设置子镜像的 `LoadOptions`**：

```
initrd=0x<ramdisk>,0x<size>
```

日志新增 `LinuxBoot: stub LoadOptions 'initrd=0x...,0x...'`。

### 3.70 重建 `boot_b`：**内核换成"能进 init"的那一份**

校验时发现关键差异——我们的载荷内核与参考镜像根本不是同一个：

| | 大小 | sha256(前 32) |
| --- | --- | --- |
| 参考镜像内核（解压后） | 45 603 328 | `45fbac20…` |
| 我们之前用的内核 | 44 448 256 | `ca459264…` |

于是按"最稳"原则重建 `boot_b`：

| 段 | 来源 | 大小 |
| --- | --- | --- |
| kernel | **参考镜像里的内核**（gunzip 后 raw Image，`MZ` + `ARM\\x64` ✓） | 45 603 328 |
| ramdisk | **参考镜像里的 initramfs**（cpio.gz，含 `dev/ proc/ sys/ tmp/ bin/ init` ✓） | 318 561 |
| dtb | 我们的裸 FDT（bootargs 含 `clk_ignore_unused` 等 + initrd 占位） | 201 892 |

回读校验全部通过：`ANDROID!`、hv2/page2048、三段 size 与源文件一致、内核 `MZ`+`0x644d5241`、
DTB `0xd00dfeed`；总大小 46 129 152 B（44.0 MB），远小于 `boot_b` 的 128 MB。

### 3.71 r60：GOP 行跨距改 1080 + 新 cmdline（内核重编交给子代理）

三条针对性调整（用户确认）：

1. **GOP 行跨距**：`PcdMipiFrameBufferWidth` 1088 → **1080**，于是 SimpleFbDxe 报告的
   `PixelsPerScanLine` 与可见宽度一致（若之前绿条是 stride 错位造成的，这一改就会消失）；
2. **cmdline 顺序**：`console=ttyS0,921600n8 earlycon=… … fbcon=rotate:0 console=tty0 rdinit=/init`
   ——`console=tty0` 放到**最后**（console 顺序决定谁最终接管），显式 `rdinit=/init`，
   并加 `fbcon=rotate:0`；
3. 同一条 cmdline 也写进**DTB 的 `/chosen/bootargs`**（内核优先用 DT 的值），
   `/chosen/linux,initrd-start/end` 占位保持不动（加载器运行时会覆盖成真实地址）。

内核按 UEFI 启动需求重新配置并打包（`CONFIG_DRM_SIMPLEDRM=y`、`CONFIG_FB_SIMPLE=y`、
`CONFIG_SYSFB*`、`DEVTMPFS*` 等）由子代理执行，产物 `work/boot_b-chopin-uefi.img`。

### 3.72 r61：initrd 必须落在 **0x51100000**（内核用内嵌 board tree）

子代理在重建内核时发现（并已核对源码）：

* chopin 的内核补丁在 `setup_machine_fdt()` 之后**用内嵌的 board tree 替换 bootloader 的 FDT**，
  而且 `CONFIG_CMDLINE_FORCE=y`；
* 内嵌树里 `/chosen/linux,initrd-*` 是 **0x51100000**（vendor 布局），
  所以我们写在 DTB / LoadOptions 里的地址**都会被丢弃** ✗。

于是 r61 在加载完 ramdisk 后，**把它复制/固定到 0x51100000**：

```
LinuxBoot: ramdisk copied to the board-tree initrd address 0x51100000 (318561 bytes)
```

（`AllocateAddress` 预留该地址；失败则打印错误并保留原地址。）

这才是"内核起来了但不跑 /init"的直接原因：内核按内嵌树去 0x51100000 找 initramfs，
而之前那里是空的。

### 3.73 内核按 UEFI 启动需求重建（`work/boot_b-chopin-ufi.img`）

子代理完成。配置变更只有这几项（`.config` 逐行核对）：

| 选项 | 旧 | 新 |
| --- | --- | --- |
| `SYSFB_SIMPLEFB` | 未设 | **y** |
| `DRM_SIMPLEDRM` | 未设 | **y** |
| `DRM_SYSFB_HELPER` | 不存在 | y（被选中） |
| `DRM_GEM_SHMEM_HELPER` | m | y（被 simpledrm 强制） |
| `FB_SIMPLE` | 未设 | **不能设**（`depends on !DRM_SIMPLEDRM`） |
| `CMDLINE` | tty0 在前 | 我们的字符串（`console=tty0` 在最后）+ ramoops/pstore 尾巴，`CMDLINE_FORCE=y` 保留 |

关键点：**内嵌 board tree 里本来就有 `/chosen/framebuffer@78e0c000`（simple-framebuffer，
1080×2400，stride 4352，a8r8g8b8），但旧配置里既没有 simplefb 也没有 simpledrm** ——
所以 `console=tty0` 永远无处可画，这正是面板只有绿条的原因。

打包（端口脚本 `build-images.sh`，`RAW_KERNEL=1`）产物：

| 项 | 值 |
| --- | --- |
| 文件 | `work/boot_b-chopin-ufi.img` |
| 大小 / sha256 | 46 200 832 B（44.06 MiB）/ `e15fb6f94f786e32eb5391b76b98a052f98daac779709708d01d9a347b7cb749` |
| 内核 | 45 672 960 B，`MZ` + `0x644d5241@0x38` ✓ |
| ramdisk | 321 394 B（cpio newc + gzip，`/init` 0775） |
| dtb | 201 920 B（`d00dfeed`，`/chosen/bootargs` 与镜像头逐字节一致） |
| 内嵌 initrd | `fdt-initrd.py` 已把内嵌树改成 **0x51100000..0x5114e772** ✓（与 r61 加载器一致） |

**init 成功的标志**：面板/串口出现 `CHOPIN MAINLINE: userspace reached (PID 1)`。

### 3.74 r62：行跨距改回 1088（1080 会造成撕屏）

r60 把 `PcdMipiFrameBufferWidth` 从 1088 改成 1080 试 stride，结果**菜单与 logo 全部撕裂** ✗
→ 说明这块面板的真实行跨距就是 **1088 像素（4352 字节）**，`SimpleFbDxe` 原来的设置是对的 ✓；
之前那条"绿条"与 stride 无关（真正原因是内核没有 fbdev 驱动认 `/chosen/framebuffer` 节点，
已由 `CONFIG_DRM_SIMPLEDRM=y` 解决）。

r62 改回 **1088**。

### 3.75 r63：新载荷的 DTB 没内联 + 内核被分配到 4 GiB 以上

r62 上机暴露两个回归：

```
LinuxBoot: kernel size=45672960 ... loaded at 0x13A600000     ← 内核在 4 GiB 以上
LinuxBoot: could not reserve 0x51100000 for the initrd (Not Found)
LinuxBoot: boot image header v2 reports a zero sized DTB
LinuxBoot: FAILED: no DTB in the boot image, not starting      ← 于是直接进菜单
```

1. 子代理打的是 "raw-DTB 变体"：DTB 在镜像里但**没写进 boot image 头的 `dtb_size/dtb_addr`**
   → 加载器（要求内联 DTB）拒绝启动；
2. `0x51100000` 的 `AllocateAddress` 返回 `Not Found`（该区域已被内存映射描述）

修正：

* **重建 `boot_b`**：kernel（新编译的 `build/chopin/.../Image`，45 672 960 B）+ ramdisk
  （321 394 B）+ **DTB 内联**（`dtb_size=201920`、`dtb_addr=0x4FF00000`）；
* **initrd 拷贝改为无条件**：即使 `AllocateAddress` 拒绝，也直接拷到 `0x51100000`
  （紧邻 ExitBootServices，此时没有其他分配者），日志会打印
  `0x… is already described by the memory map (…), copying anyway`。

### 3.76 r64：把内核搬到 4 GiB 以下

r63 上机：DTB 与 initrd 都对了 ✓，内核**起来了**（面板出现内核的测试条 ✓），但仍不进 init。

日志里一直有一条被忽略的警告：

```
LinuxBoot: kernel was loaded above 4 GiB (0x13A600000), it would have to be relocated for a 32-bit entry
```

r64 因此**在内核准备完成后，如果地址 > 4 GiB 就搬到低地址**（`AllocateMaxAddress` 申请
低于 4 GiB 的内存，拷贝后更新入口地址）：

```
LinuxBoot: kernel moved below 4 GiB: 0x13a600000 -> 0x4..... (45672960 bytes)
```

（这条警告早就存在，只是之前内核根本没被跑到；现在链路通了，它就变成唯一显著异常。）

### 3.77 重新规划：主线内核从 UEFI 启动到 init

**现状盘点**（已经确定的事实）：

* UEFI 侧链路 **全通**：UFS → GPT → `boot_b` → 读内核/DTB/initrd → EFI stub → `ExitBootServices`
  → 内核开始执行（面板出现进度条）；
* 内核侧**还没到 init**，也没有内核 printk 出现在面板上；
* 已知的、与"面板只有进度条"高度相关的两个坑（历史文档里有记录）：
  1. **`NR_FIX_BTMAPS` 预算**（全局 65 页）：`setup_arch()` 里的 `early_ioremap` 若一次映射太多页，
     会在 `paging_init()` 之前耗尽并静默停住 —— 当年就是"**10 条绿线**"（4 条 head.S +
     6 条 `setup_arch` 面包屑，死在 `[BC 05] before paging_init` 与 `[BC 06]` 之间）；
  2. **fbcon/驱动**：内核必须**有驱动认领 `/chosen/framebuffer@78e0c000`**，`console=tty0`
     才有地方可画（已通过 `CONFIG_DRM_SIMPLEDRM=y` 解决）。

**观测手段（按可靠性排序）**：

| 手段 | 得到什么 | 说明 |
| --- | --- | --- |
| **进度条数量** | 死在第几个路点 | 4 条 = head.S 阶段；每多 1 条 = 多过一个 `setup_arch` 路点；13 条 = 走完 `setup_arch` |
| **expdb / pstore** | **完整内核 printk** | cmdline 已带 `pstore.kmsg_bytes` + `ramoops.mem_address=0x48090000`；失败后**重启进 Android 立刻取 expdb**，就能拿到死点前的最后一行内核日志（当年靠这个看到 `[BC 05]`→`tmigr_init`） |
| **`earlycon`** | 早期串口输出 | `earlycon=mtk8250,mmio32,0x11002000` 已在 cmdline；需要接 UART 才能看 |

**执行计划**：

* **P0（观测）**：刷 r64 + 新 `boot_b`；数进度条条数；失败后重启进 Android，用
  `port/tools/read-expdb-log.sh` 取内核日志（`port/tools/parse-expdb.py` 解析）；
* **P1（定位）**：按"条数"对表 → 4/5/…/13 条分别对应到具体路点；再结合 pstore 最后一行的
  函数名确定死点；
* **P2（修复）**：
  - 若死在 `paging_init` 之前 → 把 `fb_band_early` 改成 **4 页分块映射**（历史修复，已验证
    能把 13 条绿线跑满）；
  - 若卡在 `do_initcalls`（当年停在 `tmigr_init` 与 `mtk_logstore_early_init` 之间）→ 用
    `initcall_debug` 找那个不打印的 initcall，逐个 `clk_ignore_unused`/`regulator_ignore_unused`
    已加，必要时把它 `__initcall` 降级或加日志；
  - 若面板始终无字但 pstore 有日志 → 是 fbcon/simpledrm 那一侧（下一步查 simpledrm 对
    `/chosen/framebuffer` 的 probe）；
  - 若内核日志到 `Run /init` 就断 → initramfs 一侧（initrd 地址已固定 `0x51100000`，
    再查 cpio 是否被正确解压）。
* **P3（验证）**：出现 `CHOPIN MAINLINE: userspace reached (PID 1)` 即达成目标。

**建议的下一步（最小动作）**：先做 **P0** 的两件事——数条数 + 取一次 expdb 内核日志。
有这两项，就能把"哪一行、哪个路点"钉死，而不是继续盲改。

### 3.78 内核 `paging_init()` 二分探针（10 条 → 精确定位）

expdb 里没有内核日志（只有 preloader/LK 的环），pstore 又被 Android 的 SELinux 挡住
（`/sys/fs/pstore`、`/proc/last_kmsg` 连 `su` 都读不到）。于是改用**屏幕条数**做二分。

在 `paging_init()` 内部与 `bootmem_init()` 里新增 8 个阶段（编号 9–16，band 13–20；
原有的自动计数 0–8 与 head.S 的 0–3 完全不动）：

| 条数 | 新增阶段 | 含义 |
| --- | --- | --- |
| 10 | — | 仍死在 `paging_init()` 入口之前 |
| 11 | 9 | 进入 `paging_init()`，死在 `map_mem()` 前半 |
| 12 | 10 | 内核镜像已映射，死在内存 bank 的 `__map_memblock()`（重点嫌疑） |
| 13 | 11 | `map_mem()` 完成，死在 `memblock_allow_resize()`/返回路径 |
| 14 | 12 | 死在 `create_idmap()` |
| 15 | 13 | 死在 `declare_kernel_vmas()` |
| 16 | 14 | `paging_init()` 返回了 |
| + | 15/16 | `bootmem_init()` 的 `early_memtest` / 结束（在 BC 07 与 BC 08 之间点亮） |

打包产物（**内联 DTB**，DTB 在头里字段 1632/1636，与本加载器一致）：

| 项 | 值 |
| --- | --- |
| 文件 | `work/boot_b-chopin-probe.img` |
| 大小 / sha256 | 46 266 368 B / `8ab2ce47bb4e72441f8dbda95cd7940dc59b417099b4a389726c5f70ad641a3c` |
| 内核 | 45 738 496 B（+8 个探针），`MZ` + `0x644d5241` ✓ |
| ramdisk | 321 394 B ✓（内嵌 initrd 范围 `0x51100000..0x5114e772` ✓ 与加载器固定地址一致） |
| dtb | 201 920 B（`d00dfeed`），`dtb_size=201920`、`dtb_addr=0x4FF00000` ✓ |

新增工具：`port/tools/pack-uefi-boot-img.py`（UEFI 载荷打包，字段位置与本加载器一致）。

### 3.79 r65：ACPI 表落地（目标转向 Windows / WinPE）

新增 `Platforms/Xiaomi/chopinPkg/AcpiTables/`（`AcpiTables.inf` FILE_GUID
`7E374E25-…` = `PcdAcpiTableStorageFile` 默认值，`AcpiPlatformDxe` 无需改 PCD）：

| 表 | 内容（已解析构建产物验证） |
| --- | --- |
| **MADT** rev5 | 8 × GICC（uid/mpidr `0/0x000…7/0x700`，GICv3 系统寄存器、无 GICC/GICV/GICH 基址，维护 GSIV 25）；GICD `0x0C000000` v3；GICR `0x0C040000` len `0x100000`（8 × 0x20000） |
| **GTDT** rev2 | GSIV 29/30/27/26，flags 0（电平、高有效）；CntControl/CntRead = `0xFFFF…FFFF`（未映射，符合 ACPI 约定；13 MHz 在 `CNTFRQ_EL0`，GTDT 无频率字段） |
| **FADT** rev6/minor3 | `Flags=0x00300000`（HW_REDUCED + 低功耗 S0）、`ArmBootArch=0x0001`（PSCI、SMC）、无 SCI、ResetReg 为 NULL GAS（复位/关机走 PSCI） |
| **SPCR** rev2 | 16550 @ `0x11002000`（GAS 字节宽）、GIC 中断 GSIV 141、波特率 as-is（rev2 无法表示 921600） |
| **DSDT** | 8 × `ACPI0007`（`_UID` 0–7）+ `PNP0501` UART（`Memory32Fixed` + IRQ 141） |

接线：`chopin.dsc [Components]` 一行 + `chopin.fdf` 一行 `INF RuleOverride = ACPITABLE`。
RSDP/XSDT 由 `AcpiTableDxe` 运行时生成（AArch64 上 `PcdAcpiExposedTableVersions=0x20` → 只发 XSDT）。

**镜像内证据**（本机复核）：`Mu-chopin.img` → gunzip → 2 MiB FD → 解 LZMA（`FVMAIN` 3 620 872 B）
→ `FACP@0x2086a4`、`GTDT@0x2087bc`、`APIC@0x208820`、`SPCR@0x208af8` ✓（`0x2002ec` 处的 `DSDT`
是 AcpiTableDxe 的调试字符串 "DSDT table not found"，不是表）。

### 3.80 `cust` 的 WinPE FAT32 镜像（1 GiB 分区 + 896 MiB WIM）

用户提供 arm64 WinPE（`BYDJ_PE_V4.0.1.iso`，932 MB）：`efi/boot/bootaa64.efi`（2.6 MB）、
`efi/microsoft/boot/bcd`（16 KB）、`sources/boot.wim`（**940 090 097 B**）。

`cust` 实际容量是 **0x40000000 = 1 GiB**（之前按 LK 的 `nr_sects` 算成 768 MiB 是错的），
所以 896 MiB 的 WIM 放得下。

工具链：`mkfs.vfat` + **`mtools`**（`mcopy -s` 一次递归写入，分步 `mcopy` 会出现目录项丢失的坑）。

产物 `work/cust-winpe.img`：

| 项 | 值 |
| --- | --- |
| 大小 / sha256 | 1 048 576 000 B（1000 MiB FAT32）/ `d2e7bf01464aaa9161f08aee47a5c8388f8dd7e4441f406a3806166095f92f83` |
| `\EFI\Boot\BOOTAA64.EFI` | 2 622 896 B ✓ |
| `\EFI\Microsoft\Boot\bootmgfw.efi` | 2 622 896 B ✓ |
| `\EFI\Microsoft\Boot\BCD` | 16 384 B ✓（指向 `\sources\boot.wim` ✓） |
| `\sources\boot.wim` | 940 090 097 B ✓（与源文件 **sha256 一致** ✓） |

刷写：`fastboot flash cust work/cust-winpe.img`（**会整块覆盖 cust**，先备份）。

### 3.81 **Windows Boot Manager 在 chopin 的 UEFI 上跑起来了**（BCD 报错）

上机结果（`work/device/uefi-winpe/bootmgr-bcd-error.jpg`）：

```
Windows Boot Manager
Windows failed to start...
File:   \EFI\Microsoft\Boot\BCD
Status: 0xc000000d
Info:   The Boot Configuration Data for your PC is missing or contains errors.
```

两个结论：

1. **arm64 的 Windows Boot Manager 已经在我们这套 UEFI 上成功执行** ✓✓ —— 同时证明 ACPI 表、
   GOP framebuffer、以及固件环境都是"Windows 可接受"的；
2. 它停在 **BCD**：`0xc000000d` = BCD 缺失/内容错误。PE ISO 里的 BCD 是针对 ISO 自身介质
   （卷标识/设备路径）编好的，原样拷到 FAT 分区后设备引用不匹配就会报这个错。

**下一步**：换一份**与卷无关（`[boot]` 相对）**的最小 BCD，指向 `\sources\boot.wim`。

### 3.82 找到 `0xc000000d` 的真因：镜像里缺 **`\boot\boot.sdi`**

解析 `cust-winpe.img` 里那份 BCD（注册表 hive）发现它引用了两个路径：

```
\boot\boot.sdi          ← 镜像里没有 ✗
\sources\boot.wim       ✓
```

PE ISO 里 `boot/boot.sdi`（3 170 304 B）是 WinPE ramdisk 的 **SDI 描述符**，
`{ramdiskoptions}` 的 `ramdisksdipath` 指向它；缺了它 bootmgr 就报 `0xc000000d` ✓。

重建后的 `work/cust-winpe.img`（1000 MiB FAT32，剩余 ~93 MiB）：

| 路径 | 大小 |
| --- | --- |
| `\EFI\Boot\BOOTAA64.EFI` | 2 622 896 |
| `\EFI\Microsoft\Boot\bootmgfw.efi` | 2 622 896 |
| `\EFI\Microsoft\Boot\BCD` | 16 384 |
| **`\boot\boot.sdi`** | **3 170 304** |
| `\sources\boot.wim` | 940 090 097 |

sha256 `386e3a3edfa497e9f4ca134c4ca991a2874208ab774a2c3ecc9a4b7215199e05`。
`port/tools/make-winpe-fat-img.sh` 已同步要求并拷贝 `boot.sdi`。

### 3.83 WinPE 引导链的关键突破：`cdboot.efi` 可用，改 `cdboot_noprompt.efi`

* 补 `\boot\boot.sdi` 后仍报 `0xc000000d` ✗ → 说明 PE 的 BCD 绑定光盘介质、原样搬到 FAT 分区不可用；
* 换成 ISO 里自带的 **`efi/microsoft/boot/cdboot.efi`**（968 000 B）作为 `\EFI\BOOT\BOOTAA64.EFI`
  → **成功执行** ✓，屏幕出现 `Press any key to boot from CD or DVD...`（这是 cdboot 自己的提示），
  因为**没有输入设备**（USB 未通）→ 超时退回启动菜单 ✓；
* 于是换用同 ISO 里的 **`cdboot_noprompt.efi`**（968 112 B，无提示版）→ 应直接加载
  `\sources\boot.wim`，无需按键 ✓。

产物 `work/cust-winpe-np.img`（1000 MiB，`\EFI\Boot\BOOTAA64.EFI` = `cdboot_noprompt.efi`，
含 `\boot\boot.sdi` 与 `\sources\boot.wim`），sha256
`a4f62f1dd6290ff181b3c3a7b9d976239050ac79794965235af3e9586c8374fa`。

**这条链证明**：UEFI → Windows 的 arm64 引导器（bootmgfw/cdboot）能跑、能读 UFS 上的 FAT 分区、
能接受我们的 ACPI 与 GOP；剩下的门槛是**输入设备**（USB）。

### 3.84 用 Windows 端生成的 BCD 重建镜像（卷相对引用）

用户在 Windows 上用 `bcdedit` 生成了一份卷相对的 BCD（8192 B，`regf` hive），内容核对：
`Windows Boot Manager` + `Ramdisk options` + `\boot\boot.sdi` + `\sources\boot.wim` ✓ —— 正是需要的那种。

`work/cust-winpe-user.img`（1000 MiB FAT32，剩余 ~93 MiB）：

| 路径 | 大小 |
| --- | --- |
| `\EFI\Boot\BOOTAA64.EFI` | 2 622 896（真 bootmgr） |
| `\EFI\Microsoft\Boot\bootmgfw.efi` | 2 622 896 |
| `\EFI\Microsoft\Boot\BCD` | **8 192（用户生成）** |
| `\boot\boot.sdi` | 3 170 304 |
| `\sources\boot.wim` | 940 090 097 |

sha256 `04426425203b3f587a56a6d0155dd6dc778aa7128d7280b7a1c5e52d08781139`。

### 3.85 用官方 Win11 ARM64 ISO 的**同一套**引导文件重建（版本一致）

之前 `0xc000000d` 反复出现，判断是"PE 的 bootmgr + 用户 Windows 的 BCD + PE 的 SDI"**版本混搭**。
现在改用官方 ISO 里的一整套（全部同一版本）：

| 路径 | 来源 | 大小 |
| --- | --- | --- |
| `\EFI\Boot\BOOTAA64.EFI` | `efi/boot/bootaa64.efi` | 3 120 648 |
| `\EFI\Microsoft\Boot\bootmgfw.efi` | 同上（ISO 无单独的 bootmgfw） | 3 120 648 |
| `\EFI\Microsoft\Boot\BCD` | `efi/microsoft/boot/bcd` | 16 384 |
| `\boot\boot.sdi` | `boot/boot.sdi` | 3 170 304 |
| `\sources\boot.wim` | `sources/boot.wim` | **700 187 372（668 MiB）** |

`work/cust-win11.img`（1000 MiB FAT32，剩余 ~321 MiB）：
sha256 `f8ac1d9620409e75b58431ac57976365b953def1171a60008fe3dc1c13e9d5a7`。

（`install.wim` 7.8 GB 不放入 WinPE 分区；装系统阶段再单独处理。）

### 3.86 r66：`BCD probe` 诊断应用（判定"固件 vs Windows"）

新增 `Platforms/Xiaomi/chopinPkg/Applications/BcdProbeApp/`（`UEFI_APPLICATION`，FFS
`7A2C9E14-…`），注册为启动项 **"BCD probe"**（`PcdChopinBootLinuxDirectly` 置 FALSE，
否则清理循环会把它删掉）。它用固件自己的 `SimpleFileSystem` 读取 PE 文件并打印：

```
BCDPROBE: v<i> BCD sz=16384 at=0x20 mt=...
BCDPROBE: v<i> BCD head: 72 65 67 66 ...      (前 64 字节，4 行)
BCDPROBE: v<i> BCD fnv=0x<16 hex> regf=1
BCDPROBE: v<i> bootmgfw.efi sz=... hd=4d 5a 90 00 ...
BCDPROBE: v<i> boot.sdi sz=... hd=24 53 44 49 30 30 30 31
BCDPROBE: v<i> boot.wim sz=... hd=4d 53 57 49 4d 00 00 00
BCDPROBE: memmap lo=0x........ hi=0x........
BCDPROBE: r1 0x01000000-0x01100000 none
BCDPROBE: r2 0x0f000000-0x10200000 none
BCDFROBE: BCD size=16384 fnv=0x<16 hex> regf=1 vols=<N> memdesc=<M>
```

**判读**（主机侧已算出参考哈希）：

| 镜像 | BCD `fnv` |
| --- | --- |
| `work/cust-winpe2.img` | `0x0f509afd2b07af13` |
| `work/cust-win11.img` | `0xd1759d3a3caf88af` |

* `fnv` 与表一致 + `regf=1` + `sz=16384` → **固件读取字节精确** → `0xc000000d` 是 Windows 侧语义问题；
* `open=Not Found`/大小不符/`regf=0`/`fnv` 不一致 → **固件侧**（路径解析/FAT 目录项/簇链/读取）；
* `r1`/`r2` 预期都是 `none` —— 我们的内存映射从 `0x40000000` 起，`0x01000000` 与 `0x10000000`
  从未描述过，这正是 `ConvertPages: failed to find range` 的来源。

入口：10 秒倒计时内按住**音量上**进 Boot Manager → 选 **`BCD probe`** → **电源键**确认。

### 3.87 `BCD probe` 实测结论：**固件读取字节精确**，`0xc000000d` 是 Windows 侧

上机日志（`work/device/uefi-winpe/`）：

```
BCDPROBE: v0 vol="WINPE" tot=998MB free=94MB
BCDPROBE: v0 BCD sz=16384 at=0x20 mt=2026-10-19 14:20:06
BCDPROBE: v0 BCD head: 72 65 67 66 0e 01 00 00 ...     ← 'regf' ✓
BCDPROBE: v0 BCD fnv=0xfD7717268D149625 regf=1
BCDPROBE: v0 bootmgfw.efi sz=2622896 hd=4D 5A 90 00 03 00 00 00 ✓
BCDPROBE: v0 BOOTAA64.EFI sz=968000  hd=4D 5A 90 00 03 00 00 00 ✓   (cdboot.efi)
BCDPROBE: v0 boot.sdi sz=3170304 hd=24 53 44 49 30 30 30 31 ✓
BCDPROBE: v0 boot.wim sz=940090097 hd=4D 53 57 49 4D 00 00 00 ✓
BCDPROBE: memmap n=58 tot=4095MB  lo=0x40000000 hi=0x140000000
BCDPROBE: r1 0x01000000-0x01100000 none
BCDPROBE: r2 0x0f000000-0x10200000 none
BCDFROBE: BCD size=16384 fnv=0xfD7717268D149625 regf=1 vols=1 memdesc=58
```

主机侧对同一镜像（`work/cust-winpe-cdboot.img`）算出的 FNV 是 **`0xfd7717268d149625`**
—— 与固件打印的**完全一致** ✓✓。

**结论**：

1. **固件的 FAT/BlockIo/文件读取是字节精确的** ✓ → `0xc000000d` 不是固件读取 bug，而是
   Windows/BCD 侧语义问题（版本/设备引用）；
2. `r1`/`r2` 都是 `none` ✓ → `ConvertPages: failed to find range …` 的来源是
   **这两段从未被内存映射描述过**（chopin 的映射从 `0x40000000` 起、最高 `0x140000000`），
   不是缺页错误；
3. 顺带发现：**内存映射最高只到 `0x140000000`（5.37 GB），而机器有 8 GB** —— 之后值得单独查
   为什么没有描述到 `0x240000000`（可能影响 Windows 的物理内存识别）。

**下一步**：`cust` 目前是 cdboot 版（按键不可用 ✗）。接下来应刷 **官方 Win11 的整套**
（`work/cust-win11.img`：官方 bootmgfw + 官方 BCD + 官方 boot.sdi + 官方 boot.wim，版本天然一致），
这是还没试过、也最可能越过 BCD 的组合。

### 3.88 r67：512 字节 BlockIo 翻译层（可试，但被 **4Kn GPT** 挡住）

新增 `Silicon/MediaTek/MT6893Pkg/Drivers/BlockIo512Dxe/`（`UEFI_DRIVER`，绑定
`BlockSize==4096 && !LogicalPartition` 的整盘 BlockIo，子句柄报 `BlockSize=512`、
`LastBlock=(父+1)*8-1`、`IoAlign=max(父,4096)`；整 4K 对齐直通、非对齐走 64 KiB 弹跳缓冲
读改写 ✓；主机侧测试 5188 项全过 ✓；`Start()` 带只读自检，屏幕打印
`BlockIo512: self-check aligned 4K read OK, 512-byte bounce read OK`）。

**硬拦路虎**：这块盘的 **GPT 是 4Kn 原生**（header 在 `0x1000`、`MyLBA=1`、条目在 `0x2000`）。
`GptLib` 要求 `MyLBA == Lba`（在 `Lba*BlockSize` 处读头），所以在 512 视图里：
`LBA 1` = byte 0x200（MBR 零填充 ✗）、备份头 = 末尾 512 字节（在 4K 备份块内部 ✗）
→ `PartitionDxe` **一个分区也建不出来** ✗ → bootmgfw 仍落在 4 KiB 分区上 ✗。

**出路（择一）**：
* (A) 分区级原位替换（对 `PartitionDxe` 自己的 BlockIo 做 `ReinstallProtocolInterface`）—— 爆炸半径大 ✗；
* (B) **在读路径里合成一个 512 原生 GPT**（推荐，自包含）：把 512 视图里的 GPT 头部/条目/备份
  按 512 单位重写（`MyLBA=1`、`AlternateLBA=child LastBlock`、`PartitionEntryLBA=2`、
  每个 `Starting/EndingLBA × 8`、两处 CRC 重算），写入方向只翻译 GPT 区域的恢复写 ✓。
  代价约 400–600 行；收益是"Windows 看到真正的 512 原生磁盘"，能否解掉 `0xc000000d` 仍需实测。

### 3.89 用公开 PDB 反汇编 `bootmgfw`：`0xc000000d` = 固件返回的 `EFI_INVALID_PARAMETER`

子代理从微软符号服务器取到 bootmgfw 的**公开 PDB**（`C711E172-…`，8 898 个符号），
把整段 `.text` 的调用点都标注成真实函数名（`work/analysis/annotated.asm`）。

**关键结论**：

1. `0xC000000D` **不是** BCD 内容检查：它由 `EfiGetNtStatusCode`（`0x52e80`）把
   `EFI_INVALID_PARAMETER (0x8000…0002)` 映射而来；坏 hive 会得到 `0xC000014C`、缺文件会得到
   `0xC0000034`/`0xC000000F` —— 都不是我们看到的码。
2. 实际路径（全部符号核实）：
   `EfiEntry → BmMain → BmpOpenStoreAndUpdateOptions → BmFwOpenDataStoreWithHash → BiOpenStoreWithHash
   → BlDeviceOpen/BlImgLoadImageWithProgress2 → ImgpReadAtFileOffset → BlFileReadAtOffsetEx
   → FatMount/FatRead → BlockIoRead → BlockIoFirmwareRead → EFI_BLOCK_IO.ReadBlocks`
   → **失败发生在读 BCD 文件的那一步**，还没开始解析 hive ✓。
3. 三份 BCD（PE 的、bcdedit 的、官方 Win11 的）**全部通过** bootmgfw 的所有检查（`regf`、major 1、
   minor 3/5 都接受、bins 长度、XOR 校验和都正确）→ **BCD 侧无问题** ✓。
4. `ConvertPages: failed to find range` 是 `MmFwGetMemoryMap`（`0x1e5908`）**故意做的内存探测**
   （AllocateAnyPages + FreePages + AllocateAddress），失败后它自行处理 → **红鲱鱼** ✓。

**修法**：ESP 的 FAT 必须是 **4Kn 原生**（`BPB_BytsPerSec = 4096`）。我们的镜像用默认
`mkfs.vfat` 是 512 ✗，而真实 Windows-on-4Kn 的 ESP 一定是 4096 ✓。已用
`mkfs.fat -F 32 -S 4096 -s 1 -n WIN11PE` 生成 `work/cust-win11-4k.img`
（`BPB=4096` ✓，sha256 `8b19cb6930e01885221e096bf09ad1064189f6ffc257872af7016fa000d46499`）。

若此版仍报 `0xc000000d`，则回退到"BlockIo512 + 合成 512 GPT"（§3.88 方案 B），让 bootmgfw
端到端看到 512/512。

### 3.90 r70 实测：**所有 BlockIo 句柄都健康** → 候选 A/B 排除

`BCD probe` 的 `blkio` 段（`work/device/uefi-r70/`）显示每个分区句柄都是：

```
BCDPROBE: b<i> dp=0401 media present=1 removable=0 ro=0 lpart=1 bs=4096 align=0 last=0x..F open=EFI_SUCCESS
BCDPROBE: b<i> hd part=<n> start=0x.. size=0x.. sigtype=2 sig=<hex>
BCDPROBE: b<i> read0=EFI_SUCCESS bb=0x...
```

即：**没有死句柄**（`open=EFI_SUCCESS` ✓）、**没有 Media 异常**（`present=1/removable=0/last≠0` ✓）、
**没有重复**（各 `start/size/sig` 都不同 ✓）。因此：

1. 候选 A（`OpenProtocol` 死在坏句柄上）**排除** ✓；
2. 候选 B（`BlockIoEfiGetBlockIoInformation` 拒绝 Media）**排除** ✓；
3. 剩下的只能是 **bootmgfw 自己的描述符检查**（`PartitionOpen` / `BlpDeviceOpen`），
   即它从 `LoadedImage->DevicePath` 构造 `BL_DEVICE` 时对某个字段不满意 ✓。

**下一步**：把启动项改成**显式 `HD()+File(\EFI\Microsoft\Boot\bootmgfw.efi)`** 路径（而不是
SFS 句柄派生的短形式），让 `LoadedImage->FilePath` / 设备路径形态符合 bootmgfw 的期望 ✓。

### 3.91 r71：显式 `HD()+FilePath` 启动项 `Windows (explicit)`

`MsBootOptionsLib.c`（+413 行）新增：

* `ChopinScanWindowsBootVolume()`：枚举 SFS 句柄，找含 `\EFI\Microsoft\Boot\bootmgfw.efi`
  （或 `\EFI\Boot\BOOTAA64.EFI`）且带 `EFI_PARTITION_INFO_PROTOCOL` 的分区；
* `ChopinBuildWindowsBootDevicePath()`：卷设备路径 + `HD(Start, Size, GPT GUID)` +
  `File(\EFI\Microsoft\Boot\bootmgfw.efi)` + END（若原路径已以 HD 结尾则复用 ✓）；
* `ChopinRegisterWindowsExplicitBootOption()`：`EfiBootManagerInitializeLoadOption` +
  `AddLoadOptionVariable(-1)` 注册（**追加在最后** ✓，去重 ✓）。

主机侧用真实函数做了离线验证（`port/tools/explicit-dp-test/`）：两种输入都得到
`vendor()+HD+FilePath+END` ✓；且 `BmGetNextLoadOptionDevicePath` 对完整路径**原样返回** ✓
（不像 SFS 短形式会走 `BmExpandMediaDevicePath` 补路径 ✗）。

镜像：`chopin-uefi-DEBUG-ufs-r71-explicit.img`（843 776 B，sha256
`b5ce255586123a44d397b32b5b4720ea6bac8ce067b13f873e13f37c052c2e42`）。

### 3.92 WinPE 线收尾：已排除项与后续路线

**已经验证成功的**（这条链路的真正价值）：

| 项 | 证据 |
| --- | --- |
| arm64 Windows Boot Manager 在 chopin UEFI 上执行 | 屏幕 `Windows Boot Manager` + 加载日志 |
| 固件从 UFS 的 FAT 分区读文件字节精确 | `BCD probe` 的 FNV 与主机一致（`0xd1759d3a3caf88af`） |
| ACPI 被 Windows 接受 | bootmgr 跑到读 BCD 阶段 |
| GOP framebuffer 被 Windows 使用 | 报错界面正常显示 |
| 每个 BlockIo 句柄健康 | `open=EFI_SUCCESS`、`read0=EFI_SUCCESS`、`present=1`、`removable=0`、`last≠0` |
| 显式 `HD()+File()` 启动项 | 日志 `Booting Windows (explicit)` |

**被系统性排除的假设**（全部实测否证）：

BCD 内容/版本 · FAT 扇区大小（512 与 4096 都试过）· 分区几何与块读取 · BlockIo 句柄死/重复 ·
Media 字段 · 512 GPT 视图（`vols=1`，未产生卷）· 设备路径短形式与显式形式。

**结论**：`0xC000000D` 来自 bootmgfw **内部** `BlpDeviceOpen`/`PartitionOpen` 对 `BL_DEVICE`
的某个字段的检查；固件侧能提供的都已到位。

**后续路线（按优先级）**：

1. **USB 启动**（原生 WoA 的唯一现实路径 ✗ Windows 无 UFS 驱动 ✗）：
   先把 `XhciDxe`/`UsbBusDxe`/`UsbMassStorageDxe` 加进 FV（纯配置 ✓），再攻 MT6360 OTG 供电 ✗；
   从 U 盘启动时设备路径是标准 USB 形态 ✓，`0xC000000D` 大概率不再出现 ✓。
2. **"弹按键"路线**（用户指定保留）：`cdboot.efi` 会打印 `Press any key to boot from CD or DVD` ✓
   —— 该提示由 UEFI `ConIn` 读取；目前我们的键不被它识别 ✗（但在 UEFI 菜单里有效 ✓）。
   以后若 USB 输入可用、或修好 `SimpleTextInputEx` 的通知/`WaitForKeyEx`，这条路可以直接进 WIM ✓。
3. **Linux 侧**（独立）：`paging_init` 探针已就绪（`work/boot_b-chopin-probe.img`），
   数绿色横条即可定位死点（对照表见 §3.78）。

### 3.93 "弹按键"路线的最终结论（2026-10-20）

**输入链路本身完全正常** ✓ —— r73（`KeypadDxe` 的 `IsKeyRegistered` 改为 `SimpleTextInputEx`
通配语义）后，`Key probe` 实测：

```
KEYPROBE: notify key scan=0x1 uni=0x0        ← 通配通知生效 ✓
KEYPROBE: notify key scan=0x2 uni=0x0
KEYPROBE: notify key scan=0x0 uni=0xd        ← 电源(Enter) ✓
KEYPROBE: phase1 end notify=14 wait=7
```

三种机制（`RegisterKeyNotify` 通配 ✓、`WaitForKeyEx` ✓、`ReadKeyStrokeEx` ✓）都能看到按键 ✓。

**但 cdboot 仍然不认** ✗：

| 镜像 | 结果 |
| --- | --- |
| `cdboot.efi`（提示版） | `Press any key to boot from CD or DVD...` → 按键无效 → 超时回菜单 ✗ |
| `cdboot_noprompt.efi`（免提示版） | `Error: Image at 0x… start failed: No mapping` ✗（`EFI_NO_MAPPING`，与固件无关） |

**结论**：`cdboot` 的按键检测不在 UEFI `ConIn`/`ConInEx` 上 ✗ —— 因此**即使接 USB 键盘也
无法满足它的提示** ✗；`cdboot_noprompt` 则自身返回 `No mapping` ✗。**"弹按键"路线终止。**

**下次继续的入口**：
1. **USB 主机栈**（`XhciDxe`+`UsbBusDxe`+`UsbMassStorageDxe` 进 FV ✓ 简单）+ **MT6360 OTG 供电** ✗
   —— 让固件能枚举 U 盘；从 U 盘启动时设备路径是标准 USB 形态 ✓，`0xc000000d` 大概率消失 ✓；
2. **Linux 的 `paging_init` 探针** ✓（`work/boot_b-chopin-probe.img`，数绿色横条，§3.78 有对照表）；
3. 若继续 WoA：UFS 上只能"启动"不能"运行"（Windows 无 UFS 驱动 ✗），系统必须放 U 盘 ✓。

---

## 4. 与 MT6789 / MT6768 参考实现的差异

| 项 | MT6789（stone） | chopin | 处理 |
| --- | --- | --- | --- |
| GIC/GPIO 布局 | `0x0C000000/0x0C040000`；GPIO 同布局 | **完全相同** | 只改 `MaxPin` |
| FD base | `0x40000000` | `0x40080000` | toml + MemoryMapLib |
| PlatformHelper.S | 空 `ret` | 需要 EL2→EL1 下沉 | 抄 MT6768 版 |
| PMIC | MT6358（WACS2 `0x880…`） | **MT6359**（WACS2 `0xC80…`） | 新写 `PmicWrapperImplLib` + `MT6359PmicDxe` |
| 时钟名 | `TOP_MSDC50_0` 存在 | 只有 `TOP_MSDC50_0_SEL` | 见 3.6 |
| 存储 | eMMC/SD（MsdcDxe） | **UFS**（Mu-Silicium 原本没有驱动） | 新写 `UfsHcDxe` + 复用 EDK2 `UfsPassThruDxe`/`ScsiBus`/`ScsiDisk` |
| 帧缓冲 | `1088×2340` | `1088×2400`，存活帧 `0x78E0C000` | dsc PCD + MemoryMapLib |
| ACPI | 无（注释掉） | 无 | 需要时新建 `Silicon/MediaTek/MT6893/*` |

---

## 5. 已知缺口与下一步（按优先级）

1. **UFS 存储：已实现，待上机验证**。架构是「MTK 平台层 + EDK2 通用栈」：
   `MT6893Pkg/Drivers/UfsHcDxe` 装两个协议（`EDKII_UFS_HOST_CONTROLLER_PROTOCOL`
   提供 HCI MMIO 与 DMA/缓存维护；`EDKII_UFS_HC_PLATFORM_PROTOCOL` 提供 MTK 钩子），
   然后由 EDK2 的 `UfsPassThruDxe`（HCI/UIC/link startup）→ `ScsiBusDxe` → `ScsiDiskDxe`
   产出 BlockIo。**平台侧动作与顺序**（对齐 `ufs-mediatek.c` 的 PRE/POST_CHANGE 语义）：

   | 时机 | 动作 | 依据 |
   | --- | --- | --- |
   | init | 只开 4 个门（`INFRA_AO_UFS`/`UNIPRO_SYSCLK`/`UFS_MP_SAP_BCLK`/`TOP_UFS_SEL`），**不动 `TOP_UFS_SEL` 父时钟**（继承 LK；chopin 没有 `ufs_sel_max/min_src`，主线也不会 reparent），并打印各时钟频率/父 | `ufs-mediatek.c:964-1012` |
   | PreHce | **只复位 UFSHCI**（`infracfg_ao 0x130/0x134` bit15）→ `XOUFS_CTRL(0x140) |= 0x80` → `AHIT(0x18)=0` | vendor `ufs-mtk.c:1176` 只传 `SW_RST_TARGET_UFSHCI`；`ufs-mediatek.c:274-280` |
   | PostHce | M-PHY 上电（`phy-mtk-ufs.c` 的 4 组位操作）→ **ref clock 握手 `0x144`**（写 REQUEST bit0、轮询 ACK bit1 ≤3 ms、再等 32 µs）→ 600 µs 延时 | `ufs-mediatek.c:340-395,730-738`；`ufs-mtk-platform.c:324-367`（Samsung 32 µs） |
   | PreLinkStartup | 用 UIC `DME_GET(0xD0A8)`，若为 1 则 `DME_SET(0xD0A8,0)`（清 UniPro power-down） | vendor `ufs-mtk.c:1139-1153` |

   三条**故意不做**的事，以及理由：

   * **不复位 M-PHY**。厂商驱动从不复位它（`SW_RST_TARGET_MPHY` 无调用者，
     `SW_RST_TARGET_ALL` 也不含 MPHY）；主线会复位 mphy_rst，但随后靠 ATF 的
     `UFS_MPHY_BACKUP/RESTORE` SMC 把 PMA 内容恢复回来——**UEFI 没有这个 SIP**，
     所以复位只会丢掉 LK 写好的 PMA。同理暂不动 `unipro_rst`（厂商只在 HCE 失败恢复
     路径才加它）。
   * **不写 `REG_UFS_MMIO_OPT_CTRL_0(0x160)` 的 DDR_EN/MRTT**：主线在
     `if (host->legacy_ip_ver) return 0;` 之后才写，而 MT6893 正是 legacy
     （`IP_VER_LEGACY_VER_MT6893 = 0x20160706`）。
   * **不碰任何稳压器**（尤其不 disable `ldo_vemc`）：厂商的
     `UFS_MTK_PLATFORM_VCC_ALWAYS_ON` 就是「VCC 绝不断电」，对应主线
     `mediatek,ufs-broken-vcc`；chopin 的 UFS `vcc-supply` 是 `ldo_vemc`，
     而 1.2 V 来自 MT6360 的 `ext_buck_vufs12`（LK 日志 `VUFS12=1200000`）。

   另外 EDK2 侧要求平台保证的对齐：`AllocateBuffer` 用 4 KiB 对齐
   （UTRD 需 128 B、UTRLBA 需 1024 B）。**未经上机验证**：链路能否在
   UfsPassThruDxe 下完成 link startup、以及那 4 组 PMA 位操作是否够
   （主线/厂商都没有更多 PMA 序列可抄：厂商 `ufs_mtk_init_mphy()` 是空函数，
   M-PHY 序列本体在 preloader/LK 里，本仓库无源码）。
2. **电源**：MT6360（SD 卡与充电，i2c5）+ MT6359 regulator 表
   （`mt6359-regulator.c`，VEMC 还带 HW trapping/TMA key）。在此之前 SD 卡不保证上电。
3. **ACPI**：Windows on Arm 需要。Silicium-ACPI 里只有 MT6768，需要新建
   `Silicon/MediaTek/MT6893/*` 与 `Platforms/Xiaomi/chopin/AcpiTables.inf`，
   再在 dsc `[Components]` 与 fdf 各加一行（`chopin.dsc` 里已留注释位）。
4. **屏幕双候选**：LK 的显示 mblock `0x77A20000+0x27E0000` 是 4 帧环，
   `0x78E0C000` 是第 2 帧；热重启后 LK 可能把面板留在另一帧（`HANDOFF.md:794`），
   届时 UEFI 的输出会看不见。需要做双候选探测或像内核那样镜像两块。
5. **时钟控制器覆盖**：`imp_iic_wrap`（I2C）、`mmsys/mdpsys/vdecsys` 等 130+ 个门
   没有映射（`MTK_CLOCK_CONTROLLER_TYPE` 只有三个控制器）。显示真正需要时再扩。
6. **EL2/KVM 交接**：见 3.4 的取舍。

---

## 6. 构建、校验与上机

### 6.1 构建（Debian + clang 19 实测）

```bash
cd /home/kily/project/mt6893/Mu-Silicium
export CLANG_BIN=/usr/lib/llvm-19/bin          # 关键：否则 CLANGPDB 找不到 lld-link/llvm-lib/llvm-rc
export PATH="$CLANG_BIN:$PATH"
git submodule update --init --recursive        # 首次，约 3 GB
python3 build_uefi.py -d chopin -r DEBUG       # DEBUG：日志画到屏幕（无 UART，见下）
python3 build_uefi.py -d chopin -r RELEASE     # 正式版无调试输出
```

移植时踩过的两个构建坑（都已修在 chopin 侧）：

* `ScsiBusDxe`/`ScsiDiskDxe` 需要 `UefiScsiLib`，而 `SiliciumPkg.dsc.inc` 没有映射它
  （它在 `MdePkg/Library/UefiScsiLib`），所以 `chopin.dsc` 的 `[LibraryClasses]`
  里显式加了一行；别的 MTK 平台要用 UFS/SCSI 栈也会需要。
* `clang 19 -Werror` 会把上游 `MediaTekPkg/Drivers/MsdcDxe` 里两处死代码当错误
  （见第 1 节的上游改动）。

产物：
* FD：`Build/chopinPkg/<TARGET>_CLANGPDB/FV/SILICIUM_UEFI.fd`（固定 2 MiB，
  开头是一条 `b +0x1000` 跳到 FV 里的 SEC —— LK 跳到 `0x40080000` 正好执行它）
* 镜像：`Mu-chopin.img`（boot.img v2，2048 字节页）

2026-10-16 在本机（clang 19.1.7）实际构建出来的产物，已归档到 `port/dist/`：

| 文件 | 大小 | 说明 | sha256 |
| --- | --- | --- | --- |
| `chopin-uefi-DEBUG-ufs-r64.img` | 831488 | **最新，建议刷这个**：**内核搬到 4 GiB 以下**（`AllocateMaxAddress` + 拷贝） | `01f7cfaa5d7d7e03a975b6e5b1268f703e173c45bc0ef035eeb01bccb41aef67` |
| `chopin-uefi-RELEASE-ufs-r64.img` | 698368 | 同上 RELEASE | `374be9c361fa01f9d5a8370992dc9070705d0378851d950616410b7ca6d9448c` |
| `chopin-uefi-DEBUG-ufs-r63.img` | 831488 | 上过机：DTB/initrd 都对、内核起来了（测试条），但仍不进 init | `7a32ab66310855714e19c2797c98a71383b4960dd9e22dc6aa6466aac88e2bb1` |
| `chopin-uefi-RELEASE-ufs-r63.img` | 698368 | 同上 RELEASE | `397959618da9736bfe18be2991f4a9f35149ddb5032ce04401d39422511944d6` |
| `chopin-uefi-DEBUG-ufs-r62.img` | 831488 | 上过机：行跨距回退 1088 ✓，但新载荷的 DTB 未内联 → 加载器拒绝启动 | `135e1bff1c6428b9c10c088cc3b882ec14bd96533aaa875640719dfecea81573` |
| `chopin-uefi-RELEASE-ufs-r62.img` | 698368 | 同上 RELEASE | `72f2f4502f5c7e939985ff1b4e9cb00bcee6ad8bbacf4453760171a7de8a1d49` |
| `chopin-uefi-DEBUG-ufs-r61.img` | 831488 | 上过机：ramdisk 固定 0x51100000（面板 stride 试了 1080 → 撕屏，已回退） | `358fd098d593022d69fc14dfa7d25fafc9771bc1f19eda877d2e731f6ebb3b8a` |
| `chopin-uefi-RELEASE-ufs-r61.img` | 698368 | 同上 RELEASE | `1eb27399fa7279b60df6d475216f3cebef09d79b616d2ce2e6b9afbbc005441b` |
| `chopin-uefi-DEBUG-ufs-r60.img` | 831488 | 上过机：GOP 行跨距 1088→1080 | `96f0c4fbed57d9640b0735beb51f4f3050ed16f9be6905e9566a4394f6b586ba` |
| `chopin-uefi-RELEASE-ufs-r60.img` | 698368 | 同上 RELEASE | `f156c215cfa1152c24cd99cf218956b96a522d52c6a16f8fe75a8622d92ad640` |
| `chopin-uefi-DEBUG-ufs-r59.img` | 831488 | 上过机：initrd 传递正确、handoff 成功，但内核只出绿条 | `e4b018697cdcb03eb11a84c802f8e83d1634ed4a10ce84a9d1c603aba3d62435` |
| `chopin-uefi-RELEASE-ufs-r59.img` | 698368 | 同上 RELEASE | `b0755d979a26def3926fbc6840c5346b9388827965c8bef2b5ff605f5dc15f10` |
| `chopin-uefi-DEBUG-ufs-r58.img` | 831488 | 上过机：TOPCKGEN 通了，但 I2C 控制器寄存器仍全 0 → A 两轮结束 | `7c8dd3bffa29cce5442569f9b0736a52c2ba9324ae52f46a677cfb52534c3078` |
| `chopin-uefi-RELEASE-ufs-r58.img` | 698368 | 同上 RELEASE | `2ebf09eb4a0581f8630bc8e0bb1d94b882a4eed7d945097963c315852cff4112` |
| `chopin-uefi-DEBUG-ufs-r57.img` | 831488 | 上过机：dump 显示 8 条 I2C 寄存器全为 0（模块时钟未开） | `b131ab494054afc9f86caf54006c03513352881a07231c1b57819bcf6d673cc2` |
| `chopin-uefi-RELEASE-ufs-r57.img` | 698368 | 同上 RELEASE | `9b6aaa1cf8af348ae641da4534c742feb7d421c8b87da923d4139af3a7c41054` |
| `chopin-uefi-DEBUG-ufs-r56.img` | 831488 | 上过机：8 条 I2C 全部 Timeout（控制器未运行） | `c752e4e4c607fcc905ef0d137f03fe455c793501799d1fbd820545c50c5cb8a9` |
| `chopin-uefi-RELEASE-ufs-r56.img` | 698368 | 同上 RELEASE | `bb1fe2191a6d3f8ab30679fa0f38bc02e372bb168c2f7688d962c661dd5199f2` |
| `chopin-uefi-DEBUG-ufs-r55.img` | 829440 | 上过机：I2C 超时（总线选错 + 控制器未初始化） | `9d0f647a0e70836eb6f3d6e6320c74d606836cd91280558b06c147ada8aff8aa` |
| `chopin-uefi-RELEASE-ufs-r55.img` | 696320 | 同上 RELEASE | `c556e1c09ecd4f88419308c09310d9b924246be5ed11989b5aa940dffe97c3c2` |
| `chopin-uefi-DEBUG-ufs-r54.img` | 829440 | 上过机：进菜单正常，但 USB 无供电（hub 灯不亮） | `64bd204b93ad6e6781ccfdfc04f6c29476df1f4b2331f058d459e8d07434d9d4` |
| `chopin-uefi-RELEASE-ufs-r54.img` | 696320 | 同上 RELEASE | `1f418f514cf1ab974beeef26593d2067a05bb66b45618b69095d11d71ab2d217` |
| `chopin-uefi-DEBUG-ufs-r53.img` | 829440 | 上过机：USB 区域映射后，仍崩在 `0x11D0110C`（I2C 基址错） | `4e58b6f4f1165d4efaff462f0de6c5652a5845d2c714c356c59a9cc5093ad3c9` |
| `chopin-uefi-RELEASE-ufs-r53.img` | 696320 | 同上 RELEASE | `cb1c31955ea567a59fef0ec60abeb03715c480666ceb64a163ee834646853dd7` |
| `chopin-uefi-DEBUG-ufs-r52.img` | 829440 | 上过机：`MtkUsbDxe` 在 `0x11203E08` 触发 Translation fault（区域未映射） | `d8ad908b65c6b0c84942315d41a8fc1db367be505bd2d41d7be7182fd112dc0b` |
| `chopin-uefi-RELEASE-ufs-r52.img` | 696320 | 同上 RELEASE | `86f78f709e92de4f9306aed739fb0c2e69819d0aa51bd64df89db554955725e4` |
| `chopin-uefi-DEBUG-ufs-r51.img` | 825344 | 上过机：原生 boot manager 正常（菜单 + 按键 + UFS LUN 都出现） | `b8dba47a5e8415bc87187038e0ef2dc9f50e3a821c025adee668ecfc40790fa8` |
| `chopin-uefi-RELEASE-ufs-r51.img` | 696320 | 同上 RELEASE | `071356c2f21741e1be4b3f011289f2c60315251654541d6876578969cffaae57` |
| `chopin-uefi-DEBUG-ufs-r50.img` | 835584 | 上过机：自研菜单（默认 Linux (UFS)） | `9b0ceefe00836eb9c49cad061d2472cd6a4527dbb39f733884b04122e1a63077` |
| `chopin-uefi-RELEASE-ufs-r50.img` | 702464 | 同上 RELEASE | `3eef9b38f0ba4ee745b9c869b0eef79b2cb1bf00660c28e90be19d5e99cbc78b` |
| `chopin-uefi-DEBUG-ufs-r49.img` | 825344 | 上过机：keypad 修好（无幻影键），但菜单已被移除 | `dc8de71d4f4c8aa2c9564b4789aeee0f03c9fc3f375fa1001ca4102031eb5076` |
| `chopin-uefi-RELEASE-ufs-r49.img` | 696320 | 同上 RELEASE | `c95151be6161d17b0365d1714809b058e9eaa3861532d41e8b159bb9c28baeae` |
| `chopin-uefi-DEBUG-ufs-r48.img` | 825344 | 上过机：initrd 已写进 DTB，但 init 仍未运行 | `df4b4259c3fd7dbec421349f2859e07c55dd366cecc35efc14fa11a88e66ac19` |
| `chopin-uefi-RELEASE-ufs-r48.img` | 696320 | 同上 RELEASE | `0af90c943d6326fa04b1dec3c716d35cb9cdd31efae74dcc351696e71cb6f689` |
| `chopin-uefi-DEBUG-ufs-r47.img` | 825344 | 上过机：内核起来了（进度条），但没拿到 initramfs → init 未运行 | `391e9c4caed81709121b57e9361beb75ed1e319cf3af5625fd1071b20d491df6` |
| `chopin-uefi-RELEASE-ufs-r47.img` | 696320 | 同上 RELEASE | `8ebe428d212f3c6b16878c2592dd0a775cdbbfc67714f9512629fd2c694aeb41` |
| `chopin-uefi-DEBUG-ufs-r46.img` | 825344 | 上过机：手动交接执行成功但无内核输出（无 fbdev） | `3805e8ab772fe5ef08226c5def1237400b4e547f52867bfe15fa543739913dd8` |
| `chopin-uefi-RELEASE-ufs-r46.img` | 696320 | 同上 RELEASE | `ad4d83c0eb7d1aabcf0430f7b328c2b1399e1b32608367b1e47b1f87efea9149` |
| `chopin-uefi-DEBUG-ufs-r45.img` | 825344 | 上过机：BlockIo 等待成功、GPT 正确，但选了 `boot_a`（UEFI 自身，无 DTB） | `3db5261c94b8fa660257a08aebb7b67df4974daf817b1417e2f29fc364174e56` |
| `chopin-uefi-RELEASE-ufs-r45.img` | 696320 | 同上 RELEASE | `ad6b954b0778067aa32b24bc04e0c344da5ac3845275c55be3a2d1a743064c9f` |
| `chopin-uefi-DEBUG-ufs-r44.img` | 823296 | 上过机：**GOP 显示成功**（文字居中），但加载器报 `no BlockIo handle found` | `09e39c4ea0eb114af28065c9098503a341582fc3b3148d5269c9490087359fc8` |
| `chopin-uefi-DEBUG-ufs-r43.img` | 823296 | 上过机：启动顺序只剩 Linux (UFS)，加载器仍报 no BlockIo | `c342c800bd258f23955dafab7833a232579ba6926c3c9970340bd3f5901d1bf4` |
| `chopin-uefi-RELEASE-ufs-r43.img` | 694272 | 同上 RELEASE | `52ec0b196f2ffd8695617edc4f138a9c8c644ddb4591e4484c6d247dbc2b83a9` |
| `chopin-uefi-DEBUG-ufs-r42.img` | 823296 | 上过机：菜单没了，但旧启动项仍先于 Linux (UFS) 被执行 | `350d4b014697678b43a0b836c387883538c0e094eb650d9cec5cb8ffbd3289b2` |
| `chopin-uefi-RELEASE-ufs-r42.img` | 694272 | 同上 RELEASE | `085e88fedbf9f7c6e1933af2e665b64a434037dba5cd101d505c4a9525d8827f` |
| `chopin-uefi-DEBUG-ufs-r41.img` | 833536 | 上过机：菜单仍被 NVRAM 里的旧启动项唤起 | `76355cf87cde3c935d32b07892df2143a03862c53faf7e6c54b8b5b5f1853d6f` |
| `chopin-uefi-RELEASE-ufs-r41.img` | 700416 | 同上 RELEASE | `c197e36e53ac5a1a999b9c42fb83dfbffb38bde7d5ffe78fcb084c6c05797c4f` |
| `chopin-uefi-DEBUG-ufs-r40.img` | 833536 | 上过机：菜单里出现 "Linux (UFS)"，但菜单本身不是想要的形式 | `b87c181d5bbeb4119b5cff45799c024762ac3465843e4314d2f125e44ffecf67` |
| `chopin-uefi-RELEASE-ufs-r40.img` | 700416 | 同上 RELEASE | `21dda164994de6fe74d6f8c2981d3d8c19e976fb307b2e12b01803ac46fbbc14` |
| `chopin-uefi-DEBUG-ufs-r39.img` | 833536 | 上过机：`dispatch -> Not Found`，加载器未被派发 | `3fb1bfa6f5764ac68553ea3bf4c12da5d4e48cd4e127a4d833f8bfb787e47ed7` |
| `chopin-uefi-RELEASE-ufs-r39.img` | 700416 | 同上 RELEASE | `79f926d938c1c980d73d910983263b300d7dc93b15cc28e86cfd9041ddb19523` |
| `chopin-uefi-DEBUG-ufs-r38.img` | 833536 | 上过机：菜单正常但加载器没被派发（depex 等不到 Dispatch） | `a5fe0309ca47fdaf6e12f5babdf46279159c7fbced7d95a239c2193722a250df` |
| `chopin-uefi-RELEASE-ufs-r38.img` | 700416 | 同上 RELEASE | `a0d19a2b537ca032204e2a96ca74880a773569ac9ed998de7e94004eee8e2b0a` |
| `chopin-uefi-DEBUG-ufs-r37.img` | 827392 | 上过机：幻影键被过滤，菜单静止 | `827f37e2a3d9411a8cfd4d0d673709e265656b9d96e0a7b860a5ec5e4e2a3303` |
| `chopin-uefi-RELEASE-ufs-r37.img` | 696320 | 同上 RELEASE | `1e12081d67534c9d5dca8de4659d433db25d4909708fb6d29be7ca1310e262b1` |
| `chopin-uefi-DEBUG-ufs-r36.img` | 827392 | 上过机：键码打出来了——`sc=0001/0002` 无限交替（keypad 驱动缺陷） | `71d6f48044658a9e6d92306958c7798307f43daeb0e2f8d5c1c4dc626c13aa0c` |
| `chopin-uefi-RELEASE-ufs-r36.img` | 696320 | 同上 RELEASE | `550e4cc06026bb397c12bdf6a92bbf2b0d687bb53cfc64593b57cbcf6c3ad1ab` |
| `chopin-uefi-DEBUG-ufs-r35.img` | 827392 | 上过机：仍自动刷屏（幻影键恒定重复） | `8153d1d8044d0de21ae5802f51eca5a448f4d4a03a9610207a25aaac98530fe4` |
| `chopin-uefi-RELEASE-ufs-r35.img` | 696320 | 同上 RELEASE | `fe478c35dfba94fd108ea6472e623f8324fbfac2cd1db762d3287406b26ad446` |
| `chopin-uefi-DEBUG-ufs-r34.img` | 827392 | 上过机：不按键也自动刷屏（幻影键事件） | `d066db3065a983aa1401c5123cd82e09fd3f734d796bf6a048cbfcbeac1722e0` |
| `chopin-uefi-RELEASE-ufs-r34.img` | 696320 | 同上 RELEASE | `251286f308ba99bf4f28a327ec1558e97efe05c8c2d4f1a26567a1ddbf018d5f` |
| `chopin-uefi-DEBUG-ufs-r33.img` | 827392 | 上过机：按键通了但连读刷屏（一次按键走多步） | `872edb34c91efc8cbcb7eac242417714253d807f78bfb3bd324bfee0bae18db1` |
| `chopin-uefi-RELEASE-ufs-r33.img` | 696320 | 同上 RELEASE | `8b4ef8629ff74b24b3235f1ed5efb5529e335979a9033fce38ae0036ed95fdb8` |
| `chopin-uefi-DEBUG-ufs-r32.img` | 827392 | 上过机：菜单可见但刷屏、按键无反应 | `121aa0368d9038b32915e6c0338b4c22143c192fec16b26f485e33c0d0e48b63` |
| `chopin-uefi-RELEASE-ufs-r32.img` | 696320 | 同上 RELEASE | `173226370e3ea9caa9944a43ec43db16d185e7d3d140ebbbd18d712e6f371cd1` |
| `chopin-uefi-DEBUG-ufs-r31.img` | 827392 | 上过机：应用跑到 `drawing the menu now` 后无输出 → 证实 ConOut 不显示 | `53995ab5755aa899a23e749cfb64c25d67df69aed7dabdc68b327022b20b5d85` |
| `chopin-uefi-RELEASE-ufs-r31.img` | 696320 | 同上 RELEASE | `c4f2e1b8253d0228094cc3bf52206401776583b92ecef9345c15a51944949faa` |
| `chopin-uefi-DEBUG-ufs-r30.img` | 827392 | 上过机：`BootMenuApp` 被启动，但随即黑屏 | `9e274584f2f58c377fabf471f9129b06e4283b5d20f059382a208f5284d2cb69` |
| `chopin-uefi-RELEASE-ufs-r30.img` | 696320 | 同上 RELEASE | `a6fabbf0639f7c23dc092229d8cf3c1325a9e057852560958821cbf7f40b6140` |
| `chopin-uefi-DEBUG-ufs-r29.img` | 827392 | 上过机：有 logo 但黑屏（菜单未排在启动顺序前面） | `3b972d531bc76c2bb2a1b926ff2e46ebc0717db61a4c7a2023d96fcf1c63be0e` |
| `chopin-uefi-RELEASE-ufs-r29.img` | 696320 | 同上 RELEASE | `74c3d41fcd0cab51f403f39df90b0859c979d99406f67413320001f7650568a7` |
| `chopin-uefi-DEBUG-ufs-r28.img` | 815104 | 上过机：INFO 关闭；菜单仍未出现（BootManagerMenuApp 画完 logo 就退出） | `d05be0ffb290810a5e8d7991c003f6edee1ea19d19b05bf746643906cdd4809e` |
| `chopin-uefi-RELEASE-ufs-r28.img` | 688128 | 同上 RELEASE | `e55978b48758610234729b2bd8e87a2a3692a5625d7438774a8cc31aed2206f2` |
| `chopin-uefi-DEBUG-ufs-r27.img` | 815104 | UFS 首次打通（BlockIo/ScsiDisk 枚举成功） | `78613d749c09b12d029acbd259d30b66893664c8ce27d06ba9ba7b833e1ea09a` |
| `chopin-uefi-RELEASE-ufs-r27.img` | 688128 | 同上 RELEASE | `e55978b48758610234729b2bd8e87a2a3692a5625d7438774a8cc31aed2206f2` |
| `chopin-uefi-DEBUG-ufs-r26.img` | 815104 | 上过机：接管 + `DME_HIBERNATE_EXIT`，链路在位但 `OCS` 仍 `0f` | `f78b8c16dcaff6a7b86ab3d4588978a3d646b118f5f5ea9e4b1dd530ac5d95b6` |
| `chopin-uefi-RELEASE-ufs-r26.img` | 688128 | 同上 RELEASE | `536a2491ba3b2cf9671cad232b8b86cf179312de111f85ebbe8e86343066868d` |
| `chopin-uefi-DEBUG-ufs-r25.img` | 817152 | 上过机：拍到 `st=1` 时 `HCS=0xf`（DP=1），证实是我们打断了链路 | `01652d00885487ca9c7dc792c1662c73cf69b404ee4d53290470103957306f65` |
| `chopin-uefi-RELEASE-ufs-r25.img` | 688128 | 同上 RELEASE | `f5752931dd05f6d409de4c96ad8cb8ede266456d4df673e8c632ca70577aa3f4` |
| `chopin-uefi-DEBUG-ufs-r24.img` | 817152 | 上过机：`DP=0`，链路仍未起 | `031c3a5b1f0a4559c3ecdb6e7cb0abef07ea5f38b3ee5724104f2cee7997e69d` |
| `chopin-uefi-RELEASE-ufs-r24.img` | 688128 | 同上 RELEASE | `1d49a4a50fb5962c52021d13c9c8ca8cc2bb3aa4abf5be1b5150cc7cd34d2f3d` |
| `chopin-uefi-DEBUG-ufs-r23.img` | 817152 | 上过机：`DP=0`、链路没起来（缺 M-PHY 上电）——由 r24 补上 | `8116d57d2e95fa2d68506cd9c64c51ba1285d4aef0b73440476df0b16d49a34d` |
| `chopin-uefi-RELEASE-ufs-r23.img` | 688128 | 同上 RELEASE | `9efe6865eeebe1d7b64514cf5f941a1b9e010af6d7abfce29d10e5058abc87c6` |
| `chopin-uefi-DEBUG-ufs-r22.img` | 815104 | 上过机：`st=6`，按键重连后更早失败 → 证实继承状态不可靠 | `c590f4456ccb12c63e8c5175a66f07ef8d42f286b8276310216d69eed5a7629b` |
| `chopin-uefi-RELEASE-ufs-r22.img` | 688128 | 同上 RELEASE | `67d8fc85b6e8ed73bdfa8fbe7930d50f160c7072ee300547d1829357dbd56108` |
| `chopin-uefi-DEBUG-ufs-r21.img` | 815104 | 上过机：新增 OCS 有效时刻采样（未及时用于判断） | `11c7331c00d0fbb423e5cc3ffb7eabb4f040e382130f86fd2e8a3b19e145530d` |
| `chopin-uefi-RELEASE-ufs-r21.img` | 688128 | 同上 RELEASE | `4feec25069eabee8941bc2c040655617efb6d4e894cc982de12b42da04c5a969` |
| `chopin-uefi-DEBUG-ufs-r20.img` | 815104 | 上过机：MMU 修好了，UFS 状态与 r17 相同（`blk=0`） | `5cf102e8e4ea4863a5132aa3e3a4591f3110fc0361c533155744dd0933e5947a` |
| `chopin-uefi-RELEASE-ufs-r20.img` | 688128 | 同上 RELEASE | `da865aa80bff9c1a59f3458e64e15254e0059d233a95e1494097d7967fe96892` |
| `chopin-uefi-DEBUG-ufs-r19.img` | 815104 | 上过机：MMU 配置失败（新增区域未 4 KiB 对齐）——由 r20 修正 | `da11a587daf60a9727d261bc3471ba5d8c297e5bb50f23326aa737cc10587f59` |
| `chopin-uefi-RELEASE-ufs-r19.img` | 688128 | 同上 RELEASE | `7c8127cb476cd98fa0352f8e4579953119ac74f0dcceeff2b675f959082a1792` |
| `chopin-uefi-DEBUG-ufs-r18.img` | 806912 | 上过机：**黑屏**（SEC 阶段写未映射的日志环）——由 r19 修正 | `6802231b3e650fb47bd525a778c48236de67f08bad719cf329725d1c80929a2b` |
| `chopin-uefi-RELEASE-ufs-r18.img` | 688128 | 同上 RELEASE | `abd6f716eb8d1b640f17ae56fea85381b6e57106b2d68f1ab85cfdd4e91d3674` |
| `chopin-uefi-DEBUG-ufs-r17.img` | 815104 | 上过机：接管模式下 `TL=RL=1`（数据通路活着）但 `UCD=FFFFFFFF` → 定位到 UTRL 基址写入被忽略 | `2fa476f5a2147890ac20a7f52c9816b221ddffdb4ac52256c995f70b1dbafe3b` |
| `chopin-uefi-RELEASE-ufs-r17.img` | 688128 | 同上 RELEASE | `f33e55dbc98b5fcb594b3972971396c79c7fcb03bec751a4d62278bc2255980e` |
| `chopin-uefi-DEBUG-ufs-r16.img` | 815104 | 上过机：电源模式变更未生效（`PWR/TL/RL` 仍 0） | `61c7e8f668ffcefb14a7a7c0881743ac13fe5949b1632d98e9edf3b0d5b60048` |
| `chopin-uefi-RELEASE-ufs-r16.img` | 688128 | 同上 RELEASE | `b261ba22ec2a19c3b4274484b1fa4092a505f6d29551796b86f9b47c9d778449` |
| `chopin-uefi-DEBUG-ufs-r15.img` | 815104 | 上过机：DME 读回证明**数据通路未建立**（PWR/TL/RL 全 0） | `9502f2edc2f88030addabc1d69bae2e6e6bea4386ce9361f4efb60e83e1b4ede` |
| `chopin-uefi-RELEASE-ufs-r15.img` | 688128 | 同上 RELEASE | `849b2f47ff2bb3a4053e05260acd8e0916f43fd67903a0a35b6cc2dfcc2cc316` |
| `chopin-uefi-DEBUG-ufs-r14.img` | 815104 | 上过机：**崩溃**（我在定时器里用了已释放的驱动接口 → Alignment fault），已由 r15 修正 | `783129457ab020a5204561b21efc4deaee96518adbc343e7c91d89f96e045216` |
| `chopin-uefi-RELEASE-ufs-r14.img` | 688128 | 同上 RELEASE | `a97c5a88d053bd82d886ef1d24c4d4aef69e8e214232cd4d8c5e58421ea55ad9` |
| `chopin-uefi-DEBUG-ufs-r13.img` | 815104 | 上过机：DME 编码修正 + 关 deep stall/LCC，但 `OCS` 仍 `0f`、仍无盘 | `c5579779ebdc6190e427ab2d3b8ac0b10d73e9829d183fc79982da716e9888f0` |
| `chopin-uefi-RELEASE-ufs-r13.img` | 688128 | 同上 RELEASE | `d4a18cb55b40805efd5e2653705edbc48a43426361d3b4c2f8ccfd240cc19dd8` |
| `chopin-uefi-DEBUG-ufs-r12.img` | 815104 | 上过机：`HCS/RSR/DBR/UBA` 全部正常 → 排除主机侧，定位到数据通路 | `ad0ee1ef26793dccb9fc5061d88b7cccdab99fc8c6451c2e04d168988225192f` |
| `chopin-uefi-RELEASE-ufs-r12.img` | 688128 | 同上 RELEASE | `7b2c821f3c99c896cee8b02ab23c4ee54fdf395dc95d341db9cb846763f72628` |
| `chopin-uefi-DEBUG-ufs-r10.img` | 815104 | 上过机：状态行给出 `OCS=0f×4`、`UEC` 全 0 → 定位到 UTP 层 | `62a22adbbb6427e2b70457ac75933309e8f0f5c2119d231c1d7e1a72d56d3c85` |
| `chopin-uefi-RELEASE-ufs-r10.img` | 688128 | 同上 RELEASE | `7b2c821f3c99c896cee8b02ab23c4ee54fdf395dc95d341db9cb846763f72628` |
| `chopin-uefi-DEBUG-ufs-r9.img` | 815104 | r8 + 阶段扩到 10（“Start 进来了”2、“CAP/VER 读成功”3）+ 打印 VER/CAP + 驱动内主动 ConnectController | `31a4daa0aab88732f03c1d2f7dfb5351b09ff3b2e3753feedba668595c760c5d` |
| `chopin-uefi-RELEASE-ufs-r9.img` | 688128 | 同上 RELEASE | `f59e7888780eb793abd50622d97df26a74e05effcc45cccd50d7a4edd41ea19a` |
| `chopin-uefi-DEBUG-ufs-r8.img` | 815104 | 上过机：标记可见，实测 **1 绿 / 0 黄**（UFS 停在自己初始化完，HCE 回调未触发） | `bb683685ccb251b8b8f36d5a594cd2404f7d239dda838a87156d13f691880614` |
| `chopin-uefi-RELEASE-ufs-r8.img` | 688128 | 同上 RELEASE | `2f5470d94c28f313fc99eb6f78631cb438b81c45e78c49cf961de2cd62014522` |
| `chopin-uefi-DEBUG-ufs-r7.img` | 815104 | 上过机：**不再崩溃**，跑到 BDS；标记因 alpha=0 不可见 | `abe4c63e858793bfc55bd46adcd9c7a837cbe5bf3cbc37e0364dff996f235d04` |
| `chopin-uefi-RELEASE-ufs-r7.img` | 688128 | 同上 RELEASE | `445a4bbea9e446d8e624100e2db667f987884f719a2fa7893fa5683bb443422f` |
| `chopin-uefi-DEBUG-ufs-r5.img` | 815104 | 上过机：UFS 初始化成功但 23 s 后 `Data abort`（FAR=0），已由 r7 修正 | `fc79611e4e924780b5ed48da5c1984deac3061cbcbc8931967033ced98fb3a4f` |
| `chopin-uefi-RELEASE-ufs-r5.img` | 688128 | 同上 RELEASE | `298973fdc6730d65a32575ee45b87e8b533308321b2ba922030d46ac48ab8aea` |
| `chopin-uefi-DEBUG-ufs-r4.img` | 815104 | 上过机：**不再崩溃**，BDS 停在等按键（标记被 BDS 重画覆盖） | `bbe6fd95dceca851a2a6e9ce89aafd54027fc8f8342d0145b6ac26cef167d3df` |
| `chopin-uefi-RELEASE-ufs-r4.img` | 688128 | 同上 RELEASE | `6a27a6fd1c0ae87a26a6f431c027eebf505cb80e654e7d0b137c0a142a2f3fd2` |
| `chopin-uefi-DEBUG-ufs-r3.img` | 813056 | 上过机：过缓存断言后遇 CPU 异常 | `4aaa0fa4570962cb2b64be856904be816045ac5a18982a376c47c00ff64a76b0` |
| `chopin-uefi-RELEASE-ufs-r3.img` | 688128 | 同上 RELEASE（无屏幕日志、ASSERT 不停机的兜底） | `c08cd798eaf46c585549fa25290e884fb686b174f86cf0b0b7c5981a5a04c22d` |
| `chopin-uefi-DEBUG-ufs-r2.img` | 813056 | 上过机的 UFS 版：卡在 `ArmCacheMaintenanceLib` 断言 | `08422cf25206cc7a4f760e22bd96086487839bda495a628737b90a3b68c190bf` |
| `chopin-uefi-DEBUG-r2.img` | 788480 | 上过机：BDS 已起来（无 UFS） | `7aeaad4d3ae55719645a7352e836a9b4ee2aaeec1dd77e8aaf7cb3ca0fa9d28e` |
| 其余 `*-ufs.img` / `DEBUG.img` / `RELEASE.img` | — | 更早的两版，仅存档 | — |

**上机请用 DEBUG 版**：`SiliciumPkg` 只在 DEBUG 构建里把 console 绑到
`FrameBufferSerialPortLib`（画在 LK 留下的 framebuffer 上），RELEASE 版是 Null console，
屏幕上不会有日志（BDS 界面仍会显示）。DEBUG 版遇到 `ASSERT` 会停在原地——这正是
第一次上机能定位时钟名问题的原因；RELEASE 版则会把断言当无事发生继续跑，适合兜底。

### 6.2 校验产物

```bash
python3 port/tools/verify-chopin-uefi-img.py port/dist/chopin-uefi-DEBUG-ufs.img --expect-kernel-size 0x200000
python3 port/tools/check-chopin-uefi-memory-map.py \
    Mu-Silicium/Platforms/Xiaomi/chopinPkg/Library/MemoryMapLib/MemoryMapLib.c
```

第一个脚本逐项比对 `kernel_addr/ramdisk_addr/tags_addr/dtb_addr/page_size/
header_version/header_size`、`cmdline` 的 `bootopt=` 前缀，以及 kernel 的 gzip 头
（必须 `FLG=0`）；`--fix-gzip` 能在「页数不变」的前提下就地重压。
第二个脚本检查 MemoryMapLib：DDR/MMIO 各自不重叠、MMIO 不落在 DDR 里、
以及所有被其它模块按名字查找的区域都在（重叠会让 `ValidateMemoryMap()` 直接
`CpuDeadLoop()`）。两者在本次产物上都是 PASS。

### 6.3 上机（谨慎，先准备好回滚）

1. 备份原厂：`fastboot getvar all` + 至少保存一份能刷回去的原厂 `boot.img`
   （或确认手里有完整原厂包）。解锁 bootloader 是前提。
2. 冷启动进 fastboot，`fastboot flash boot Mu-chopin.img`，`fastboot reboot`。
3. **观测只有屏幕**：这台机器没有引出串口（`HANDOFF.md:50`），
   所以要用 **DEBUG 构建**——`SiliciumPkg` 在 DEBUG 下把 console 换成
   `FrameBufferSerialPortLib`，会直接在 LK 留下的 framebuffer 上画日志。
4. 若无输出：优先怀疑「面板当前扫的是另一帧」（第 5.4 节），冷启动重试一次；
   再怀疑 LK 是否真的跳转（用 `port/tools` 里的 expdb 取日志流程看 LK 的 `boot linux-64`）。
5. 回滚：`fastboot flash boot <原厂 boot.img>`。
6. 交互：音量上/下（KPD）+ 电源键（MT6359）已经接上；UEFI BDS 里可用它们导航。

### 6.4 上机前必须知道的三个风险

* **内部存储是否可用还没验证**（UFS 驱动这一版才加进去）；SD 卡因为 MT6360 没有驱动
  不保证上电。ACPI 也还没有，所以现阶段验证的是「LK → SEC → DXE → BDS + 屏幕 + 按键
  (+ 存储枚举)」这条链路。
* 看门狗：LK/preloader 已经把 31 秒的狗武装起来，`PlatformSecLib` 在 SEC 第一阶段
  就关掉它；如果 SEC 在关狗之前卡住（比如 FD 没被正确加载），设备会 31 秒重启。
* GZ 仍在链条里（除非刷了 NoGZ 的 tee.img）。UEFI 在 EL1、GZ 在 EL2 是允许的组合，
  但任何 EL2 相关的期望（KVM）都不成立。

---

## 7. 仍未确证 / 下一步要做的验证

| # | 未确证项 | 状态 / 怎么证 |
| --- | --- | --- |
| 1 | LK 不校验 ARM64 头 | ✅ **已上机证实**：raw FD 直接起来，没有 BootShim |
| 2 | `0x40280000` 起栈是否空闲 | ✅ 已上机证实（SEC/DXE 跑到 DXE 驱动加载阶段） |
| 3 | `bootopt` 与 64 位交接 | ✅ 已上机证实（GZ 无 32 位 abort） |
| 4 | `0x78E0C000` 是否每次都是存活帧 | ⬜ 冷/热启动各试一次，必要时实现双候选 |
| 5 | MT6359 pwrap 在 UEFI 下可读 | 🟡 间接证实（`MT6359PmicDxe` 被 dispatch）；按键响应仍要试 |
| 6 | **UFS 能否在 EDK2 栈下完成 link startup 并枚举出磁盘** | ⬜ 下一次上机：看屏幕是否有 `UfsPassThru`/`ScsiDisk` 日志与 `BlockIo` |
| 7 | MTK M-PHY 的 4 组位操作是否足够（主线无 PMA 序列表） | ⬜ 同上；若 link startup 失败，用 `work/vendor-ufs/` 交叉核对 PMA |
| 8 | SD 卡（MSDC1）与 MT6360 上电 | ⬜ 需要先写 MT6360 驱动 |
| 9 | ACPI / Windows on Arm | ⬜ 需要新建 MT6893 ACPI 表 |
