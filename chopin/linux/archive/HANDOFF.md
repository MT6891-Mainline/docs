# chopin (Redmi Note 10 Pro / MT6893) 主线移植 —— 交接文档

> 面向下个会话。详细历史见 `port/SUMMARY.md`（27+ 轮，倒序），操作细节见 `port/NEXT.md`。
> 本文：背景 → 现状（含证据）→ 仓库与纪律 → 下一步（显示 / USB）→ 工具与陷阱。
>
> **UEFI 线（2026-10-16 起）见 `port/UEFI-PORT.md`**：Mu-Silicium 里已新增
> MT6893 硅层包 + chopin 平台包，能构建出与厂商逐字段一致的 boot.img；
> 硬件参数证据在 `port/audit/mt6893-uefi-hardware-params.md`，
> Mu-Silicium 代码解剖在 `port/audit/mu-silicium-mtk-port-anatomy.md`。
>
> **Wi-Fi 线（2026-10-18 起）见 `port/WIFI-PORT.md`**：整套 downstream
> connectivity（conninfra + CONNAC2X2 SOC3_0 fullmac）已从 MT6895-Mainline
> 的 7.2 化代码搬进来，Kconfig/Makefile/defconfig/DT 全部接好，Image + dtb
> 构建通过。固件与 NVRAM 不随内核走，用 initramfs 新增的 `fwload` 从
> vendor 逻辑分区 / `mediatek/mt6893/WIFI` 取。上机 bring-up 是下一步。

---

## 2026-10-18：WiFi 关联 + DHCP 成功

- 扫描/关联/握手/DHCP 全通：`myron` 5745MHz WPA2-PSK CCMP，`udhcpc` 拿到
  `10.141.253.115/24`。
- Alpine 3.24.2 作为 initramfs root 启动；`wpa_supplicant`/`iw` 由
  `tools/build-alpine-rootfs.sh` 离线装入；`/usr/bin/wifi-connect` 一键连接。
- init 增加 PID 1 电源信号处理（修复 `reboot` 卡住）、真实 rootfs 启动模式
  （`chopin.root=userdata`，参考 mt6895 的 switch_root 流程）。
- chopin 的 `userdata` = `/dev/sdc75`；rootfs 目前未启用（等发行版镜像）。

## 一、背景

### 设备与目标
- **Xiaomi Redmi Note 10 Pro（中国版）**，代号 **chopin**，SoC **MT6893 / MT6891Z**（Dimensity 1100），
  8 GiB DRAM @ `0x40000000`，序列号 `ib6dsk6pxofarglb`，机型码 `M2104K10AC`。
- 目标：把 **Linux 7.2.8 主线内核**跑起来并逐步 bring-up 外设（PMIC → 存储 → USB → 显示）。
- 前作：同代 **MT6895-Mainline**（`7.2-mt6895-xiaomi-xaga`）与 **MT6893-Mainline**，本移植多处机制取自它们，
  提交信息里都注明了出处。

### 引导链（关键，决定了所有方案）
```
preloader → LK → GenieZone (GZ, EL2 虚拟机监控器) → Linux (EL1)
```
**没有 U-Boot，也没有 UEFI。** 由此产生的三个硬约束：

1. **`bootopt=64S3,32N2,64N2` 必须出现在 boot.img 头部的 cmdline**，否则 GZ 会以 32 位模式启动内核。
2. **boot.img 里必须放厂商自己的 device tree。** LK 会把 `dtbo` 分区的覆盖层套在 boot.img 的 DTB 上，
   而这个覆盖层需要基树里的 **60 个符号**（`mmc_fixed_3v3`、`reserved_memory`、`chosen`、`i2c0..9`、
   `gpio`、`touch`、`usb0phy`、`mtkfb`、`dsi0`、`charger` …）。给它一棵精简树，fixup 就会失败，
   **LK 根本不会启动内核**：
   ```
   ERROR: ufdt_overlay_do_fixups():No node __symbols__ in main dtb.
   ERROR: ufdt_overlay_do_fixups():Couldn't find 'mmc_fixed_3v3' symbol in main dtb
   ufdt_apply_overlay() failed!
   boot image dtb_overlay fail
   ```
   → 所以采用 **MT6895 的办法：把我们的 DTB 编进内核，在 `setup_arch()` 里替换掉 LK 传进来的那棵**。
3. **LK 会在运行时注入整个 `mblock` 保留内存布局**（其日志里的 `mblock_reserve-R` / `mblock_create`），
   而 boot.img 的 DTB 里**没有**这些。内核一旦自己分配了 modem/GZ/显示 拥有的内存，就会出事 ——
   这正是当初 UFS 首次 DMA 触发 `arm64_serror_panic` 的根因。

### 可观测性（为什么有那个日志驱动）

> **⭐ ringfix31 起：有 USB 串口了 —— 这是首选的看日志方式。**
>
> 设备模式下 gadget 在 PC 上枚举为
> `0525:a4a7 Netchip Technology, Inc. Linux-USB Serial Gadget (CDC ACM mode)`，
> PC 侧出现 `/dev/ttyACM0`，initramfs 把日志推到 `/dev/ttyGS0`。
> **不要再拍照了**（`screen /dev/ttyACM0 115200`）；而且 ringfix32 起
> `/dev/ttyACM0` 上还有**一个交互式 shell**（见第二节「USB」末）。
> 屏幕/expdb 保留为备用通道（比如 gadget 没起来时）。

设备没有可用串口，且 `pstore/ramoops` 在此平台**完全不工作**（复位后 28/28 次 dump 全是 0）。
preloader 与 LK 都会往 `0x7ffbf000` 的 DRAM 环（256 KiB）追加日志，由 `0x11df00` 的常驻 SRAM 头描述，
LK 会在下次启动时把这个环的尾部抄进 **expdb 分区**。于是内核侧的
`drivers/misc/mtk-logstore-console.c` 把 printk 镜像进这个环，并用 expdb 把它取回来。

---

## 二、现状

### 已验证可用

| 子系统 | 状态 | 证据 |
| --- | --- | --- |
| 内核启动到用户空间 | ✅ | `CHOPIN MAINLINE: userspace reached (PID 1)` |
| 用上自己的设备树 | ✅ | `CHOPIN-DTB: overriding LK FDT with embedded board tree` |
| reserved-memory | ✅ | 29 个固定区、`mblock-` 63 次、`OF: reserved mem` 92 次 |
| PMIC（pwrap + MT6359） | ✅ | `mt6397-rtc mt6359-rtc: registered as rtc0` |
| **UFS 存储** | ✅ | `scsi host0: ufshcd` + `sd 0:0:0:2 [sdc] 31240192 4096-byte logical blocks: (128 GB/119 GiB)`；ringfix26 起四个控制器复位也接上了（见第二节「第九层」） |
| 内存安全 | ✅ | `arm64_serror_panic` = 0 |
| 早期可观测性 | ✅ | `[BC 00]…[BC 05+]` 面包屑 + 屏幕条纹（head.S 4 条 + setup.c/driver） |
| **显示（framebuffer）** | ✅ ringfix24 | **用户确认屏幕工作**；`mblock-19` 加 `no-map` + simplefb `reg` 收敛 + 条纹改 `ioremap_wc`（见第五节）。expdb 存证待补 |
| **USB（设备模式 + 串口控制台）** | ✅ ringfix31/32 | PC 上枚举 `0525:a4a7` + `/dev/ttyACM0`；`udc state` = `configured`。**日志从此不用拍照**；ringfix32 起该端口还是个 shell。见第二节「USB」 |
| **PMIC 按键** | ✅ ringfix37 | `input: mtk-pmic-keys as .../mt6359-keys/input/input0`（补 MFD 子节点 + `CONFIG_KEYBOARD_MTK_PMIC=y` 的副产物） |
| **不再自发重启（看门狗）** | ✅ ringfix33 | `mtk-wdt 10007000.toprgu: Watchdog enabled (timeout=31 sec, nowayout=0)`，稳定运行远超 4 个超时周期。见第二节「看门狗」 |

**UFS 报错链共剥了八层**（详见 SUMMARY）：保留区缺失 → 时钟名 → 时钟表长 → `phys` →
vcc 固定稳压器 → PHY probe 逐级报错 → `#phy-cells` → **`mediatek,ufs-disable-mcq`**
（触发 `UFSHCD_QUIRK_BROKEN_LSDBS_CAP`，最后一把钥匙）。

**第九层（ringfix26，已修）**：UFS 的四个控制器复位没接上 —— `ufs-mediatek.c:231` 用
非 optional 的 `devm_reset_control_get()` 按名字取，DT 里没有就报
`Failed to get reset control hci_rst/unipro_rst/crypto_rst/mphy_rst: -2`，四个句柄全为 NULL，
于是 `ufs_mtk_host_reset()`（在 HCE enable 时 assert 全部四个 → 等 ~100µs → deassert）
**是个空操作**，控制器只能靠引导器留下的状态。

两处 DT 缺一不可：provider 侧 `/infracfg_ao@10001000` 要有 `#reset-cells = <1>`
（`clk-mt6893-infra_ao.c` 通过 `mtk_register_reset_controller_with_dev()` 注册，
`#reset-cells` 取自这个节点；厂商树不需要它，因为厂商 UFS 驱动是 syscon 直写 INFRA_RST）；
consumer 侧 `/ufshci@11270000` 要有 `resets` + `reset-names`。

两个名字对不上的对应关系**不是猜的**，是从厂商内核（`n10p/kernel_mt6893-chopin_kvm.zip`
→ 已解到 `work/vendor-ufs/`）的 `mediatek/mt6893/ufs-mtk-platform.h` 逐位核出来的：

| 驱动名字 | DT id | idx | SET / CLR | bit | 厂商宏 |
| --- | --- | --- | --- | --- | --- |
| `hci_rst` | 0 | 47 | `0x130` / `0x134` | 15 | `REG_UFSHCI_SW_RST_*` |
| `unipro_rst` | 1 | 71 | `0x140` / `0x144` | 7 | `REG_UNIPRO_SW_RST_*` |
| `crypto_rst` | 3 | 117 | `0x150` / `0x154` | 21 | `REG_UFSCPT_SW_RST_*` |
| `mphy_rst` | 2 | 73 | `0x140` / `0x144` | 9 | `REG_UFSPHY_SW_RST_*` |

厂商源码自己给出了那两个"名字对不上"的答案：`SW_RST_TARGET_UFSCPT` 分支的注释是
**"reset AES"**（所以 UFSCPT = crypto），而 `SW_RST_TARGET_MPHY` 分支操作的是 `REG_UFSPHY_*`
（所以 mphy = UFSPHY）。

### SPI 时钟（ringfix27，已修）

八个 SPI 控制器在基树里全是 `status = "okay"`，而八个**全部 probe 失败**，每次启动 16 行：

```
spi0..3  "error -EINVAL: failed to clk_set_parent"   + "probe with driver mtk-spi failed with error -22"
spi4..7  "error -ENOENT: failed to get spi-clk"      + "probe with driver mtk-spi failed with error -2"
```

根因和 UFS 时钟同源：基树是**厂商扁平树**，时钟引用是**厂商 ID**，而我们的主线 clk 驱动编号不同。
**厂商内核里根本没有 mt6893 的时钟头** —— 它复用 **MT6885** 的（它自己的 pinctrl 节点就写着
`mediatek,mt6885-pinctrl`）。所以解码必须用厂商树的
`include/dt-bindings/clock/mt6885-clk.h`（已解到 `work/vendor-clk/`）。对上之后每个节点都是：

| DT 里 | 厂商宏 | 角色 |
| --- | --- | --- |
| `&topckgen 0x53` = 83 | `TOP_MAINPLL_D5_D4` | `parent-clk` |
| `&topckgen 0x1c` = 28 | `TOP_MUX_SPI` | `sel-clk` |
| `&infracfg_ao 0x1e/0x3b/0x3d/0x3e` | `INFRACFG_AO_SPI0/1/2/3_CG` | `spi-clk`（spi0-3） |
| `&infracfg_ao 0x4d/0x4e/0x6c/0x6d` | `INFRACFG_AO_SPI4/5/6/7_CK_CG` | `spi-clk`（spi4-7） |

这些索引在我们 ID 空间里落到别处：主线 `CLK_TOP_SPI_SEL` 是 **111**，所以厂商的 28 是另一个
topckgen 时钟、**不是 mux** → `clk_set_parent()` 报 -EINVAL；而厂商的门索引
30/59/61/62/77/78/108/109 要么落在我们 clk 表的空隙里、要么超出末尾（注册 78 个门）→ -ENOENT。

