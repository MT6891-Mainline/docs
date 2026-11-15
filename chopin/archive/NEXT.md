# 下一步：两次刷机，把"看不见"变成"看得见"

> 本轮（2026-10-01）找到了**真正的根因**：内核 log_store 驱动的 SRAM 头字段偏移错了
> 0x34 字节。所以上一轮"内核零输出 / 死在 early_initcall 之前"这个结论**不成立** ——
> 那不是对内核的测量，是对一个有 bug 的驱动的测量。详见 `SUMMARY.md`。

---

## 第 1 步：刷探针载荷（约 30 秒，屏幕直接给答案）

```bash
fastboot flash boot_a port/dist/boot-probe.img
fastboot reboot
```

（`sha256 = 32e1a2586a1c9b4e350aee8e7a7b030a0ef0211ee073c9f989ef8aad5ba4f1de`）

这是一个 **728 字节的裸机 arm64 程序**，LK 把它当内核加载执行。它分三段做：

**阶段 1（先画）**：5 个候选 framebuffer 各画**一种唯一颜色**的 600 行色带：

| 阶段 1 颜色 | 地址 | 含义 |
| --- | --- | --- |
| 🟩 绿 | `0x78e0c000` | LK 最后一次 `mt_disp_config_fb_layer` 用的地址（**最可能**） |
| 🟥 红 | `0x77a20000` | 原厂 reserved-memory 里叫 `framebuffer` 的那块 |
| 🟦 蓝 | `0x76020000` | `display-others_logo_fb-addr`（正好一帧 0x9f6000） |
| ⬜ 白 | `0x76a20000` | `logo_db_addr_pa` |
| 🟪 紫 | `0x7e020000` | `atag,videolfb-fb_base_l` |

**中间（可能出事的部分）**：往 LK 的 log_store 环（`0x7ffbf000`，每 4 KiB 一处从
`0x10000` 到 `0x3f000`）写 109 字节横幅，并按**正确的** SRAM 偏移
（`0x11df00+0x4c` 的 `sz_lk`、`+0x20` 的 `buf_point`）把记录长度撑到 `0x3f200`。

**阶段 2（后画）**：在同一批地址的第 600–699 行再画一条 100 行的色带，
颜色换成青/黄/品红/灰/橙 —— 用来证明"中间那段没把 CPU 打死"。

### 请看屏幕并告诉我

| 你看到的 | 结论 |
| --- | --- |
| 600 行色带 + 下面 100 行另一种颜色 | ✅✅ **LK 执行了我们的镜像**（阶段 1），**且 SRAM 访问没出错**（阶段 2）→ 颜色同时告诉你哪个 fb 是活的 |
| 只有 600 行色带，没有第二条 | ✅ 执行了，但**访问 SRAM 时异常了** → 要换通道（这条信息上一轮完全拿不到） |
| 屏幕完全没变（还是原开机第一屏） | ❌ 镜像没被执行，或 fb 写入被拦 —— 换思路 |
| 屏幕撕裂/花屏 | 写进去了但 stride/地址不对 |

大约 30 秒后看门狗会复位（正常现象）。**无论屏幕结果如何，都请接着做第 2 步**：
横幅是否出现在 expdb 里，是判断"通道本身通不通"的独立证据。

> 阶段 2 的位置：屏高 2400 行，阶段 1 占 0–599 行（屏幕上方约 1/4），
> 阶段 2 占 600–699 行，紧贴在它下面。

---

## 第 2 步：刷真正的内核，然后立刻取 expdb

```bash
fastboot flash boot_a port/out/boot-owndtb.img
fastboot reboot
```

（`sha256 = 62921e12ce42fbfe84fdfdb45c38940bc71c04ca77fbe9699da55a52016cbdea`；
两个变体的区别只是 DTB：`boot-owndtb` 用我们自己的设备树，`boot-stockdt` 用原厂基础树。
**先刷 owndtb**。）