修法：给 8 个节点重述三路时钟，父与 mux 八个共用 ——
`parent-clk = CLK_TOP_MAINPLL_D5_D4`（厂商自己的选择，且是 `spi_sel` 四个合法父之一）、
`sel-clk = CLK_TOP_SPI_SEL`、`spi-clk = CLK_INFRA_AO_SPI0..SPI5 / _SPI6_CK / _SPI7_CK`。

⚠️ **这个头文件里混着好几套 ID 空间**：`CLK_TOP_*`(174)、`CLK_INFRA_AO_*`(74)，还有
`CLK_MDP_*`、`CLK_MM_*`……**数值全部重叠**（22 同时是 `CLK_INFRA_AO_SPI0` 和别的空间里的东西）。
**按数值反查名字会静默取错**，必须先由 phandle 定到 provider，再在该 provider 的命名空间里查。
这个坑我踩了两次才写对校验脚本。

**收益为零，成本很低**：8 条总线上只有 spi5 挂了个 `st54spi@0`（`compatible = "st,st54spi"`，
ST54 NFC），而**主线没有这个驱动**（`drivers/nfc/` 里 grep `st54` 零命中）。所以这轮修的是
"控制器可用"，不是"NFC 能用"。

### USB：✅ 已跑通（ringfix31，含设备模式串口控制台）

**结论先说**：设备模式下 gadget 在 PC 上枚举成功（`0525:a4a7`）、`/dev/ttyACM0` 可用、
日志可以直接读；`/sys/class/udc/11201000.usb/state` = **`configured`**。
**病根是 PHY 的模拟侧**：厂商 `mu3phy` 的 `phy_init_soc()` 里有两步主线不做的事 ——
① `U3D_USB30_PHYA_REG1(u3_sif2_base+0xb04)` 的 `RG_SSUSB_VUSB09_ON`(bit29) 打开 U2 PHY 的
**0.9V 模拟电源**；② `U2PHYDTM1` 的 `FORCE_VBUSVALID(13)/FORCE_AVALID(10)/FORCE_SESSEND(12)`
**强制 session 有效**（主线只写旁边的 `RG_` 值 bit5/2/4，依赖硬件 VBUS 比较器，而这块板的
PHY VBUS 脚没接到 Type-C 的 VBUS —— 由充电 IC 感知）。所以补丁用 DT 属性
`mediatek,force-session-valid` 限定只对本板生效，并在 `power_off` 里对称清掉；
u2port 的 `reg` 窗口从 `0x400` 扩到 `0x1000`（`0xb04` 在外面）。

**关键判据**（这轮就是靠它一分为二的）：`udc state`
—— `not attached`（我们这侧没拉起来，当时 IRQ 计数也是 0）vs `configured`（链路已起）。
PC 侧判据是 `lsusb` 里的 **`0525:a4a7`**（传统 `g_serial` 的默认 ID）。

**另外两件事**：
- **ringfix32 起 `/dev/ttyACM0` 上是一个交互式 shell**（init.c 里自己写的，因为 initramfs
  只有一个静态二进制、host 的 busybox 是 x86-64）：`help`/`dmesg [substr]`/`cat`/`head`/`ls`/`lsr`/
  `grep`/`write`/`clk`/`irq`/`udc`/`phy`/`usb`/`reboot`/`poweroff`。
  **从此排查不用再刷机**。shell 输出用 `usb_only` 只走 USB，不污染 ring/expdb。
- **`panic=0`**（ringfix32 起）：之前 `panic=15` 会在 panic 后 15 秒自动重启，形成 15 秒一轮的
  重启环，而 pstore 是坏的、panic 信息只在屏幕上停 15 秒。现在改成挂住不重启，方便拍照/读串口；
  代价是 panic 后要**长按电源键**手动复位。顺带修掉了 `build-images.sh` 默认 cmdline 里
  **`panic=15` 出现两次**的重复 bug。


- 整条主机栈能 probe：`mtu3` + `xhci-mtk` + `mediatek,generic-tphy-v2` 三个都起来，r28 里
  `xHCI platform device register success...` / `new USB bus registered, assigned bus number 1`，
  控制器读得出 `hcc params 0x01400479 hci version 0x110`。
- 设备模式也起得来：r29 里 `dr_mode: 2`、`IP version 0x1001(U3 IP)`、
  `gadget (high-speed) pullup D+`，且 initramfs 里 **`/dev/ttyGS0` 存在**（g_serial 绑定了）。
- PC 侧证明**硬件通路本身没问题**：LK 的 fastboot 在这台 PC 上枚举成功过
  （`idVendor=18d1 idProduct=d00d`，`SerialNumber: ib6dsk6pxofarglb` = 本机）。

**卡在哪**：设备模式下图集枚举不出来 —— PC 侧 `lsusb` 没有新设备，PC `dmesg` 里
**一次都没有 `0525:a4a7`**（传统 `g_serial` 的默认 ID；看到它才算我们的 gadget 真的枚举了）。

**已经排除/查清的**（都是读了源码，不是猜）：

| 事项 | 结论 |
| --- | --- |
| PHY 基址/子块偏移 | **对的**。主线 V2 的 `U2PHY_COM=0x300`、`U2FREQ=0x100` 与厂商 `mtk-phy-asic.h` 的 `+0x300`/`+0x100` 完全一致（U3 那几项不同，我们只接 USB2） |
| 主线 xhci 的 ippc | `platform_get_resource_byname(..., "ippc")`，拿不到就 `has_ippc=false` 跳过整个 IP 配置；**但 mtu3 的 `ssusb_host_setup()` 本来就会替子 xhci 上电/使能端口**，所以子 xhci 只给 `"mac"` 是上游通行写法（mt8195/MT6895/MediaTek 官方 mt6893.dtsi 都如此） |
| `dr_mode=peripheral` 下 PHY 的 device 强制位 | **没人写**。`ssusb_set_mode()` 在 `dr_mode != OTG` 时直接 `return`；`grep phy_set_mode drivers/usb/mtu3/` **零命中**；主线 `phy-mtk-tphy.c` 里 **没有** role-switch/`software-role-switch` 支持，所以参考板那句 `mediatek,software-role-switch` 在 7.2.8 是空属性 |
| 时钟 | mtu3/xhci 的时钟全是 `devm_clk_bulk_get_optional`；按 MediaTek 官方 mt6893.dtsi 补齐了 `sys_ck=CLK_TOP_USB_TOP_SEL` / `ref_ck=CLK_INFRA_AO_SSUSB` / `xhci_ck=CLK_INFRA_AO_SSUSB_XHCI` |

**构建顺序**：`ringfix28` = 主机模式（栈起来了，根集线器 0 端口待查）；
`ringfix29` = 设备模式 + `CONFIG_USB_G_SERIAL=y` + init 把日志 tee 到 `/dev/ttyGS0`；
`ringfix30` = **纯诊断**（UDC state、tphy debugfs、`/dev/mem` 读 U2 PHY 寄存器、USB 时钟、中断）。

那次二选一的答案是 **`not attached`**（IRQ 计数 0），所以问题在我们这侧，然后顺着
"主线 `u2_phy_instance_power_on()` 与厂商 `phy_init_soc()` 的差异"找到了上面那两步。
`ringfix30` 加的那段诊断（UDC state / tphy debugfs / USB 时钟 / 中断）值得留着 ——
`ringfix32` 的 shell 里有 `udc`、`phy`、`usb`、`clk`、`irq` 五个命令就是它。

**顺带**：init.c 的 `print_file()` 一直用 `open()+read()`，**列不了目录** —— 所以从 r25 起
`init: simplefb bound :` 那行一直是空的（预存在的小 bug，r30 里补了真正的 `dump_dir_tree()`）。


| 子系统 | 状态 | 已知情况 |
| --- | --- | --- |
| **USB** | ⏸ 已隔离 | 基树里 `usb3@11200000` / `usb_xhci@11200000` / `usb0phy@11e40000` 被置 `disabled`；开着时 mtu3 是内核最后打印的东西（t≈0.36 s），且它在其余设备 initcall 之前跑，把别的驱动全挡住了 |

> 显示代码已上机验证可用但**尚未进 git 仓库**（4 个文件：`chopin-board.dts`、
> `mtk-logstore-console.c`、`chopin_defconfig`、`head.S` 的注释）。
> `port/tools/audit-repo.py` 会把这 4 个报成 `OLD COPY IN REPO`（退出码 1），属预期。

---

### 看门狗：板子每 31 秒自己重启（ringfix33，已修）

**症状**：板子自己定时重启，而 `panic=0` 也止不住。原因是 **`panic=` 只管 panic 之后的行为，
管不了硬件复位** —— `panic=15` 表现为"15 秒一轮重启环"，`panic=0` 表现为"挂住、然后被看门狗踹掉"，
两者现象很像，**极易误判成"panic 没生效"**。

**根因**：基树里 `toprgu@10007000` 的 compatible 是 `mediatek,toprgu`，而**主线 `mtk_wdt` 的兼容表里
没有这个字符串** → 驱动从不 probe → bootloader 武装的看门狗**没人接管、没人喂** → 31 秒后硬复位。

两个反直觉的点，务必记住：

1. **驱动一直就在内核里。** 符号叫 **`CONFIG_MEDIATEK_WATCHDOG`**（**不是** `CONFIG_MTK_WDT` ——
   我第一次就写错了，而且它因为 `default ARCH_MEDIATEK` **一直是 `=y`**）。
   第一次 grep 构建日志返回 0 还把我误导了一次：**增量构建根本不会重编那个文件**，
   日志里没有它 ≠ 它没编进去。**缺的只是一行设备树绑定，不是驱动。**
2. **喂狗的机制早就开着了。** `CONFIG_WATCHDOG_HANDLE_BOOT_ENABLED=y` 的作用正是
   "bootloader 已启动的看门狗，在用户空间接管之前由内核核心代喂" —— 但它要求
   **先有驱动 probe 并标记 `WDOG_HW_RUNNING`**（`mtk_wdt_init()` 就是干这个的）。
   我们的 initramfs 没有 watchdog 守护进程，所以驱动不 probe 就必然被复位。

**修法**：`compatible = "mediatek,mt6589-wdt", "mediatek,toprgu";`。
选 `mt6589-wdt`（data 为 NULL）是**故意选最保守的**：各 SoC 的 data 只差"软件复位线数量"，
我们树里没有任何消费者，而这个条目**不注册 reset controller**，零假设。
`mediatek,disable-extrst` **故意不加**：`WDT_MODE_EXRST_EN` 在本 SoC 的语义还没确认，
而止住复位应该靠**喂狗**，不是改它的复位语义。

**验证**：`mtk-wdt 10007000.toprgu: Watchdog enabled (timeout=31 sec, nowayout=0)`，
之后稳定运行远超 4 个超时周期（用户实测 2 分钟以上无重启）。
`timeout=31` 正是 `WDT_MAX_TIMEOUT`，与"约 30 秒一轮"的现象吻合。

### ⭐ 当前采用基线：**ringfix61**（2026-10-02）

`port/dist/chopin-ringfix61-host-settle-stockdt.img`
sha256 `84dd3a3ef88aa089da67f568fa62aad315ab4cafa1d4098d1fc3d6f0e4a3fdb0`

对应提交：内核 **`5baaae9`**、port **`f782c93`**。它包含本会话的全部成果：
**真充电 + 电池状态 + 电流可控 + 热重启屏幕可读 + 热拔插 shell 自愈 + USB host 模式（含切回 device 可靠）**。
（回退良品：ringfix59 `3279b638a026d067d0363ddfb2596996e59f646012e824b3aeb29033bc4eaf97` ✓、
ringfix60 `2e3057f1986da873c8e0bc913307acadada118a19359462da0b7aa6d92c5f98c` ✓。）

历史基线 **ringfix37**（`bae616304a4e21817e9a1b2a221ff0b6921d42b2bc99bc78726d093277203404` ✓，提交 `d834b63` ✓）
是"能启动、显示正常、无 hang"的干净对照 ✓，仍可用于二分定位 ✓。
r38–r42 那串显示排查**已试过、不采用、已从源码撤销** ✓（备份在 `work/audit/backup-r38to42-*/` ✓，镜像仍在 `port/dist/` ✓）——
**它想解决的问题已由 r55/r56 的"缓冲区镜像"方案解决** ✓（见下文对应小节 ✓）。

### 显示：✅ **已解决**（ringfix55–56，2026-10-02）—— 下面是当时钉死机制的完整证据链

> **结果**：驱动每 2 秒把 fbcon 的缓冲（CAND0 `0x78e0c000`）镜像到另一个候选（CAND1 `0x77a20000`）✓，
> 走驱动已持有的 **NonCacheable 映射** ✓（`ioremap_wc` ✓，写成 `memcpy` + `wmb()` ✓）。
> **面板扫哪一块都能看到文字** ✓ ⇒ 热重启后不再"只剩条纹" ✓（用户已实测 ✓）。
> 细节见下面的「热重启后只剩条纹/像变砖」一节 ✓。
> **下面这些结论仍然有效** ✓（活缓冲会变 ✓、三类定位手段都不可靠 ✓、不要读显示寄存器 ✓），
> 只是**不再需要**去猜 LK 的配置了 ✓。

**现象**（历史）：有些启动里屏幕停在 LK 那一屏、看不到内核日志；有些启动里正常。
**根因**（已用实验钉死）：**LK 每次启动留给内核的帧缓冲地址会变**，而我们 DTB 里写死一个
（`/chosen/framebuffer@78e0c000`），所以只有"LK 恰好留了这个地址"的那些启动里 fbcon 才可见。

**证据链**（都是实测，不是推理）：

1. 启动条纹颜色就是为此设计的：**绿 = `0x78e0c000` 活**、**红 = `0x77a20000` 活**；
   用户在 r37 看到绿、在更早的启动里看到红 ✓ → **活缓冲确实每 boot 不同**
2. 用 `fbwrite`（r39 加的一次性工具）往 `0x78e0c000` 画品红色带 → **屏幕立刻出现** ✓
   往 `0x78416000`（寄存器里读到的候选）画 → **毫无变化** ✗ → 那些寄存器是残留
3. **不是卡死**：任何一次都能用 USB 串口/shell（`/dev/ttyACM0`、`gt_serial` 枚举 `0525:a4a7`）——
   内核一直在正常跑，屏幕上"卡住"的样子骗了我们好几轮

**试过但失败的三条路**（别再重复）：

| 路线 | 为什么断 |
| --- | --- |
| 读显示寄存器找活地址 | `ovl0+0xf60` 等读到 `0x78416000/0x78416870`，但画上去屏幕没反应 → **残留**；真正在扫的 `0x78e0c000` **不在**这些寄存器里（扫了 6 个 OVL + 4 个 RDMA + DSI + MDP） |
| LK 的 `atag,videolfb-fb_base_l/h` | 本机 LK **不更新**它：发布的是厂商旧值 `0x7e020000`（落在 TEE 保留区，显然不是活地址） |
| 读 LK 打进 DRAM 环的日志文本 | 环里（`/sys/kernel/debug/logstore/{low,high}`）**找不到 `fb_layer` 那类句子**；这条只做到"无害地试一下"就随 r42 一起撤了 |

⚠️ **另有一条副作用教训**：r38–r41 曾在**每次启动**读 ~48 KB 显示寄存器，而其中不少是"读即清除"
的状态位、面板活着又只靠 LK 留下的状态 —— 诊断不该能扰动被诊断对象。**以后要读显示寄存器，
用手动触发**（r39 的 `fbwrite`、`fbscanf` 那类工具已经证明有用）。r42 已把自动扫描撤掉。

**当时的结论**（已被 r56 取代 ✓）：日志以 USB 串口为准 ✓、屏幕不可靠 ✗。
**现在**：屏幕**可读** ✓（镜像方案 ✓），并且它是"USB 串口不在时"的**唯一现场通道** ✓
（r53 那次失去两条通道的教训就是这么来的 ✓）。
**仍然成立**：想彻底摆脱对 LK 遗留状态的依赖，正解是**写 DRM/DSI 驱动自己接管显示** ✓
（那时连镜像都不需要 ✓）—— 这仍是一个值得做的**后续**项目 ✓。

### 报错清理（ringfix34，用户实测三项全部消失）

| 报错 | 根因 | 修法 |
| --- | --- | --- |
| `<0> Initramfs unpacking failed: invalid magic` | 我们内嵌 DTB 的 `/chosen` 带着**厂商的陈旧 initrd 范围**（`0x51100000..0x524d748b` = 厂商那块 20.7 MB ramdisk）。MTK 的 LK 按 boot.img 头把 ramdisk 放到同一地址但**不在 FDT 里公布范围**，于是内核按陈旧范围解包：先正确解出我们的 297 KB（**所以 `/init` 照跑**），再读后面 ~20 MB 垃圾就撞坏 magic | 打包时用 `port/tools/fdt-initrd.py` **就地改写 Image 内嵌副本**的这两个属性（必须在 gzip 前）。**MTK 自己的构建也这么干** —— 厂商树里那俩值就是这么来的 |
| `<4> OF: fdt: not creating '/sys/firmware/fdt': CRC check failed` | 换树代码直接赋值 `initial_boot_params`，而 `of_fdt_crc32` 早前是对 **LK 的树**算的 → 后来的检查拿我们的树比 LK 的校验和 | 改用 `early_init_dt_verify()`（重校验头部 + **重算 CRC**）。副作用正面：`/sys/firmware/fdt` 现在真的会出现 |
| `<4> mt6359-keys / mt6359-accdet: Failed to locate of_node [id: -1]` ×2 | 与 RTC 那次**同类但更彻底**：RTC 是节点在、compatible 写错；这两个是基树里**压根没有节点**，而 `mt6397-core.c` 的 `mt6359_devs[]` 要它们 | 补 `keys`/`accdet` 子节点；顺带把 `CONFIG_KEYBOARD_MTK_PMIC` 从 `=m` 改成 `=y`（原来根本没加载）→ 可能白捡电源/Home 硬件按键。子节点顺序对应驱动 `irqnames[] = {"powerkey","homekey"}`。**`accdet` 主线无驱动**，只是消警告 |

**故意保留的三个**（都查过，不是缺陷）：

| 报错 | 为什么不动 |
| --- | --- |
| `<4> [Firmware Bug]: Kernel image misaligned at boot` | `build-images.sh` 顶部记着**原厂** boot.img 实测布局：`kernel 13284146 bytes @ 0x40080000` —— **厂商自己的内核也在同一非 2MB 对齐地址**。这是 MTK 引导链固有属性，改它要偏离原厂布局（LK 风险）换一条装饰性警告 |
| `<5> ufshcd-mtk: find ti,syscon-reset fail` | 本移植改用主线 reset 框架接那四个复位，不需要厂商 syscon；补上反而重复处理 |
| `<4> regulator: Not disabling unused regulators` | 来自我们自己 cmdline 的 `regulator_ignore_unused`（**承重**：保住 bootloader 配好的稳压器） |

**同一轮里还揪出一个我们自己的 bug（ringfix35）**：shell 的输出隔离**没做全** ——
`redirect_to_kmsg()` 把 fd 1 dup 成了 `/dev/kmsg`，而只有走 `kmsg_tx()` 的路径才检查 `usb_only`；
`dump_dir_tree()` 用的是裸 `printf`，于是 `ls /sys/firmware/` 时**每条目录项都往内核日志灌一行**
（症状：`dmesg` 里出现 `  /sys/firmware//devicetree/base/__symbols__/xxx = `，**`= ` 后面是空的**，
因为文件内容走 USB 而前缀走了 printk；双斜杠来自命令里带的尾斜杠）。两个后果：把启动日志挤出
环形缓冲（黑匣子被冲掉），以及每行进一次 printk 走到控制台、**慢到每秒一行**。
修法：进 shell 前 `dup2(sh_fd, 0/1/2)`，把三个 fd 整个切到 USB 串口；`usb_only` 保留用于 kmsg_tx
以免同一行写两遍。

### ⚠️ 黑匣子曾经是坏的（ringfix36 修）

`ringfix33` 加的黑匣子**从诞生起就没真正工作过**：驱动里 `win_write()` 写完之后
**没有推进窗口游标**（全文件对 `low_off`/`high_off` 只有初始化赋值，没有一次写后自增）。
结果是**每一条 printk 都覆盖在窗口同一处**，环里只留得下**最后一条**日志 —— 挂死之后
拿不到任何历史。而面包屑 `[BC xx]` 之所以一直在，是因为它们走的是**另一条路径**
（自己的 `bc_low` 游标 + `early_ioremap`，见 `mtk_logstore_breadcrumb*`）。

这也解释了为什么这个项目一直那么依赖**屏幕**：环里其实从来没有过内核日志。
`ringfix36` 补上 `*cursor += len;`。**从 r36 起，环才真正是"能扛复位的黑匣子"。**

**r37 又揪出第二个、更隐蔽的 bug**：console 回调 `logstore_write()` 重复写入 —— 因为
`vprintk_emit()`（`kernel/printk/printk.c:2484`）里的钩子**在记录提交之前就已经镜像了一遍**，
而 console 带着 `CON_PRINTBUFFER | CON_ENABLED` 又写一遍，两个游标独立 → 窗口里交错成乱码。
**游标 bug 掩盖了它**（第二遍正好覆盖第一遍），游标一修就暴露了 ✓。
r37 把 `logstore_write()` 改成**空操作**（保留注册，不贡献字节）→ 环里是干净有序的日志 ✓。

**读黑匣子的正确姿势**（ringfix37 起实测有效）：

```
tail 40 /sys/kernel/debug/logstore/high     # 最新内容（窗口是"旧→新"）
find <pat> 20 /sys/kernel/debug/logstore/high   # 命中后打印 20 行上下文
```

⚠️ **环是多个写者共享的**：preloader、LK、内核各用**自己的游标**写同一个环，
所以 `cat` 出来会混着 `[PWRAP]`/`[PMIC]`/`[UFS]`/`[LK_BOOT]` 这些 **LK 的东西**。
**内核的行有 `[  <时间>]` 前缀**，按这个区分。

**显示控制器的活缓冲探测（ringfix37，结果是否定的）**：读 `ovl0+0x000c`（`DISP_REG_OVL_EN`）
作为"块活没活"的探针，外加 6 个候选地址寄存器。实测：
**探针 = `0x00000001`（块上电、可读）**，但 `rdma0/rdma1 +0x0f00`、`ovl0/ovl1 +0x0f40`、
`disp_rdma4`、`mdp_rdma0` **全是 0** → **这些偏移对 MT6893 是错的** ✗。
也就是说"活缓冲是哪个"这个问题**还没答**，需要去找 LK logo 路径真正用的那个寄存器
（而不是继续猜）。好消息：**r37 屏幕上一切正常**，所以这个问题不阻塞。

**另外**：低窗口（`0x800..0x10000`）是 **LK 自己写日志的区域，每次启动都会被覆盖** ——
你 dump 出来的 `[PWRAP]`/`[PMIC]`/`[UFS]`/`[SEC]` 那些就是 LK 的，不是内核的。
**要读内核的东西，读 `high` 窗口**（124 KB，LK 不碰）。

### 黑匣子：`/sys/kernel/debug/logstore/{low,high}`（ringfix33）

那个 DRAM 环**能扛住看门狗硬复位**（bootloader 自己就把日志放那儿），所以**上次启动的最后遗言
只存在那里**。LK 只可靠转发一个固定前缀进 expdb，而这条转发在本机很不稳（见第五节「expdb 的坑」）——
**前几轮之所以分不清"panic 还是复位"，就是因为缺这个通道。**

驱动现在把两个窗口导出成 debugfs 文件（`low` 62 KiB = LK 可靠转发的那段，`high` 124 KiB），
读接口把 **NUL 折叠成单个换行**（环是零填充的），所以 initramfs shell 里 `grep` 直接可用：

```
grep chopinport /sys/kernel/debug/logstore/low   # 同时看到上一次+这一次的标签 → 证明环扛过了复位
grep wdt        /sys/kernel/debug/logstore/low   # 复位原因
grep panic      /sys/kernel/debug/logstore/low
```

⚠️ 顺带确认一条**死路**：**`/dev/mem` 在 arm64 上读不了 MMIO** —— arm64 没有
`devmem_is_allowed()`，走通用实现（返回 0），所以 `read_mem()` 必然 `-EFAULT`。
之前那三个 `128: (read failed: Bad address)` 就是这么来的。**"从用户空间 dump 寄存器"这条思路
以后不要再试**（要读寄存器就加驱动导出，就像这次的黑匣子）。


## 三、仓库与工作纪律

### 两个 git 仓库（各管一半，别混）