### 先看屏幕（这一步现在同样有即时答案）

**真内核自己也会画屏了。** 启动流程里 13 个阶段（`head.S` 4 个 + `setup_arch()`
9 个）每到一个就在同一批 framebuffer 上多画一条 16 行的横条：

- 横条是**绿色系** ⇒ 活的 framebuffer 是 `0x78e0c000`（LK 最后一次
  `mt_disp_config_fb_layer` 指的地址）
- 横条是**红色系** ⇒ 活的 framebuffer 是 `0x77a20000`（原厂叫 `framebuffer` 的那块）
- **横条数量 = 走到的阶段号**。全部 13 条 ≈ 208 行（屏幕上方约 9%）。

所以：

| 屏幕 | 结论 |
| --- | --- |
| **蓝色**横条（整块蓝） | ⚠️ 卡在 `__enable_mmu` 的 granule 检查（`__no_granule_support`） |
| 有 N 条绿/深绿横条 | ✅ 内核在执行；停在 N 条 ⇒ 第 N+1 个路点之后就是死点 |
| 4 条（绿/深绿交替） | 走过 `head.S` 的 K00–K03，死在 `__enable_mmu` 之后、`setup_arch` 第一个面包屑之前 |
| 5 条以上 | 已经进入 `setup_arch`，看 `[BC nn]` 的最后一行 |
| 完全没有横条 | ❌ 内核的 `head.S` 第一条指令都没跑完，或 LK 没跳（换 stockdt 变体试试） |

写 framebuffer 是安全的：那块区域在原厂 DT 里**没有** `no-map`，线性映射覆盖它，
而且地址就是普通 DRAM —— 访问不可能缺页。Kconfig 开关 `CONFIG_MTK_BOOT_PROGRESS_FB`
可以随时关掉。

看完屏幕后等它卡住/自动重启（`panic=15`，看门狗约 30 秒）→ 进 Android。
**进去后立刻取 expdb，不要再重启**（每次启动都会覆盖上一轮的转存）：

```bash
# 手机上（需 root）：
su -c 'dd if=/dev/block/by-name/expdb of=/sdcard/expdb.img bs=1M'
# 拷到电脑：
adb pull /sdcard/expdb.img
```

或者只把文本带出来（更快）：

```bash
su -c 'strings /dev/block/by-name/expdb' > /sdcard/all.txt
adb pull /sdcard/all.txt
```

把 dump 放到 `work/expdb/` 下即可，我这边有专门的解析器：

```bash
port/tools/parse-expdb.py work/expdb/你的dump.img            # 全量报告
port/tools/parse-expdb.py work/expdb/你的dump.img --markers  # 只看标记命中
```

它会自动处理 40 MB 完整分区和 20 MB 尾部两种 dump，找出 pl/lk 日志区、
逐次启动的 session，并把我们的标记（`K0n`、`[BC nn]`、`[mtk-logstore]`、
`Linux version`、探针横幅、`chopinport=`、`GZ kernel exception` …）连同
所在 session 一起报出来。自带回归自测：`port/tools/test-parse-expdb.py`。

### 2026-10-01 第十三轮：★★★ 照 MT6895 的做法，DTB 塞进内核、内核自己替换 ★★★

第十二轮查明"自建 DTB 必须提供覆盖层要的 60 个符号，否则 LK 不引导"。你提醒的 MT6895
方案正是解法：**把板级 DTB 编进内核镜像的 `.rodata`，`setup_arch()` 里紧接
`setup_machine_fdt()` 之后把 `initial_boot_params` 换成我们的树**，并重读
`/chosen/bootargs`。这样 LK 那边一点都不用动。

已实现（新增 `arch/arm64/kernel/chopin-board.dts` / `chopin-dtb.S` + Makefile 规则 +
`setup.c` 替换），并顺手修掉一个真 bug：我们的 `/chosen` 缺
`#address-cells/#size-cells`，framebuffer 的 `reg` 会按错误 cell 数解析。