| 仓库 | 管什么 | 工作方式 |
| --- | --- | --- |
| `git/chopin-linux-7.2.8/` | **内核移植**（38 个文件；基线 = 干净 7.2.8） | 代码改在 `linux-7.2.8/`，再**复制**进仓库；收工用 `port/tools/audit-repo.py` 验（必须报 0） |
| `port/`（自带 `.git`，分支 `master`） | **文档、工具、诊断 initramfs、探针源码、审计证据** | **就地版本控制**：没有"复制进仓库"这一步，所以不可能忘 —— `git -C port status --short` 就是完整审计 |

`port/` 用**简单 ignore 清单**（只忽略 `out/`、`out-repo/`、`dist/` 里的镜像、`fbtest` 的二进制、
`__pycache__`），**刻意不用白名单** —— 白名单正是内核侧丢掉 23 个文件的方式
（被 ignore 的文件不会出现在 `git status` 的未跟踪列表里，于是一直没人发现）。
忽略什么、为什么，都写在 `port/.gitignore` 的注释里。

**收工前两个都查**：

```bash
port/tools/audit-repo.py        # 内核侧：0=干净 1=仓库有旧版本 2=仓库缺文件（危险）
git -C port status --short      # port 侧：应为空
```

内核仓库的历史（真 diff，干净基线作第一个提交）：
```
3f40e04  arm64: chopin: wire up the UFS controller resets        ← 显示之后
d834b63  arm64: chopin: make the log_store ring a real black box, and probe the display   ← ⭐ 当前基线（=ringfix37）
aace1e1  arm64: chopin: fix the three real boot errors (initrd range, FDT CRC, PMIC children)
463d815  arm64: chopin: bind the toprgu watchdog, and add the ring black box
ad2706b  arm64: chopin: bring up USB in device mode (the tphy session signals)
043b20f  arm64: chopin: fix the SPI controller clocks
b77aab5  arm64: chopin: bump the boot.img version tag to ringfix25
1dd13d2  arm64: chopin: make the LK framebuffer usable (simplefb)
ae06834  misc: actually build the logstore console driver            ← 补漏
04518b3  pinctrl: mediatek: fix MT6893 register base names and EINT window  ← 补漏
360d6d0  arm64: chopin: boot-stage tags and framebuffer stripes in head.S   ← 补漏
1ceab37  arm64: dts: mediatek: add the port's own MT6893 SoC tree    ← 补漏
56d35d1  clk: mediatek: add MT6893 clock and reset drivers           ← 补漏
85fd88f  soc: mediatek: PMIC wrapper, SPMI and UFS PHY support for MT6893
b446d59  misc: mtk-logstore-console - readable kernel log out of an MTK boot chain
c8824dc  arm64: chopin: embed our board DTB in the image and swap it in setup_arch
97a6033  arm64: dts: mediatek: chopin board tree built on the live Android device tree
2ca60ae  Linux 7.2.8 (clean upstream baseline)
```

- **移植全部改动 = 38 个文件**（15 改 + 23 新增），权威清单见 `port/MANIFEST.txt`。
  （UFS reset 那笔只改了 `chopin-board.dts`，没有新增文件，所以仍是 38。）
- 标「补漏」的 5 个提交是第二次会话补的：那 23 个文件当时只存在于工作树、
  从未进过仓库（原因与后果见下方「仓库完整性」一节）。**别再把「15 个文件」当成全部。**
- ⚠️ `2ca60ae` 自称为"干净上游基线"，但**有一个文件不是上游原版**：`.gitignore` 被换成了
  一份 17 行的精简版（上游是 196 行）。无实际危害，但做逐字节比对时别被它绊倒。
- 工作区干净。**每次收工前跑 `port/tools/audit-repo.py`，它必须报 0。**

### 目录分工
| 路径 | 用途 |
| --- | --- |
| `linux-7.2.8/` | **构建树**（改这里、编译这里） |
| `git/chopin-linux-7.2.8/` | git 仓库（干净基线 + 我们的提交） |
| `work/upstream-base/linux-7.2.8/` | 用户提供的**干净 7.2.8**（基线来源，勿动） |
| `build/chopin/` | 编译输出（O= 目录） |
| `port/` | 工具、文档、镜像 |
| `work/` | 取证材料（expdb dump、参考仓库、恢复材料） |

### 三棵内核树：哪棵是工作源码（重要，先读这段）

| 路径 | 体积 | 角色 | 判断依据 |
| --- | --- | --- | --- |
| **`linux-7.2.8/`** | 1.8 G | **工作源码 + 构建树**。改代码改这里，编译也在这里 | 改过的文件以 `port/MANIFEST.txt` 为准 |
| `git/chopin-linux-7.2.8/` | 3.5 G | **git 仓库**（干净基线 + 10 个提交）。看历史、做 diff、回退用这里 | 已补齐，见下 |
| `work/upstream-base/linux-7.2.8/` | 1.8 G | **干净 7.2.8 基线**，`2ca60ae` 那个提交的来源 | 无 chopin 文件；`setup.c` = 11569 B（原始大小） |

### ✅ 仓库完整性（2026-10-01 第二次会话审计并补齐）

**移植实际动 38 个文件，不是 15 个。** 之前仓库只提交了 15 个，另外 23 个
（10 个 `clk-mt6893-*.c`、clk 的 `Kconfig`/`Makefile`、3 个 dt-bindings 头、`mt6893.dtsi`、
2 个 clock binding、`head.S`、`drivers/misc/Makefile`、pinctrl 修正…）**只存在于工作树**。

**为什么会漏，两个盲区（都已堵掉）**：

1. `linux-7.2.8/.gitignore` 被换成了"白名单"，只放行那 15 条路径 → 新文件被 ignore，
   `git status` 的未跟踪列表里**根本不显示**（第一遍用 `git status` 审计时 16 个文件一个都没露头）。
2. 旧的校验循环只 `cmp` 它自己列的 15 个名字 —— 自证循环，必然报 `same=15 / diff=0`。

**漏掉时的后果**（这就是为什么它不是"文档问题"）：仓库拿不出能跑的内核 ——
`chopin-board.dts` 要 `#include <dt-bindings/clock/mediatek,mt6893-clk.h>`，该头不在仓库 → **DTB 都编译不出来**；
`drivers/misc/Makefile` 少一行 → logstore 驱动**静默不编译**（唯一日志通道没了）；
没有时钟 provider → **UFS 回到坏的**。

**已补齐**（5 个新提交，`2ca60ae..HEAD`）：`56d35d1` clk + `1ceab37` mt6893.dtsi +
`360d6d0` head.S + `04518b3` pinctrl + `ae06834` misc/Makefile。
`head.S` 那个提交是**补提交**，内容里不含屏幕相关注释（那部分随显示改动单独提交）。
审计证据留在 `work/audit/`，26 个文件的原始清单在 `work/audit/OUT-OF-SYNC.txt`，
38 个移植文件的全量清单在 `work/audit/PORT-FILES.txt`。

**校验方法：不要再逐名对比，用这个工具**（全树对比派生文件集，再看仓库有没有）：

```bash
port/tools/audit-repo.py            # 0=干净 1=仓库有旧版本 2=仓库缺文件（危险）
port/tools/audit-repo.py --brief
port/tools/audit-repo.py --update-manifest   # 重新生成 port/MANIFEST.txt
```

它**不读 .gitignore**（这正是上次的盲区），代价是 9.5 万文件的全树 diff，一两分钟。

`port/MANIFEST.txt` 是**生成物**（`--update-manifest` 重建），列出 38 个文件。
它是**清单，不是 .gitignore** —— 别把它当成忽略规则，那正是上次出事的方式。

**一句话操作约定**：
- 改代码 → 改 `linux-7.2.8/`，然后 `make O=../build/chopin ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j16 Image dtbs`
- 看历史 / 回退 / 出补丁 → 在 `git/chopin-linux-7.2.8/` 里做（`git log`、`git diff 2ca60ae HEAD`）
- **永远不要动** `work/upstream-base/linux-7.2.8/`（只在重建 git 历史时才需要它）
- **收工前跑 `port/tools/audit-repo.py`**，它报 2 就说明有源码快丢了
- 往仓库补文件**只复制**（`cp`），**不移动、不删除**；新文件要 `git add -f <path>`
  （`.gitignore` 现在与仓库一致，一般不需要 `-f`，但留着这句没坏处）

### ⚠️ 版本标签的坑（曾经把我误导过一次）

内核 cmdline 来自 `CONFIG_CMDLINE`（因为 `CONFIG_CMDLINE_FORCE=y`），所以日志里的版本标记是
**编译进内核的**，不是 boot.img 里的。**改了代码却忘了递增标签，内核就会自称上一版**：

实测：`ringfix23` 的内核在 dump 里显示成 `chopinport=ringfix22`（当时我因此误判"用户还没刷"）。

所以每次构建要**标签与代码一起改**，并且**必须重新生成配置**，否则 `.config` 里还是旧值：

```bash
# 1) 改标签（两处都要）：
#    linux-7.2.8/arch/arm64/configs/chopin_defconfig
#    port/tools/build-images.sh
sed -i 's/chopinport=ringfix[0-9]*/chopinport=ringfixNN/' \
    linux-7.2.8/arch/arm64/configs/chopin_defconfig port/tools/build-images.sh
# 2) 重新生成 .config（关键，否则改动不生效）
cd linux-7.2.8 && make O=../build/chopin ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- chopin_defconfig
# 3) 编译
make O=../build/chopin ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j16 Image dtbs
```

**校验标签确实变了**（不用上机也能验）：
```bash
strings build/chopin/arch/arm64/boot/Image | grep -m1 "chopinport=ringfix"
```

### 纪律（前一次事故的教训，务必遵守）
1. **备份只放工作区** —— `/tmp` 不跨命令持久
2. **任何路径一律绝对** —— 相对路径配 `cd` 会造成误删/误改（曾因此删掉 8 个文件）
3. **删任何东西前先确认内容已另存**；建仓库用**新目录**，不原地删 `.git`
4. 改完 `SUMMARY.md` 等文档后**断言校验**（曾出现静默写入失败）

---

## 四、源码丢失与恢复（背景交代，影响可追溯性）

**事故**：用干净基线重建 git 历史时，我把 15 个改动文件的备份放在 `/tmp`（不持久），
随后在同一条命令里用相对路径做还原（`cd` 后路径解析错），走了 `rm -f` 分支删掉 8 个被修改的文件；
同命令还 `rm -rf .git` 断了退路。**源码因此丢失。**

**恢复依据**（都在盘上）：
- `build/chopin/drivers/misc/mtk-logstore-console.o` —— **带完整 DWARF**（函数清单、局部变量、行号表）+ 全部格式字符串
- `build/chopin/arch/arm64/kernel/chopin-board.dtb`（277450 B）—— 重建 `chopin-board.dts` 后可**逐字节比对**
- `build/chopin/.config` —— `savedefconfig` 还原 `chopin_defconfig`
- `work/Android-chopin.dts` —— `androidbase.dts` 的源
- `port/SUMMARY.md` + 各提交 message —— 逐条记录了每一处改动及其原因

**恢复结果**：
- ✅ `chopin-board.dts` —— 编译出的 DTB **SHA256 与丢失前产物完全一致**（`c2ee71e2…`），属精确恢复
- ✅ 其余 14 个文件按文档重打；`setup.c` 的 FDT 替换块 + 9 个面包屑
- ⚠️ `mtk-logstore-console.c` 是**依据 DWARF/反汇编/文档重建**的等价实现（772 行，函数 16/16、
  静态布局逐字节、30/30 行号锚点对齐；语句级行号有 ±1–3 行偏差）。**行为已上机验证等价**。
- 提取材料：`work/recover/logstore-driver.md`、`work/recover/scratch/`（旧产物的全 DWARF/反汇编/字符串）、
  `work/recover/PLAN.md`（事故记录与恢复顺序）
- 注意：参考用的旧 `.o` 已被重新编译覆盖（字节层面不可再取回），但全部证据已事先提取

**恢复过程中我自己的两个疏漏（已修）**：
1. `setup.c` 只重打了 FDT 替换块，**漏了 9 个面包屑调用** → 同时造成「环里没有 `[BC ]`」和「顶部少绿线」
2. 补回面包屑后暴露 `fb_band_early` 每次映射 17 页、而 `early_ioremap` 的 `NR_FIX_BTMAPS` 是
   **全局 65 页预算** → 第二个面包屑就耗尽 → 引导停在 `paging_init()`。
   改为 **4 页分块映射**后，屏幕 13 条绿线稳定 ✓

---

## 五、显示（framebuffer）：✅ 已跑通（ringfix24，用户确认屏幕工作）

### 原来的失败（从 expdb 挖出的准确根因）

```
simple-framebuffer 78e0c000.framebuffer: simplefb: cannot reserve video memory at [mem 0x78e0c000-0x7b5ebfff]
[    0.654234]  simplefb_probe+0x370/0xb3c
simple-framebuffer 78e0c000.framebuffer: probe with driver simple-framebuffer failed with error -12
init: /proc/fb : (empty)
```

`request_mem_region()` 失败**只是警告**（`simplefb.c` 会退回继续），致命的是紧接着的
`ioremap_wc()` 返回 NULL → `-ENOMEM`。原因在 `arch/arm64/mm/ioremap.c`：

```c
if (WARN_ONCE(pfn_is_map_memory(__phys_to_pfn(phys_addr)),
              "ioremap attempted on RAM pfn\n"))
        return NULL;
```

`mblock-19-framebuffer` 在基树里是 `map non-reusable`（**没有 `no-map`**），
所以它仍是 System RAM、在线性映射里 → arm64 拒绝 ioremap → simplefb 拿不到那块内存。
参照：跑通的 MT6895-xaga 对它的 `mblock-12-framebuffer` 正是这么做的。

### 实际改的四处（都在工作树里，**尚未进仓库**）

1. `chopin-board.dts`：给 `&{/reserved-memory/mblock-19-framebuffer}` 覆盖 `no-map;`
   → `memblock_is_map_memory()` 变 false → `ioremap_wc()` 放行
2. `chopin-board.dts`：simplefb 的 `reg` 从越界的 `0x27e0000` 收敛成 **`0x9f6000`**
   （一帧 = stride 4352 × 2400）。必须落在保留区内，否则 ioremap 又去 alias System RAM
3. `mtk-logstore-console.c`：条纹绘制从 `__va()` 改成 `ioremap_wc()`
4. `head.S`：只改注释（K00–K03 的条纹不受影响，见下）

### ⚠️ 第 3 项那个不显然的坑（下次改这里先读）

`ioremap()` 依赖 `vmalloc_init()`，而它在 `mm_core_init()` 里、**比 `setup_arch()` 还晚**。
所以 `setup_arch()` 里 `paging_init()` 之后的那几个面包屑**两头都不通**：
`__va()` 不行（区已 no-map，线性映射不再覆盖它）而 `ioremap()` 还不行（vmalloc 没起来）。
现在的做法是**待画队列**：`fb_band_late()` 先试着 ioremap，不行就把 band 记进
`fb_pending[]`，到 logstore 的 early_initcall（此时 vmalloc 已可用、且在 simplefb 之前）
由 `fb_flush_pending()` 统一补画 —— 所以"条纹数 = 走到第几个阶段"的语义完整保留。

`head.S` 则完全不受影响：`K00..K03` 全部在 `bl __enable_mmu` **之前**，
MMU 关闭时的物理直写，no-map 与否无关。

### 屏幕现在是什么样

`CONFIG_LOGO=y` + `CONFIG_FONT_8x16=y` + `CONFIG_FRAMEBUFFER_CONSOLE=y` 都开着，
所以 simplefb 一旦 ioremap 成功，**fbcon 会接管整屏并画内核日志**（比条纹更有用的证据）。
不过 fbcon 也会把条纹覆盖掉，所以条纹只在 simplefb probe 之前有意义。

### ringfix24 上机证据（2026-10-01 17:37，照片已存档）

`work/device/IMG_20261001_173726-r24-landscape.jpg`、`IMG_20261001_173708-r24-portrait.jpg`
（sha256 已核对）。屏幕上的关键行：

```
chopinport=ringfix24                       ← 标签终于对上了（不再出现 ringfix23 自称 22）
MTK-UFSPHY: mmio ok / clocks ok (unipro, mp) / phy_create ok / provider registered, probe done
mt6397-rtc mt6359-rtc: registered as rtc0
ufshcd-mtk 11270000.ufshci: caps: 0x108    ← BROKEN_VCC(1<<3) + DISABLE_MCQ(1<<8)，两条 DT 属性都生效
ufshcd-mtk 11270000.ufshci: clk "ufs" present / clk "ufs_sel" present
scsi host0: ufshcd
init: step 5: framebuffer state
init: simplefb bound    :
init: no shell in initramfs, staying alive
```

**屏幕本身就是最硬的证据** —— 能看见这些字符，就说明 simplefb 拿到了那块内存、fbcon 在画、
显示控制器在扫出。之前 `/proc/fb` 为空、没有 `Console: switching to colour frame buffer` 的状态结束了。

⚠️ 照片上**看起来**像 `caps: 0x100`、`aphy_rst` 是**误读**：驱动里只有 `mphy_rst`
（`ufs-mediatek.c:248`），而 `broken-vcc` 属性确实在编译进内核的 DTB 里，所以 caps 必为 `0x108`；
r23b 的 dump 文本也印证是 `0x108`。

### ⚠️ expdb 的坑：内核日志**不是每次都能取回**

ringfix24 这次取回的 expdb（`work/expdb/live-r24.img`，取两次逐字节相同）里
**没有内核的大部分 printk** —— 只有 18 处 `[BC 0]` 面包屑、6 处 `no shell in initramfs`、
9 处 `chopinport=ringfix24`。`Linux version` / `CHOPIN-DTB` / `scsi host0` / `simplefb` **一条都没有**。

**这不是这次改动引入的**：翻遍 `work/expdb/`，`live-r21`、`live-r21b`、`live-r22`、`live-r23`
（都早于本次改动）同样是"只有 BC 面包屑、没有内核日志"；只有 `live-r20.img`（日志落在 pl/lk 区）
和 `live-r23b.img`（日志落在主区，主区非零率 7.62%）这两次抓到了完整日志。
即 `expdb` 的内核日志通道**一直是间歇性的**，约 2/7 命中。

好在那条「落在 pl/lk 区」的通道（head.S 往 LK 一定转发的低窗口里写）**每次都通** ——
`[BC 0]` 与 `chopinport=` 从未缺席，所以"走到了哪一步 / 跑的是哪一版"永远可判。
**现在屏幕也能看了，排障优先看面板**，expdb 用来拿完整时间线。

如果要把 expdb 通道做可靠，下一步该查 `SYS_RAMCONSOLE_RAW` 那段的写入时机与长度
（LK 日志里可见 `kedump add: SYS_RAMCONSOLE_RAW[0] 800/800@2000`，只有 0x800 字节）。

若下次又不亮，expdb 里有三条独立判据（initramfs 已经会 dump）：

```
init: /proc/fb       : 0 simple
init: /proc/consoles : tty0  -WU
init: simplefb bound : 78e0c000.framebuffer
```

### 离线可验的点（不用上机）

- 编译产物 `chopin-board.dtb` 里 `mblock-19-framebuffer` 必须有 `no-map`
- simplefb 的 `reg` 末地址 `0x79802000` 必须 < `0x7a200000`
- 所有 reserved-memory 子节点里**不能有别的节点**与 `0x77a20000..0x7a200000` 重叠
  —— 否则 `early_init_dt_reserve_memory()` 会因 `-EBUSY` **静默放弃 no-map**，显示又坏
- 屏幕照片证据：用户提供过 IMG_20261001_141617.jpg（绿带 + 多条黄条）
- 候选地址：`FB_CAND0 = 0x78e0c000`（绿 `0xff00ff00`/`0xff00a000` 交替）、
  `FB_CAND1 = 0x77a20000`（红 `0xffff0000`/`0xffa00000`）；黄条 `fb_note()` 用 `0xffffff00`，
  `FB_NOTE_BASE 20 + slot*2`

---

## 十、充电与电池状态：MT6360（**ringfix47/50 上机验证：已充电 + 电池状态可见** ✓✓）

目标：让这台机器在 mainline 下**真的充电**、并看得见电池状态。以下每条都注明依据来源。

### 已确证的硬件事实（读厂商源码/DT 得出）

| 事实 | 出处 |
| --- | --- |
| MT6360 的 PMU（charger/TCPC/ADC 都在这一个地址）= **I2C 0x34** | 主线 `mt6360-core.c` 的 `MT6360_PMU_SLAVEID` |
| 充电状态寄存器 = **0x34A**，bit[7:6]：0 没充 / 1 充电中 / 2 充满 | 主线 `mt6360_charger.c` |
| 芯片 ID 寄存器 0x00 的 bit[7:4] 应为 **0x5**（另见 0x300 DEV_INFO） | 厂商驱动自检逻辑 |
| 电量计 = **TI BQ27Z561 @ i2c0 0x55**（不是 PMIC 自带 gauge） | 厂商 `chopin-mt6891.dtsi`：`use-ti-gauge` + `bq27z561@55` |
| 电池：设计容量 **5000 mAh**、充电上限 **4.46 V** | 厂商 `bq,charge-full-design` / `battery_cv` |
| **MT6360 在 i2c5（0x11F00000）**，四个地址 0x34(PMU)/0x4E(TCPC)/0x1A(PMIC)/0x64(LDO) | base 树（LK 的树，它跟这颗芯片通过信）里 `i2c5@11F00000: subpmic@34, usb_type_c@4e, subpmic_pmic@1a, subpmic_ldo@64` —— **与主线 `mt6360-core.h` 的四个 SLAVEID 完全一致** |
| 电量计 TI BQ27Z561 在 **i2c0（0x11015000）@0x55** | base 树 `i2c0@11015000: bq2597x-slave@66, bq27z561@55, xmusb350-charger@35` |
| MT6360 PMU 中断 = GPIO 24，TCPC 中断 = GPIO 25 | base 树自己的 `mt6360_pmu_dts`（`intr_gpio_num = 24`）与 PD 节点（25）；厂商 `k6891v1_64.dts` 同值 |

### ⚠️ 教训：**设备挂在哪条总线上，以 base 树（LK 的树）为准**

排查充电时我按"厂商内核 dtsi 的 `&i2cN` label"去找 MT6360，把节点先后放到了 i2c6、i2c0 —— 两次都错 ✗。
真正权威的是 **我们自己的 base 树**（`mt6893-xiaomi-chopin-androidbase.dts`，就是 LK 用来跟这些芯片通信的那棵）：

```
i2c5@11F00000: subpmic@34, usb_type_c@4e, subpmic_pmic@1a, subpmic_ldo@64
i2c0@11015000: bq2597x-slave@66, bq27z561@55, xmusb350-charger@35
```

教训有两条：(1) 厂商 *内核* 的 label 与地址映射是他们各自树里定义的，不能跨树照搬 ✗；
(2) **同一个地址不能再声明第二个节点**（i2c6 上的 0x34 已被 `speaker_amp@34` 占用，
硬加自己的节点 = `-EBUSY` ✗）—— 正确做法是**就地改那个已有节点的 compatible**（如 `subpmic@34`）。

**另一条**：`i2cscan` 用"裸读"探测，**部分芯片（如 TI 电量计）会拒绝裸读** ✗ ——
所以"扫不到"不等于"不在"；`i2creg <bus> <addr> <reg>`（带寄存器地址的读）才是可靠判据。

### 阻塞链（本轮最重要的发现）

```
MT6360 MFD 强制要中断      mt6360-core.c: devm_regmap_add_irq_chip(..., client->irq, ...)
   ↓ 需要
pinctrl / EINT 能用        MFD 的 #interrupt-cells = <2>，IRQ 走 &{/pinctrl}
   ↓ 但是
pinctrl 从来没绑定过 ✗      base 树写 "mediatek,mt6885-pinctrl" + reg_bases，
                           主线驱动只认 "mediatek,mt6893-pinctrl" + reg/reg-names
```

port 自己的提交 `04518b3` 早就写明这件事（并修好了驱动侧：base name 从 10 个改回 8 个，
否则 `nbase = count_reg_names - nbase_names` 为负 → `mtk_pinctrl_probe()` 直接失败），
**但因为 chopin 走的是 LK 派生的树，一直没被触发** ✓ —— 也就是说**这台机器上所有 GPIO 类外设
（触摸、指纹、USB 开关、这里的中断）此前都没有立足点**。

### ✅ 上机结果（ringfix47，2026-10-02）