已验证内嵌 DTB 真的进了 `Image`（`Image+0x113e000` 处按字节找到）。

**请刷 stockdt 变体**（boot.img 里保持厂商 dtb，LK 才肯引导）：

```bash
fastboot flash boot_a port/dist/chopin-ringfix9-stockdt.img
fastboot reboot
```

期望 `init: DT model` 显示 **`Xiaomi Redmi Note 10 Pro (chopin)`** ——
那说明内核已经用自己的树了，PMIC/UFS/USB 会在同一次启动里给出 probe 结果。

### 2026-10-01 第十二轮：★★★ 自建 DTB 必须"以厂商树为底"（60 个符号）★★★

加了 `-@` 之后错误**变了但没解决**：

```
之前: Couldn't find 'mmc_fixed_3v3' ... / No node __symbols__ in main dtb
```

把设备上的 `dtbo_a` 抓下来分析，**这份覆盖层需要基树提供 60 个符号**
（`mmc_fixed_3v3`、`reserved_memory`、`chosen`、`i2c0..9`、`gpio`、`touch`、
`usb0phy`、`mtkfb`、`dsi0`、`charger` … 完整清单见 `work/dtbo-required-symbols.txt`）。
我们那份精简 DTB 提供不了 → fixup 失败 → **LK 覆盖层失败后就不再引导内核**
（AVB 失败不是原因，stockdt 也报 AVB 失败但照常启动）。

**所以下一版 DTB 要以 `work/vendor.dts`（厂商基树反编译，含全部符号）为底，
只改覆盖层不碰的节点**：`pwrap`、`ufshci`、`ufs_mphy`、`usb_xhci`、`usb3` 可以改；
`usb0phy`、`dsi0`、`mtkfb`、`mmc0`、`gpio`、`i2c*` 会被覆盖层覆盖回去。

**这一版不需要你刷机** —— 我先把这个 DTB 做出来，再给你镜像。

### 2026-10-01 第十一轮：★★★ 找到 `owndtb` 重启的真正原因（一行编译开关）★★★

上一版容器包装后**仍立刻重启**，于是去翻 LK 自己的日志，里面直接写明了：

```
ERROR: ufdt_overlay_do_fixups():No node __symbols__ in main dtb.
ERROR: ufdt_overlay_apply():failed to perform fixups in overlay
[1611] ufdt_apply_overlay() failed!
[1611] boot image dtb_overlay fail
```

**LK 应用 dtbo 覆盖层时要按名字走 `__symbols__`，我们的 DTB 没带 `-@` 编译所以没有这个节点**
→ fixup 失败 → LK 不跳内核。厂商树有 `__symbols__`，所以能启动。

已修：`arch/arm64/boot/dts/mediatek/Makefile` 加 `DTC_FLAGS_mt6893-xiaomi-chopin += -@`
（DTB 28931 → 38696 字节，`__symbols__` 已存在，节点无损失）。

同时去掉了 `initcall_debug`（已完成使命，只会淹没日志），并让 initramfs 的日志过滤器
屏蔽 initcall 噪音。

请刷：

```bash
fastboot flash boot_a port/dist/chopin-ringfix8-owndtb.img
fastboot reboot
```

这次期望 **`init: DT model` 显示 `Xiaomi Redmi Note 10 Pro (chopin)`** ——
那就说明我们自己的 DTB 终于生效了，PMIC/UFS/USB 也会在同一次启动里给出 probe 结果。

### 2026-10-01 第十轮：★★ 找齐外设阶段的前置阻塞点（需要你刷一次做判定）★★

**核心结论：必须换成我们自己的 DTB。** 内核一直在用厂商树，而厂商树的 compatible
是 `mediatek,ufshci` / `mediatek,ufs_mphy` / `mediatek,mt6885-mmc` /
`mediatek,mt67xx-xhci` / `mediatek,dsi0` / `mediatek,mt6885-phy` —— **主线驱动一个都不匹配**，
所以存储/USB/显示再怎么也起不来。唯一例外是 pwrap（之前已补 `mt6885-pwrap`）。