```bash
chopin# cat /sys/class/power_supply/mt6360-chg.2.auto/uevent
POWER_SUPPLY_STATUS=Charging          <-- 充电中
POWER_SUPPLY_ONLINE=1                 <-- VBUS 在线
POWER_SUPPLY_CHARGE_TYPE=Fast
POWER_SUPPLY_CONSTANT_CHARGE_CURRENT=500000   (500 mA，芯片默认；可写)
POWER_SUPPLY_CONSTANT_CHARGE_CURRENT_MAX=5000000
POWER_SUPPLY_CONSTANT_CHARGE_VOLTAGE=4350000
POWER_SUPPLY_INPUT_CURRENT_LIMIT=500000
```

链条：`11f00000.i2c5` probe ✓ → `mt6360_irqs ... buck1_oc_evt`（MFD 中断 chip 注册 ✓ = pinctrl/EINT 通了 ✓）
→ `mt6360-chg.2.auto` / `mt6360-tcpc.5.auto` / `mt6360-adc.1.auto` 三个子设备 probe ✓
→ `/sys/class/power_supply/mt6360-chg.2.auto` 报 **Charging** ✓。

**注意两个容易误判的点**（都踩过）：
1. 子设备名带 `PLATFORM_DEVID_AUTO` 的 `.N.auto` 后缀（`mt6360-chg.2.auto`），
   按 `mt6360-charger` 去 `cat` 会得到 "(missing)"，**看起来像驱动没绑** ✗ —— 用 `power` 命令或 `ls /sys/class/power_supply` 发现真名 ✓。
2. **`/dev/i2c` 会拒绝已被内核驱动占用的地址** ✗ —— 所以 MT6360 绑定之后，
   `i2cscan`/`mt6360`/`icreg` 对 0x34 必然失败 ✓（这**不是**硬件问题）。
   反向也成立：`i2c-0` 上还能扫到的 0x35/0x66 正是**没有主线驱动**的两颗芯片 ✓。

### 充电电流只有 500 mA —— 可写属性可以调

`input_current_limit` / `constant_charge_current` 都是**可写**的 sysfs 属性：
芯片硬件默认 500 mA，而厂商在这块板子上要 2 A（`ac_charger_current = 2000000`）✓。
`write /sys/class/power_supply/mt6360-chg.2.auto/input_current_limit 2000000` 即可（本 shell 的 `write` 命令 ✓）。

### 电量计（i2c0 0x55）现状：部分可读

`POWER_SUPPLY_MANUFACTURER=Texas Instruments` ✓、`CURRENT_NOW=-1000` µA ✓、`POWER_AVG=-10000` µA ✓ 是真值；
但 `VOLTAGE_NOW`/`CAPACITY`/`CYCLE_COUNT`/`TEMP` 都是 `65535`(0xFFFF) ✗ → **同一颗芯片部分寄存器读得到、部分读不到**，
高度怀疑主线 bq27xxx 对 BQ27Z561 用的命令集与厂商驱动不同（待下一轮对照厂商 `bq27z561` 驱动核实）。
另外 `charge-full-design` 没被采用（读出来是 0xFFFF ✓ 而我们写的 `energy-full-design` 生效了 ✓）→ 属性名待核。

### ringfix45 做了什么（待上机）

1. **pinctrl 改为主线绑定**：9 个 reg 窗口 + `reg-names` + `#interrupt-cells = <2>`，
   并 `/delete-property/ reg_bases, reg_base_eint`（binding 是 `additionalProperties: false`）。
   值取自 port 早期那份主线绑定树 `mt6893-xiaomi-chopin.dts` ✓。
2. **i2c0 / i2c6 修正**：compatible 改 `mediatek,mt8192-i2c`，clocks 换主线 ID
   （`CLK_INFRA_AO_I2C0/I2C6` + `CLK_INFRA_AO_APDMA`）—— 与当年修 8 个 SPI 控制器同一套写法。
3. **MT6360 节点**：`pmic@34`（`interrupts = <24 IRQ_TYPE_EDGE_FALLING>`）+ `charger`
   （`richtek,vinovp-microvolt = <14500000>`，厂商的 `max_charger_voltage`）+ `adc`
   + `tcpc`（`<25 IRQ_TYPE_LEVEL_LOW>` + `usb-c-connector`）。形状按主线 binding 逐条校验过：
   MFD required 五项全满足；**`#interrupt-cells` 必须是 1**（一开始我写错成 2 ✗，
   binding 原话："The first cell is the IRQ number"）。charger/adc 子节点不是可有可无：
   缺了 MFD 会打 `Failed to locate of_node`（和之前 MT6359 按键那个警告同源）。
4. **电量计**：`bq27z561@55` + `simple-battery`（5000 mAh / 4.46 V / 3.2 V）。
5. **配置**：`CHARGER_MT6360` / `MEDIATEK_MT6360_ADC` / `TYPEC_MT6360` / `TYPEC_TCPM` / `TYPEC_TCPCI`
   全部 **`=y`**（initramfs 不加载模块；TCPM/TCPCI 不改成 `y` 的话 MT6360 会被迫降级成 `m` ✗ ——
   这个坑踩了两次，和当初按键驱动一样）。
6. **initramfs 直读工具**：`i2cscan` / `i2creg` / `mt6360` —— **不依赖任何驱动**就能读芯片 ID 与
   0x34A 充电状态。这是"到底在不在充电"的地面真值，比任何 dmesg 都硬。

### ✅✅ 电池状态也验证了（ringfix50，2026-10-02）

MT6360 自己的 PMU ADC 给出**真实数值**：

```
in_voltage4_label = vbat     in_voltage4_raw = 3067   in_voltage4_scale = 1250
in_current6_raw   = 119      in_current6_scale = 2500
⇒ VBAT = 3067 × 1250 = 3,833,750 uV = 3.83 V      （合理的锂电池电压）
⇒ IBAT =  119 × 2500 =   297,500 uA = +297.5 mA   （正 = 流入电池，与 Charging 一致）
```

一次实测里 IBAT 只有 ~300 mA，因为那次启动的输入限流还是芯片默认 500 mA ✓；
`ringfix51` 起由 initramfs 在启动时写 2 A（厂商配置值 ✓），所以之后 `battery` 里 IBAT 应明显变大 ✓。

### ✅ 热重启后"只剩条纹、像变砖"：机制与修复（ringfix55–56，**已上机验证**）

**归因纠正**：这个现象**在 r37 上就能复现** ✓ ⇒ **不是 r53 引入的** ✗，是长期存在的问题 ✓。

**照片证据 + 代码对照**：
- 代码：`FB_COLOUR_GREEN` 写 **CAND0**（0x78e0c000 = **fbcon 画的那块** ✓）、`FB_COLOUR_RED` 写 **CAND1**（0x77a20000 ✓）；
  而 `fb_note()` 用**黄色写在两块上** ✓、`fb_progress()` 的进度条纹按候选分色 ✓。
- 用户照片：卡住的那次屏幕上**有黄色 notes（两块都写 ⇒ 一定看得见 ✓）**、**有红色进度条纹（⇒ 面板在扫 CAND1 ✓）**，
  而 **fbcon 的文字完全没有** ✗（它只往 CAND0 写 ✗）。
- 正常的那次（拔线前）：屏幕上是**完整内核日志** ✓ ⇒ 那次面板扫的是 CAND0 ✓。

⇒ **机制**：热重启（Linux → LK）后 LK 把活动缓冲留在 **另一块** ✓，fbcon 的文字写进了看不见的地方 ✗。
叠加上"USB 串口也起不来"✗（另一个问题 ✓）⇒ **两条观测通道同时归零** ⇒ 看起来像变砖 ✓，只能冷启动到 fastboot 恢复 ✓。

**ringfix55 的修法**（安全、不动 USB ✓）：我们的驱动本来就有整个保留区的 **NonCacheable 映射** ✓
（`ioremap_wc(FB_REGION_BASE, …)` ✓，两块候选都在里面 ✓），于是**每 250 ms 把 CAND0 的内容 `memcpy` 到 CAND1** ✓
（源用缓存别名 `__va` ✓ 快；目的走 WC ✓ 只需 `wmb()` ✓ 无需 cache 维护 ✓）：
**面板扫哪一块，文字都在** ✓✓（扫 CAND0 时这份拷贝落在没人看的地方，代价只是一次 memcpy ✓）。
可用 `/sys/kernel/debug/logstore/fb_mirror` 开关（0600 ✓）随时关掉对比 ✓。
**假设的边界**：假定两块候选的几何相同（1080×2400 ✓ linelength 4352 ✓ = 0x9f6000 字节 ✓）；
若 CAND1 的 stride 不同，镜像会歪斜 ✗（一眼可见 ✓）。

**验证结果**（用户实测）✓：热重启后第一屏顶部虽红 ✓，但**能进入屏幕控制台、可以连串口** ✓、
**不再卡第一屏** ✓ ⇒ 镜像确实把 fbcon 的内容带到了面板正在扫的那块 ✓✓。

**它同时解决了 host 模式的前置** ✓：host 模式必然让 USB 串口消失 ✗，
现在**屏幕**是可靠的第二条观测通道 ✓（失败也不用重刷、能直接看到原因 ✓）。

### ✅ 拔插后 shell 消失：真因与修复（ringfix57–59，**已上机验证**）

**现象**：拔掉 USB 后串口挂掉 ✗；插回后 PC 侧**确实重新枚举**（PC dmesg：每次都有
`new high-speed USB device 0525:a4a7` + `cdc_acm ... ttyACM0` ✓），但 minicom 连上**没有提示符** ✗，
用户以为系统崩了 ✗。

**排查：两条被证伪的假设 + 一条正确的**
1. ✗ "设备没重新枚举" —— PC 侧 dmesg 证明每次插回都重建 ttyACM0 ✓。
   （用户先前 `ls /dev/tty*` 看不到，是因为**当时线正拔着** ✓ —— 教训：报告现象时要记下当时的物理状态 ✓。）
2. ✗ "需要重绑 mtu3" —— `g_serial` 的 `/dev/ttyGS0` 在**绑定时注册一次、插拔不重建** ✓，
   所以普通拔插后 shell 的 fd 依然有效 ✓ ⇒ **重绑反而是破坏性的** ✗（ringfix57 一度如此 ✗，已改 ✗）。
3. ✓ **真因**：`shell_loop()` 里读失败就 `return` 退出 ✗ ⇒ 再没人读 `/dev/ttyGS0` ✓
   ⇒ minicom 连得上却永远没提示符 ✓。**内核全程正常** ✓（屏幕时间戳继续走到 73.9 s ✓，
   真崩会看到看门狗复位+时间戳归零 ✓）。

**修复（ringfix59）**：读失败（`EIO` 或真 `EOF`）**不再退出** ✓，改为
"等待 `/dev/ttyGS0` → 重新 `open` + `dup2` 接管 → 打印 `console reconnected` → 继续读" ✓。
同时把 `r == 0` 也当"控制台丢失" ✓（`O_NONBLOCK` 下正常无数据是 `EAGAIN` ✗ 不是 0 ✓）。

**保留 ringfix58 的"有证据才动手"** ✓：看护在插回（charger `online` 0→1 ✓）时先等 2 秒，
读 `/sys/class/udc/11201000.usb/state` ✓：
`configured` ⇒ **什么都不做**（只在 kmsg 记一行 ✓）；否则才重绑 + 重启 init ✓。

**结论**：拔插重连现在**完全可用** ✓ —— 设备重新枚举 ✓ + shell 自愈 ✓。

### ✅ USB host 模式：可用（ringfix60，**已上机验证**）

**做法**（都是主线现成机制 ✓）：
- charger 下加 OTG 稳压器 `usb-otg-vbus-regulator` ✓（4.425–5.825 V ✓，驱动 `mt6360_otg_rdesc.of_match` 认这个节点名 ✓）
- mtu3：`dr_mode = "otg"` ✓ + `usb-role-switch` ✓ + **`role-switch-default-mode = "peripheral"`** ✓ +
  `vbus-supply = <&otg_vbus>` ✓
- 启动时 initramfs 把角色**再钉一次 device** ✓（双保险 ✓）；需要时用 **`usbhost <秒数>`** 开一个**限时 host 窗口** ✓，
  到点自动切回 ✓（`cmd_usbhost` 还会等 `/dev/ttyGS0` 回来并重新接管 ✓）

**关键设计点**：`role-switch-default-mode = "peripheral"` 是**唯一**防止"开机直接进 host ⇒ 串口在用户态之前就没了"的东西 ✓；
驱动通过 `usb_get_role_switch_default_mode()` 读它 ✓（ringfix53 那次误判是另一回事 ✓ —— 见下面失败记录 ✓）。