`owndtb` 以前失败是因为原厂 dtb 区是 **Android DTBO 容器**（`0xd7b7ab1e`），
我们塞的是裸树。新增 `port/tools/wrap-dtb.py` 复现该容器，**与原厂逐字节相同**。

同时 initramfs 现在会**回收完整 512 KB 内核日志**（不再被窗口覆盖骗），
所以这一版无论成败都能读到真实原因。

**请按顺序刷**（第一张是判定性的，成败都要看 `init: DT model` 这一行）：

```bash
# 1) 判定：LK 是否接受我们自己的 DTB（期望 DT model = Xiaomi Redmi Note 10 Pro (chopin)）
fastboot flash boot_a port/dist/chopin-ringfix7-owndtb-container.img
fastboot reboot

# 2) 若 1) 黑屏/重启不进内核，退回厂商树拿日志（型号会是 MT6891Z/CZA）
fastboot flash boot_a port/dist/chopin-ringfix7-stockdt.img
fastboot reboot
```

无论哪种，刷完都刷回原厂进 Android，我自己 adb 拉 expdb。

### 2026-10-01 第九轮：★★ **内核对 userspace 全部打通（12/12 次启动完整自证）** ★★

ringfix6 后 expdb 里有完整的 userspace 输出（见 `SUMMARY.md` 第九轮）：
`init: step 1..4`、`clk_summary`、`DT model : MT6891Z/CZA`、`no shell`，
`short write` 与 `Unable to handle` 均为 0。**前半段目标（启动到用户空间）已完成。**

**接下来是目标的后半段。** 已用日志确认三处缺口：

| 缺什么 | 具体 |
| --- | --- |
| **PMIC** | 厂商树 pwrap 用 `mediatek,mt6885-pwrap`，**(更正) 这个 compatible 只出现在我们驱动的注释里、并不在 match 表**；子节点是 `mediatek,mt6359-pmic`（表里要 `mediatek,mt6359`）—— 两处都不匹配，所以厂商树下 pwrap 从来就没 bind 过。我们自己的树两处都对得上（`mt6893-pwrap` + `mt6359`） |
| **存储** | 驱动都编了，但 `mmc@11230000` / `ufs@11270000` 在板级 DTS 里没 `okay` |
| **显示** | `simple-framebuffer` 没 probe（`FB_SIMPLE=y` 已编），需查原因或改用 `DRM_SIMPLEDRM` |

### 2026-10-01 第八轮：userspace 输出被 `/dev/kmsg` 的两条限制吃掉了

刷 ringfix5 后，expdb 里**真的出现了 userspace 的输出**（8 次启动都出现）：

```
 <6>Freeing unused kernel memory: 3392K
 <6>Run /init as init process
 ...
 CHOPIN MAINLINE: userspace reached (PID 1)
=============================================
init: kernel    : Linux version 7.2.8 ... #45 SMP PREEMPT
init: cmdline   :        <-- 到这里就没了
```

**8 次启动全部停在 `init: cmdline   :`**，`/proc/cmdline` 的内容一个字都没出来。
翻 `kernel/printk/printk.c` 的 `devkmsg_write()`，两条限制同时命中：

| 限制 | 后果 |
| --- | --- |
| `if (len > PRINTKRB_RECORD_MAX) return -EINVAL;`（`PRINTKRB_RECORD_MAX = 1024`） | `/proc/cmdline` 有 ~1.6 KB，**一次 `write()` 直接 `-EINVAL`，内容全丢** |
| `if (!(devkmsg_log & DEVKMSG_LOG_MASK_ON)) { if (!___ratelimit(...)) return ret; }` | `devkmsg_log` 默认是 `ratelimit` —— **前几条之后全部被静默丢弃** |