**设备实测（用户屏幕日志）**：
```
mtu3 11201000.usb: gadget (high-speed) pullup D-        ← 切 host
usb 1-1: new high-speed USB device number 2 using xhci-mtk
hub 1-1:1.0: USB hub found / 4 ports detected
usb 1-1.3: new full-speed USB device number 3 using xhci-mtk
input: YICHIP Wireless Device ... hid-generic ... USB HID v2.00 Keyboard/Mouse
mtu3 ... pullup D+                                     ← 到点自动切回 device
init: console lost (cable out?); waiting for it to come back / init: console back
```
⇒ **xhci 起来 ✓、枚举到 Hub ✓、无线键鼠作为 HID 输入设备被识别 ✓✓**。

**一个容易误判的现象**：host 窗口结束时，手机对面接的是 **Hub 的上行口** ✗，
Hub 不会反向枚举设备 ✗ ⇒ PC 侧读不到 ttyACM0 ✓（手机侧 `pullup D+` ✓ 与 shell 自愈 ✓ 都正常 ✓）。

**ringfix61 修好了"切回 device 不可靠"** ✓（**已上机验证**：切 host 后再切回 device，串口能重新连上 ✓）：
切回 device 后改为**轮询等主机真的来配置我们（最多 5 秒）** ✓，等不到才**重绑 mtu3** ✓（重跑 `phy_init` ✓ +
重新拉起 gadget ✓，同时清掉 host 窗口在 PHY 里留下的状态 ✓，并关掉可能残留的 OTG 升压 ✓）。
**为什么不能用"单次读 UDC 状态"**：`configured` 可能是上一个 host（Hub）留下的**陈旧值** ✗ ⇒ 会错误地跳过重绑 ✗
（ringfix58 就是这么漏的 ✓）。同时把 `usb-otg-vbus` 稳压器状态打到 kmsg/屏幕 ✓，便于确认 VBUS 有没有关掉 ✓。

**测试安全提示**：手机当主机时会**输出 5 V** ✗，若对面是 PC 的口（也在供 5 V ✗）就是**两边对着灌** ✓，
PC 的口可能进入保护状态 ✗（换口或重启 PC 才恢复 ✓）。建议 host 测试用 **OTG 转接头 + U 盘/键鼠** ✓，
测完要恢复串口就**拔掉转接头、把手机直插 PC** ✓。

### ⚠️ 失败记录：把 mtu3 改成 OTG 会让整机失去观测通道（ringfix53，**已撤回**）

**做了什么**：`dr_mode: peripheral -> otg` ✓ + `usb-role-switch` ✓ + `role-switch-default-mode = "peripheral"` ✓
+ charger 下加 OTG VBUS regulator ✓（依据：`usb_get_role_switch_default_mode()` 确实会读那个属性 ✓）。

**结果（用户实测）**：
- 开机卡在 LK 那一屏（红线 = LK 留的是另一块缓冲 ✓），**USB 串口也起不来** ✗
- 运行中拔插 USB 后也无法重新连上串口 ✗
- **只能 fastboot 重刷** ✓

**教训（写下来别再犯）**：
1. **动唯一调试通道之前，必须先有一条能替代它的通道** ✗。
   当时屏幕本来就不可靠（红线 ✓），于是"串口没了 + 屏幕不可读"= 两条通道同时归零 → 只能重刷 ✗。
2. 代码里**读得到**某个属性 ≠ 设备上行为符合预期 ✗ —— 没上机验证过的机制，
   不要用它来给"切断观测"的设计背书 ✓。
3. **撤回方式**（可复用的做法）：board 树从提交 `ec95dff` **逐字节恢复** ✓，
   再与 r52 镜像内核里嵌入的 DTB 对比验证：**同尺寸、仅差 4 字节**（= `fdt-initrd.py` 打的 initrd-end 补丁 ✓）
   ⇒ DT 确实回到已验证状态 ✓✓。

**待查（重要）**：r53 那次启动为什么 gadget 起不来 ✗。环是共享且能扛复位的 ✓，
重刷好镜像后那段日志**很可能还在** ✓：

```bash
grep chopinport /sys/kernel/debug/logstore/high    # 找 ringfix53 横幅
grep mtu3       /sys/kernel/debug/logstore/high
grep usb_role   /sys/kernel/debug/logstore/high
grep vbus       /sys/kernel/debug/logstore/high
```

**host 模式的正确路线（修正后）**：切换角色必然让 gadget 消失 ✗ ⇒ 必须**先备好第二条观测通道** ✓：
优先用**能扛复位的日志环**做"事后复盘" ✓（做完 host 测试 → 重启 → 读环 ✓），
并且**先搞清 r53 为何失败** ✓，否则下一次还会以同样方式失去串口 ✗。

### 三条"电池状态"路线的结论（都走到头了，别再重复）

| 路线 | 结论 |
| --- | --- |
| TI **BQ27Z561**（i2c0 0x55） | ✗ 芯片**应答**（读失败会返回错误，而它返回 0xFFFF ✓）但**所有数据寄存器都是 0xFFFF** → 未初始化；厂商的 `bq27xxx_battery.c` 里 **Z561 出现 0 次** → Android 用的是 MTK 私有电池栈，主线驱动能否驱动它未知 ✗ |
| **MT6359 auxadc** 的 VBAT/IBAT | ✗ 走 **IMP（阻抗测量）** 路径：返回的已是转换后的值，驱动**又乘一遍 scale**（7×1800/2=6300）→ 实测 1719 V ✗；而且**不稳定**（连读两次电流差 2 倍 ✗） |
| **MT6360 PMU ADC** ✓ | ✓✓ 原版主线驱动、已绑定、scale 正确：**VBAT = 通道 4（1250 µV/LSB）**、**IBAT = 通道 6（2500 µA/LSB）** |

因此 `battery-adc`（generic-adc-battery）节点被**撤掉** ✗ —— 它把 IIO 值当 mV 再 ×1000，
而这两个 ADC 驱动的 scale 是 µV 语义 ✗，接哪个源都是错的 ✓。
要让 `power_supply` 节点也正确，需要（a）改 generic-adc-battery 的 ×1000，或（b）写个小的 MT6360 电池驱动 ——
目前用 initramfs 的 `battery` 命令取代（直接 raw × scale，自洽可信 ✓）。

### 两个工具 bug（排查中踩到，值得记）

1. **`PLATFORM_DEVID_AUTO` 的 `.N.auto` 后缀**：MFD 生成的子设备叫 `mt6360-chg.2.auto`，
   按 `mt6360-charger` 去读会得到 "(missing)" ✗ —— **看起来像驱动没绑**。同理 IIO 设备叫 `mt6360-adc.1.auto` ✓。
2. **sysfs 几乎全是符号链接**：shell 的 `ls`/`lsr` 原来用 `d_type == DT_DIR` 判断 ✗，
   于是 `/sys/bus/iio/devices/iio:deviceN` 这类**链接目录被当成文件读**，显示 `= (empty)` ✗。
   改用 `stat()`（跟随链接）后正常 ✓。之前几次 "(missing)/(empty)" 的困惑都源于此。

### 上机验证清单（ringfix45）

```bash
mt6360             # ★ 自动找 MT6360：应在 i2c5/0x34，芯片 ID 0x5X + 充电状态(0x34A bit7:6)
i2cscan            # 全总线 + 每条的 of_node（如 i2c5@11F00000）
dmesg mt6360       # MFD/charger/tcpc 探测结果
ls /sys/bus/platform/drivers/mt6893-pinctrl   # 有 10005000.pinctrl = 绑定成功（r46 已确认 ✓）
dmesg mt6360       # MFD/charger/tcpc probe 结果
ls /sys/class/power_supply
cat /sys/class/power_supply/mt6360-charger/status
cat /sys/class/power_supply/bq27z561-0/capacity
```

### 仍未解决 / 下一步

- **MT6359 自己的 auxadc**（`mt6359-auxadc.c` 主线自带 ✓）：电池电压/温度的第二条路径，
  但需要 PMIC 的 nvmem 校准 cell，本轮没做。
- **USB host 模式**：等充电闭环之后（VBUS 由 MT6360 的 OTG boost 提供，
  需要让 `power-role` 允许 source，并把 mtu3 从 `peripheral` 切过去）。
- **充电管理**（限流/温控/QC/PD 快充）：本轮只求"能充 + 看得见"。

## 六、下一步

**已完成的两个大件**（细节见第二节）：USB 设备模式 + 串口控制台（ringfix31/32）、
看门狗修复（ringfix33）。**从 ringfix31 起日志走 USB 串口，从 ringfix32 起有交互式 shell、
`panic=0`，从 ringfix33 起板子不再每 31 秒自重启。**

### 还缺的（优先级大致从上到下）

> **2026-10-02 起，优先级最高的一条已变成「充电」** —— 详见文末「十、充电与电池状态」。

| 项 | 现状 | 关键点 |
| --- | --- | --- |
| **USB host 模式** | 只跑通过一次（ringfix28 控制器起来了、根集线器 0 端口） | 设备模式已稳；host 需要 **5V VBUS/OTG 升压**（MT6895 参考用 MT6375 的 OTG 稳压器）。要看 `xhci-mtk` 的 `ippc` 资源与端口使能 |
| **Type-C（MT6360 TCPC/充电器）** | 无驱动 | 主线有 `tcpci_mt6360.c` + `mt6360_charger.c`，但我们的 config 里都是 `=m`。**这也会顺带解决 VBUS**（还有 `en_wdt`/`apwdtrst_en` 两个未接管的看门狗） |
| **显示** | 只有 simplefb（framebuffer 已跑通） | 真 DRM/DSI 驱动 + 面板初始化，是量级最大的一块 |
| **电池 / 充电** | 无 | 同上，走 MT6360/MT6359 |
| **Wi-Fi（CONNAC2X2 SOC3_0）** | 驱动已导入、构建通过（`build/net-build7.log`） | 见 `port/WIFI-PORT.md`；上电走 MT6893 自己的 SPM 序列，固件/NVRAM 用 initramfs 的 `fwload` |
| **BT / 音频 / 触控** | 触控已完成；BT 只带了源码（不编译）；音频全无 | 各自独立的大件 |
| **剩余良性报错** | 见第九节 | `mt6359-keys/accdet` 的 of_node、`Initramfs unpacking failed`、`/sys/firmware/fdt` CRC、`[Firmware Bug] Kernel image misaligned` 等 |
| **上游化** | 未做 | 当前补丁是能跑的形态：`mediatek,force-session-valid` 是我起的名字，要上游得配 binding 文档；`mtk-logstore-console` 是平台专用驱动 |

### 排查方式已经变了（重要）

**不要再用照片。** 顺序是：`screen /dev/ttyACM0 115200` → shell 里
`dmesg <关键字>` / `cat` / `grep` / `clk` / `irq` / `udc` / `usb` / `reboot`；
复位类问题看 `/sys/kernel/debug/logstore/low`。屏幕只在 gadget 没起来时才是主通道。

---

## 七、工具与命令

### 构建 + 打包

**推荐：从 git 仓库构建**（能证明仓库自足，而且版本串会带上提交号，dump 可追溯到具体提交）：

```bash
cd /home/kily/project/mt6893
R=/home/kily/project/mt6893/git/chopin-linux-7.2.8
B=/home/kily/project/mt6893/build/chopin-repo
make -C $R O=$B ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- chopin_defconfig
make -C $R O=$B ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j16 Image dtbs

# 打包（stockdt 变体才能刷；ramdisk 与原厂基树沿用 port/out 里的）
KERNEL=$B/arch/arm64/boot/Image \
DTB=$B/arch/arm64/boot/dts/mediatek/mt6893-xiaomi-chopin.dtb \
RAMDISK=/home/kily/project/mt6893/port/out/initramfs.cpio.gz \
STOCK_DTB=/home/kily/project/mt6893/port/out/stock-base.dtb \
  port/tools/build-images.sh /home/kily/project/mt6893/port/out-repo

cp port/out-repo/boot-stockdt.img port/dist/chopin-ringfixNN-stockdt.img
sha256sum port/dist/chopin-ringfixNN-stockdt.img
```