**ringfix6 的两处修正**：

1. cmdline 加 **`printk.devkmsg=on`** → 关掉 userspace 写入的限流。
2. `port/initramfs/init.c` 的输出改为**分块写入（每块 ≤256 B）**，并对短写报错；
   同时加了 `init: step 1..4` 路点（挂载伪文件系统 / 设备树快照 / 诊断）。

```bash
fastboot flash boot_a port/dist/chopin-ringfix6-stockdt.img
fastboot reboot
```

期望看到 `init: step 1..4` 全部出现，以及 `clk_summary`、`/proc/interrupts` 快照 ——
那样 userspace 侧的观测面就完整了。

### 2026-10-01 第七轮：★ **内核已经启动到用户空间** ★

修好 `ioremap` 之后，日志不再被窗口溢出骗到，内容变成了**完整的一次启动**：

```
?6Freeing unused kernel memory: 3392K
[    4.450919] Freeing unused kernel memory: 3392K
?6Run /init as init process
[    4.451631] Run /init as init process
?7  with arguments:        ?7    /init
?7  with environment:      ?7    HOME=/   TERM=linux   chopinport=...
```

而且日志里有**完整的时间戳**（`[    4.450919]`）—— 那是我们注册的 console 在被
printk 调用（钩子那份没有时间戳），两条同时出现，说明两个通道都在工作。

迹象汇总：

| 观察 | 含义 |
| --- | --- |
| 十二次 `Freeing unused kernel memory` + `Run /init as init process` | 十二次启动都**走到了 exec /init** |
| 日志覆盖到 `late_initcall`（`clk_disable_unused`、`genpd_power_off_unused`、`regulator_init_complete`…） | **所有 initcall 全部跑完** |
| `Attempted to kill init` / `Kernel panic` / `Failed to execute` 均为 0 | PID 1 **没有死、也没有 exec 失败** |
| 全部 clk 驱动 probe 成功（`clk_mt6893_*`、`topckgen`、`infracfg_ao`…） | 时钟子系统起来了 |

**所以目标里的"使内核启动到用户空间"已经达成。**

唯一还看不见的是 **PID 1 自己的输出**：内核启动时用的是
`console=tty0 console=ttyS0,...`，`/dev/console` 绑在 **dummy tty0** 上，
userspace 写进去的东西全部消失。修法见下。

### 2026-10-01 第七轮附：让 userspace 的输出可见

`port/initramfs/init.c` 现在**第一步就把 fd 0/1/2 重定向到 `/dev/kmsg`**：

```c
fd = open("/dev/kmsg", O_WRONLY);
dup2(fd, 0); dup2(fd, 1); dup2(fd, 2);
```

写 `/dev/kmsg` 会走 `printk()` → 我们的 `vprintk_emit` 钩子 → 环 → LK → expdb。
**一行改动就让 userspace 半边也进入可观测范围**，不需要串口。

请刷：

```bash
fastboot flash boot_a port/dist/chopin-ringfix6-stockdt.img
fastboot reboot
```

期望在 expdb 里看到 `CHOPIN MAINLINE: userspace reached (PID 1)` 以及后面的
`/proc/version`、`clk_summary`、`/proc/interrupts` 快照 —— 那样下一步
（PMIC → 存储 → USB → 显示）就有了完整的观测面。

### 2026-10-01 第六轮：**根因找到了 —— arm64 禁止 `ioremap()` 系统 RAM**

ringfix4 的屏幕仍然只有 **1 条黄条**（间距两格，所以这次可以确定只有步 0），
而步 0 与步 1 之间只有一件事：`ioremap(0x7ffbf000, 0x40000)`。翻内核源码，答案立刻就出来了：