`O=` 必须在仓库外，构建不会污染仓库源码树（收工自检会验这一点）。
**注意 `CONFIG_LOCALVERSION_AUTO=y`**：仓库里构建出来的横幅是 `Linux version 7.2.8-g<短提交号>`，
而工作树里构建的是光秃秃的 `7.2.8` —— 所以两种构建的 Image **逐字节不会相同**
（版本串长度不同会让后面的 `.rodata` 整体挪几字节，字符串表看起来差异很大）。
判断是否同一个内核要看 **`System.map` 的符号名序列** 与 **各节大小**，不是看 Image 哈希。
仓库脏时还会多一个 `-dirty`，那正好是个提醒。

从工作树构建（改动后快速迭代用，`linux-7.2.8/` 不是 git 仓库，所以无提交号）：

```bash
cd /home/kily/project/mt6893/linux-7.2.8
make O=../build/chopin ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j16 Image dtbs
cd ..
./port/tools/build-images.sh            # 产出 port/out/boot-{stockdt,owndtb,...}.img
```

### ✅ 仓库构建已验证上机启动（2026-10-01 17:57）

`ringfix25` 这一版是**纯用仓库源码**（`HEAD = b77aab5`）构建并打包的
（`port/dist/chopin-ringfix25-repobuild-stockdt.img`,
sha256 `48c642dd793862032229e1521ada659a78fe73daad00eb20674b65984f369f1f`）。

判定它「和工作树构建是同一个内核」的两条硬证据（**别看 Image 哈希**，理由见上）：

- `System.map` 的 **223091 个符号名序列 md5 完全相同**
- 各主要节大小完全相同（`.text` 17924096、`.rodata` 14253016、`.init.text` 605836 …）

上机结果 —— **能启动，显示正常**（照片 `work/device/IMG_20261001_175647-r25-landscape.jpg`、
`IMG_20261001_175703-r25-portrait.jpg`）：

```
simple-framebuffer 78e0c000.framebuffer: format=a8r8g8b8, mode=1080x2400x32, linelength=4352
Console: switching to colour frame buffer device 128x75
simple-framebuffer 78e0c000.framebuffer: fb0: simplefb registered!   ← 之前一直缺的就是这一行
mt6397-rtc mt6359-rtc: registered as rtc0
scsi host0: ufshcd
init: step 5: framebuffer state
```

`linelength=4352` 正是我们写的 stride，`mode=1080x2400x32` 正是我们写的几何 ——
说明 `reg` / `stride` / `width` / `height` 全对上了。expdb（`work/expdb/live-r25.img`）里
`chopinport=ringfix25` 命中 9 次、**`ringfix24` 0 次**、`arm64_serror_panic` 0，版本无歧义。

⚠️ 唯一没能从 expdb 拿到的自证标记是版本横幅里的提交号：`gb77aab51b980` 在
`work/expdb/live-r25.img` 里 **0 命中** —— 因为它是最早的那条 printk，而 expdb 只稳定转发
「LK 一定转发的那段」（见下面「expdb 的坑」）。它只在**屏幕**上看得到。
所以「这是仓库构建的」这一点，屏幕照片是主证据，expdb 只证到版本号那一层。

### 仓库完整性（**每次收工前跑**）
```bash
port/tools/audit-repo.py        # 0=干净 1=仓库有旧版本 2=仓库缺文件（危险）
```
全树对比派生移植文件集，1–2 分钟。报 1 就把文件 `cp` 进仓库再提交；报 2 说明有源码只在工作树里。

### 刷机（**只能刷 stockdt**）
```bash
fastboot flash boot_a port/dist/chopin-ringfixNN-stockdt.img
fastboot reboot
```
`owndtb*` 变体**永远不要用** —— 自建 DTB 让 LK 的 dtbo 覆盖层 fixup 失败，LK 就不引导内核。

### 取日志
```bash
adb exec-out 'su -c "cat /dev/block/by-name/expdb"' > work/expdb/live-rNN.img   # 41943040 B
# 有时返回 0 字节，重试即可
```

### 分析 dump 的套路
一个 dump 里混着 preloader / LK / GZ / 内核四家日志，**时间戳格式相同**。
按「内核钩子前缀」分簇最可靠 —— 内核每条 printk 都以 `\x01<level>` 开头：
```python
import re
d = open('work/expdb/live-rNN.img','rb').read()
ms = [m.start() for m in re.finditer(rb"\x01[0-9]", d)]   # 每次内核启动一簇
# 注意：GZ 的页表 dump 里也有类似字节，需按 ~250 KB 间隔分簇并以已知 banner 校准
```
**判断跑的是哪一版**：看 `chopinport=ringfixNN`（cmdline 来自 `CONFIG_CMDLINE`，因为 `CMDLINE_FORCE=y`）。

### 常用判定点
```
chopinport=ringfixNN                      ← 版本
[BC 00] … [BC 08]                         ← setup_arch 走到哪一步
CHOPIN-DTB: overriding … (277450 bytes)   ← 内嵌 DTB
OF: reserved mem / mblock-                ← 保留区
mt6397-rtc mt6359-rtc: registered as rtc0 ← RTC
MTK-UFSPHY: provider registered, probe done
scsi host0: ufshcd / sd 0:0:0:2 … 128 GB  ← UFS
CHOPIN MAINLINE: userspace reached (PID 1)
arm64_serror_panic                        ← 必须为 0
```

---

## 八、陷阱清单（都真实踩过）

1. **`ioremap()` 禁止映射 System RAM** —— arm64 会 `WARN_ONCE("ioremap attempted on RAM pfn")` 然后返回 NULL。
   环必须走 `__va()`（`paging_init` 后）或 `early_ioremap()`（之前，且不可跨 `early_ioremap_reset()`）。
2. **`early_ioremap` 是全局 65 页预算**（`NR_FIX_BTMAPS`，`FIX_BTMAPS_SLOTS = 7`）——
   每次映射 17 页会在两个面包屑内耗尽并让引导死在 `paging_init()`。分块映射。
3. **日志窗口「写满就停」会隐藏死点** —— 必须是「装不下就从窗口头重来」。
4. **`/dev/kmsg` 单条上限 1024 B**（`PRINTKRB_RECORD_MAX`）且默认限流 ——
   cmdline 需 `printk.devkmsg=on`，写入要 ≤256 B 分块。
5. **GIC 的 `#interrupt-cells`**：我们自己的 `mt6893.dtsi` 是 **4**，Android 基树是 **3**，中断要按基树写。
6. **`CONFIG_CMDLINE_FORCE=y`** → DTB 里的 `bootargs` 不生效，cmdline 来自 `CONFIG_CMDLINE`。
7. **`/chosen` 必须自己写 `#address-cells/#size-cells/#size-cells`**，否则子节点 `reg` 按错误 cell 数解析。
8. **GZIP 内核头**必须 `FLG=0`、`MTIME=0`，否则 LK 报 `decompress kernel image fail!!!`。
9. **Image 的 `text_offset` 必须是 `0x80000`**，否则 LK 算错入口点。
10. **boot.img 布局必须与厂商一致**：base `0x40000000`、kernel_offset `0x80000`、
    ramdisk_offset `0x11100000`、tags_offset `0x7c8000`（DTB @ `0x47c80000`）、
    dtb_offset `0x1f78000`、pagesize 2048、header_version 2。
11. **厂商 compatible / 时钟编号与主线完全不同** —— 每补一个属性，驱动就往里走一层
    （UFS 剥了八层就是这个模式）。
12. **`mediatek,ufs-disable-mcq` 不是可选项** —— 没有它 legacy UFS 会以
    `legacy doorbell mode not supported` 失败。
13. **`mt6359_rtc` 在基树里叫 `mediatek,mt6359-rtc`，但主线只认 `mt6358-rtc`** —— 不改则 RTC 不注册。
14. **别用 `make ... | head -N` 过滤构建输出** —— `head` 提前退出会以 SIGPIPE **打断 make**，
    而 `echo $?` 拿到的是管道状态、显示 0。实测这样得到过"标签没更新、Image 没重链"的假成功。
    要过滤就把完整输出重定向到文件再 grep，并检查 make 自己的退出码。
15. **不要在照片上认数字** —— 高分屏照片缩到 1134px 后 `8/0`、`5/2`、`x/y` 极易误读。
    实测把 `0x9f6000` 读成 `0x9e9000`、`135x150` 读成 `128x75`、`0x108` 读成 `0x100`、
    `1101d000` 读成 `1101a000`，连 initramfs 的匹配计数也从照片上读错过一位。
    能拿到 dump 文本或构建产物就不要靠照片；**计数、地址、寄存器值尤其不能靠照片**。
17. **`setup_arch()` 期间调 `ioremap()` 会产生 WARNING，而且必然失败** ——
    `ioremap()` 要 `vmalloc_init()`（在 `mm_core_init()` 里，远晚于 `setup_arch()`）。
    我们的条纹代码在 `setup_arch()` 就会画，所以在那里 `ioremap_wc()` 会撞上 arm64 的
    `WARN_ONCE(pfn_is_map_memory(...))`。**正确做法是显式闸门**（`fb_window_usable`，
    由第一个非早期调用者置位），早于它就照旧入队到 `fb_pending[]`。
    这个 bug **只有在 USB 串口能读日志之后才被看到** —— 照片上看不清一行 WARNING。
16. **`mediatek,mt6893-clk.h` 一个文件里混着几套重叠的 ID 空间** —— `CLK_TOP_*`(174)、
    `CLK_INFRA_AO_*`(74)、`CLK_MDP_*`、`CLK_MM_*`…，**数值全部重叠**
    （22 同时是 `CLK_INFRA_AO_SPI0` 和别的东西）。把厂商 ID 翻成主线 specifier 时，
    **先由 phandle 定到 provider，再在该 provider 的命名空间里查**；按数值反查名字会静默取错。

---

## 九、已知良性报错清单（**不要重复排查**）

ringfix26 的完整屏幕日志（`work/device/IMG_20261002_0733*.jpg`）逐条定位过，以下全部
**在 `work/expdb/live-r23b.img`（早于第二次会话全部改动）里就已存在**，没有一条是移植引入的：

| 报错 | 位置 | 原因 | 判定 |
| --- | --- | --- | --- |
| `Initramfs unpacking failed: invalid magic at start of compressed archive` | `init/initramfs.c:739` | LK 声明的 initrd 区 `20316K`，真实 ramdisk 只有 287K；gzip 解完第一个成员后把尾部垃圾当第二个成员。`Freeing initrd memory: 20316K` 印证"区域比档案大" | **良性**：`init` 正常运行，文件已全部解出。唯一一条 ERROR 级，别被它带偏 |
| `Find ti,syscon-reset fail` | `ufs-mediatek.c:2369` | 那是 TI 平台的 syscon device-reset，MTK 无此节点，`dev_notice` 后 `skip_reset` | 良性（notice 级） |
| `mtk-spi ... failed to clk_set_parent`(-EINVAL) / `failed to get spi-clk`(-ENOENT) | `spi-mt65xx.c:1276 / 1268` | 厂商 SPI 节点引用厂商时钟 ID（MT6885 空间），我们 clk 树里对不上 | **ringfix27 已修**（16 行消失，见第二节「SPI 时钟」）；主线仍无 ST54 NFC 驱动 |
| `mt6359-keys / mt6359-accdet: Failed to locate of_node [id: -1]` | `mfd-core.c:221` | MFD cell 的 `of_compatible` 与厂商节点名不匹配 —— **与当初 RTC 同一类坑** | 良性（无驱动绑定）；将来要用就照 RTC 的手法改 compatible |
| `OF: fdt: not creating '/sys/firmware/fdt': CRC check failed` | `of/fdt.c` | 我们在 `setup_arch()` 换了 FDT，头里 CRC 与内容对不上 | 良性；代价是没有 `/sys/firmware/fdt` |
| `/dsi0@1400e000: Fixed dependency cycle(s)` | OF 核心 | 厂商 DTS 里 dsi 与 panel 互相引用 | 良性 |
| `infra_ao_pmic_ap / infra_ao_pmic_tmr: deviceless` | `clk_summary` 转储 | 未被任何设备使用的时钟（是我们 initramfs 自己打的诊断） | 良性 |
| `ufshcd_populate_vreg: unable to find vdd-hba/vccq/vccq2-supply` | `ufshcd.c` | 这些稳压器没有主线驱动；`ufs_fixed_vcc` 已覆盖 | 良性（`caps: 0x108` 里 BROKEN_VCC 生效） |
| `System clock to 2006-09-30T...` | rtc | RTC 从 epoch 起步 | 良性 |

**判断一条报错是不是移植引入的，最快的办法**：去 `work/expdb/live-r23b.img` 里 grep 一遍 ——
那是唯一带完整内核日志的 dump（详见第五节「expdb 的坑」）。