```c
/* arch/arm64/mm/ioremap.c */
void __iomem *__ioremap_prot(phys_addr_t phys_addr, size_t size, pgprot_t pgprot)
{
        ...
        /* Don't allow RAM to be mapped. */
        if (WARN_ONCE(pfn_is_map_memory(__phys_to_pfn(phys_addr)),
                      "ioremap attempted on RAM pfn\n"))
                return NULL;
```

**arm64 的 `ioremap()` 不允许映射系统 RAM** —— 它会 `WARN_ONCE` 然后返回 `NULL`。
log_store 环（`0x7ffbf000`）是普通 DRAM（reserved 但没有 `no-map`），所以：

1. `ioremap()` 打印一条 WARN + 调用栈（这正是 dump 里 `Unable to handle` / `Call trace` 的来源）；
2. 返回 `NULL` → `logstore_setup()` 返回 `-ENOMEM` → **驱动器直接退出**，
   所以 `fb_note(1)` 没画（1 条黄条），也永远没有 `[mtk-logstore]`。

**修正（ringfix5）**：改用**线性映射别名** `__va(buf_pa)`，并在每次写入后
`dcache_clean_poc()`（LK 和显示控制器走它们自己的映射）。这正是 `paging_init()`
之前那些面包屑一直在用的地址 —— 它本来就是好的，只有 `setup()` 走了弯路。

```bash
fastboot flash boot_a port/dist/chopin-ringfix6-stockdt.img
fastboot reboot
```

期望：`[mtk-logstore]` 自检行出现，`register_console` 的 `CON_PRINTBUFFER`
回放把**整段 printk 缓冲**灌进窗口（于是我们能拿到从 `Linux version` 开始的完整日志），
黄条应该明显多于 1 条或者屏幕直接有别的变化。

### 2026-10-01 第五轮：日志窗口会溢出 —— 先修好"看得见"，再谈定位

`initcall_debug` 打开了，日志量暴增（一次启动 ~3500 行）。结果暴露出**我自己的一个设计缺陷**：

**两个日志窗口是"写满就停"，不是环形。** 写满之后 `logstore_append` 直接不再写，
于是窗口里的最后一行**只是"窗口写满的位置"，不是"内核死掉的位置"**。
ringfix3 的 dump 就踩了这个坑：最后一行停在某个 clk initcall，看起来像是卡在那里，
其实是窗口满了。（ringfix2 那次窗口只有 ~13 KB，远没写满，所以当时
"最后一条 calling 是 mtk_logstore_early_init"的结论仍然成立。）

**ringfix4 的修正**：

1. **窗口改为"装不下就从头重来"**，而不是"写满就停"。代价是最旧的输出被覆盖，
   但保证**最新的输出一定从窗口开头连续存放** —— 于是窗口里的最后一行就是死点。
2. **黄色步骤横条的间距改成两格**（原来相邻，两条会连成一块，数不出来）。
   现在每条黄色之间空一格，**黄色数量 = logstore 驱动走到第几步**：
   0=进入 `mtk_logstore_early_init`，1=两个映射建立，2=首次 SRAM 读返回，
   3=读全返回，4=写返回，5=publish，6=console 结构就绪，7=register_console 返回，
   8=首次 printk 返回。

ringfix3 的屏幕只看到**一条**黄条（步 0），所以嫌疑落在
`ioremap(0x7ffbf000, 0x40000)` 上（步 0 与步 1 之间只有这一件事）。
但旧布局里步 0/1 的横条是相邻的，可能是两块被看成一块 —— ringfix4 会给出确定答案。

```bash
fastboot flash boot_a port/dist/chopin-ringfix6-stockdt.img
fastboot reboot
```

### 2026-10-01 第四轮：**initcall 追踪一击命中 —— 卡在我们自己的 logstore 驱动里**

`initcall_debug` 生效，日志里每个 initcall 都有 `calling <函数名>`。最后一条是：

```
7calling  rcar_sysc_pd_init+0x0/0x320 @ 1
7calling  mtk_logstore_early_init+0x0/0xf0 @ 1     ← 最后一条，之后什么都没有
```

所以卡点在 **`mtk_logstore_early_init()` 里面**，而且连 `[mtk-logstore]` 自检行都没出来
→ 卡在 `logstore_setup()` 的早期。最可疑的是**读 SRAM 头 `0x11df00`**：
那是一次总线访问，挂住是"静默死"，不是干净缺页；而 **低窗口本来就落在 LK
天然转存的范围里，SRAM 头其实不需要**。

**已做的修正（ringfix3）**：

1. **SRAM 头访问改为可选，默认关闭**（`CONFIG_MTK_LOGSTORE_SRAM_HEADER`）。
   不 ioremap `0x11df00`、不读不写；低窗口照旧工作。
2. **printk 镜像不再在 setup 期间被关掉**（原来 `ls = NULL` 会让 oops 文本丢失 ——
   这正是"最后一条 calling 是我们、之后什么都没有"的原因）。
   现在映射一建立就接管镜像，并且**在 SRAM 读成功之前绝不碰 SRAM**（防止缺页递归）。
3. 加了 **黄色步骤横条**（在第 13 条绿色之后、留一段空档）：
   黄色数量 = logstore 驱动走到第几步，屏上直接可读。

请刷 **stockdt**：

```bash
fastboot flash boot_a port/dist/chopin-ringfix6-stockdt.img
fastboot reboot
```

- 屏幕：13 条绿 + N 条黄。**N 少一条 = 卡在第 N+1 步**（0=进入 early_init，
  1=两个映射建立，2=第一次 SRAM 读返回，3=读全返回，4=写返回，5=publish，
  6=console 结构就绪，7=register_console 返回，8=第一次 printk 返回）。
- 如果 SRAM 头就是元凶，现在它已经被跳过，内核应当**继续往下走**，
  屏幕上会出现更多黄色横条或整屏变化。

### 2026-10-01 第三轮：内核已跑完 setup_arch，正在定位卡住的 early_initcall

修掉回归后，内核一路开到 `do_initcalls`：expdb 里有**完整可读的内核日志**
（GICv3 832 SPIs、arch_timer 13 MHz、console、VFS cache、RCU 全就绪），
最后一行来自 `early_initcall(tmigr_init)`。所以卡在 `tmigr_init` 与
`drivers/misc` 之间的某个不打印的 early_initcall 里。

**本轮 cmdline 加了 `initcall_debug`**：每个 initcall 都会打印
`calling  <函数名> @ <pid>`，被日志钩子原样镜像。日志里最后一个
`calling X` 就是卡住的那个。

请刷 **stockdt** 变体（`owndtb` LK 不跳，先别用）：

```bash
fastboot flash boot_a port/dist/chopin-ringfix6-stockdt.img
fastboot reboot
```

### 2026-10-01 第二轮实测结果（已用 adb 取证 + 修复）

| # | 结论 | 证据 |
| --- | --- | --- |
| 1 | `chopin-ringfix1-stockdt.img`：内核执行到 **K03** 后停住 | 屏幕 4 条横条（浅绿/深绿交替 = 阶段 0/1/2/3） |
| 2 | **K0n 标签真的回来了** —— 低窗口在真内核上同样有效 | `parse-expdb.py` 在 4 个 session 里找到 `K00`…`K03`，位置正是 ring `+0x800`…`+0x830` |
| 3 | GZ 以 64 位启动且**没有异常** | `boot linux-64` ×2，`GZ kernel exception` 0 次 |
| 4 | **这次停在 K03 是我自己的回归，不是内核原有的 bug** | 我把早期面包屑改成了走 `logstore_append()` → `ring_write_va()` → `__va()`，而它在 `paging_init()` **之前**跑，线性映射还不存在 → 第一次 `[BC]` 就取数异常。内核自己的异常向量已装好，所以 GZ 不会报 —— 看起来就是"4 条后静默死"。已修回 `early_ioremap` + `memcpy_toio`。 |
| 5 | `chopin-ringfix1-owndtb.img` **完全没有横条** | 对应 session 里没有 `jump to K64` → **LK 用我们自己的 DTB 时根本没跳内核**。所以现阶段请用 **stockdt** 变体。 |

另外新增：`__no_granule_support`（`__enable_mmu` 里唯一能到达的静默 `wfe/wfi` 死循环）
现在会画一条**蓝色**横条并打 `K09` 标签 —— 这样下一次如果还是 4 条，
就能区分"卡在 `__enable_mmu`"和"死在 MMU 打开之后的盲区"。

### 实测已经确认的事情（2026-10-01，探针 + adb 取证）

| # | 结论 | 证据 |
| --- | --- | --- |
| 1 | **LK 确实加载并执行我们的 arm64 镜像** | 探针把屏幕画成绿+青（阶段 1 和阶段 2 都执行了） |
| 2 | **活的 framebuffer 是 `0x78e0c000`** | 屏幕是**绿色**系（`0x78e0c000` = 绿），不是红（`0x77a20000`） |
| 3 | **SRAM 头 `0x11df00` 在 EL1 可访问**（不触发异常） | 阶段 2（青色，在写 SRAM 之后才画）出现了 |
| 4 | **expdb 转存通道可用**，探针的 48 条横幅全部回来 | `parse-expdb.py` 在 2 个 session 里各找到 48 条，ring `+0x10000`…`+0x3f000` |
| 5 | 但**记录长度不跨复位保留**：下一次启动 LK 只转存它自己那段（约 126 KiB） | 探针之后的 stock 启动里，`ring > 0x1D000` 的横幅一条都没有 |
| 6 | 因此内核标记必须放在 **LK 天然会转存的范围（≈`0x2000`–`0x10000`）** 里 | MT6895 已跑通的 `xaga-marker-writer.c` 把整个环放在 `0x2000..0x10000`，且**完全不碰 SRAM 头** |

根据第 5、6 条，内核现在**同时写两个窗口**：低窗口 `0x0800..0x10000`
（MT6895 验证过的范围）和高窗口 `0x20000..0x3f000`（万一长度被保留）。

### 日志侧的四类独立标记（**两条互不相干的通道**）

屏幕是一条通道，expdb 是另一条。**任何一条单独成立就能推进**：

| 标记 | 位置 | 说明 |
| --- | --- | --- |
| `K00` `K01` `K02` `K03` | ring `0x3e000`+ | `head.S` 汇编级面包屑，MMU 关闭时物理直写。`K03` 出现 ⇒ 走到 `__primary_switch` |
| `[BC nn] <阶段名>` | ring `0x3e000`+ | `setup_arch()` 里 9 个路点，`paging_init` 前后都有 |
| `[mtk-logstore]` / `Linux version` | ring `0x20000`–`0x34000` | 真正的 printk 文本（`CON_PRINTBUFFER` 回放 + `vprintk_emit` 同步镜像） |
| `chopinport=ringfix1` | LK 日志 | 确认刷的是这一版 |
| `jump to K64 0x40080000` | LK 日志 | 确认 LK 跳了内核 |

内核往环里写日志时，**同时写三个长度字段**（`sz_lk`@SRAM+0x4c、`buf_point`@+0x20、
`reserve[0]`@+0x5c），三者都指向 `0x3f200` —— 这样无论 LK 用哪一个来判断
"要转存多少"，我们的数据都在里面。

---

## 回滚

```bash
fastboot flash boot_a n10p/boot_a.img
```

（该文件与设备分区前 1 MB 的 sha256 已核对一致。）

---

## 如果第 1 步屏幕完全没变

那就是"我们的镜像没被执行"，需要换一组判据重新做实验（例如换 `RAW_KERNEL=1`
未压缩打包、或改 LK 头部字段）。把这一个事实告诉我即可，我会据此设计下一轮。
