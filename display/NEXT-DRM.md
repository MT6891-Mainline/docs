# 下一轮上机：DRM 显示（ringfix62，**首次真 DRM**）

> 本轮（2026-10-03）把 MT6893 的 DDP/DSI/面板在主线里补齐了：MMSYS 路由表、
> 显示 mutex、mt6893-dsi、k10a 面板驱动、板级设备树节点、defconfig。
> **已上机一次**：驱动侧整条 bind 成功，但屏幕仍黑 —— 结果与下一步见下面第〇节。

---

## 〇、上机结果（2026-10-03 第一次，单模式构建）

设备实测：

| 证据 | 值 |
| --- | --- |
| `card0-DSI-1/status` | `connected` |
| `card0-DSI-1/enabled` | `enabled` |
| `card0-DSI-1/dpms` | `On` |
| `card0-DSI-1/modes` | `1080x2400`（**一行 ⇒ 刷的是单模式构建**）|
| `card0/device/graphics/fb0` | 存在（DRM fbdev 注册成功）|
| `card0/device/device/uevent` | `DRIVER=mediatek-drm` |
| 屏幕 | **仍黑** |

**结论：整条软件栈 bind 成功**（mmsys→clk-mt6893-mm→DDP 七组件→mt6893-dsi→
mipi_tx→panel→connector→fbdev），这是第一次。剩下的问题是"为什么没有像素"。

### 用现有镜像就能做的三件事（**不需要重刷**）

```bash
chopin# drm                                   # 要完整输出，别在 /proc/fb 处截断
chopin# irq                                   # 看 disp/dsi/mutex 计数
chopin# dmesg drm                             # 或 dmesg mtk-dsi / dmesg panel
chopin# write /dev/tty0 HELLO                 # ★ 让 fbcon 往 DRM framebuffer 写字
```

**★ 人眼先看两件事**（比任何日志都直接）：

1. **屏幕有没有背光**？完全漆黑（连一点灰光都没有）与"背光亮着但全黑"是两回事 ——
   背光由 LM36273（I2C，厂商私有驱动，本轮完全没碰）提供，LK 点亮后状态应当保留；
   我们的初始化序列还会写 `0x51 = 0x03 0xFF`（面板侧最大亮度）。若连背光都没有，
   那"面板不解码"和"背光没开"就混在一起了。
2. `write /dev/tty0 HELLO` 之后屏幕上有没有字？`console=tty0` + `FRAMEBUFFER_CONSOLE=y`
   意味着内核日志本来就一直画在 DRM 的 framebuffer 里（`__FB_DEFAULT_DMAMEM_OPS_DRAW`
   直接写 `screen_buffer`，就是被扫描的那块内存），所以**如果面板能解码，屏幕上应该
   早就有满屏内核日志**。若写进去也没反应，基本就锁定"链路在发、面板不解码"。

判读：

| 现象 | 结论 |
| --- | --- |
| `clk_summary` 里 `mm_dsi0_mm_clk`/`mm_dsi0_intf_clk`/`mipi_tx0_pll` 都是 enabled，且 **disp/dsi/mutex 中断计数在涨** | DSI 真在发数据 ⇒ 问题在面板侧（最可能：不接受非压缩流）⇒ 直接做 DSC |
| 时钟 enabled 但**中断恒 0** | CRTC 配了但没真正跑起来（mutex/timing 问题）⇒ 查 dmesg 里的 DSI/mutex 报错 |
| 时钟根本没 enable | CRTC 没 enable 成功 ⇒ 看 dmesg 里 atomic commit 的报错 |
| `/proc/fb` 为空（第一次抓到的样子）| fbcon 没拿到 fb（画面即使扫出来也是黑的）⇒ 查 `dmesg fb`、`/sys/class/graphics/fb0` |

### 已上机第二次（同一张两张模式的构建）

| 现象 | 值 | 含义 |
| --- | --- | --- |
| 开机时 **simplefb 阶段能看到开机日志**，随后很快黑屏 | —— | ★★ **面板 + LK 的 DSI 配置（DSC 模式）是好的**：LK 那套配置能把 framebuffer 内容显示出来 |
| `cat /sys/class/drm/card0-DSI-1/modes` | `1080x2400` **两行** | 确认刷的是两模式的新构建 ✓ |
| `chopin# fbpaint` | **没有彩条** | DRM 接管后整条链路没有出图（不是"没人画"的问题） |

两条合起来把范围缩到：**LK 配置能亮、我们这套接管后不亮**。剩下的分歧只有两种：
(a) 面板不解码我们的非压缩流（DSC 必需）；(b) 我们的 DSI/PHY 寄存器编程本身没把
数据发出去（或时序不对，面板锁不上）。

### 已上机第三次（`drm` 的 DRM state + `fbpaint` 仍无彩条）

`/sys/kernel/debug/dri/0/state` 说明 **DRM 侧已经完全就绪**：

| 证据 | 值 | 含义 |
| --- | --- | --- |
| `crtc[61]: crtc-0` | `enable=1 active=1`，`mode: "1080x2400": 36 **139000** 1080 1564 1574 1582 2400 2454 2464 2474` | 首选 834 Mbps 模式被选中且在跑 |
| `plane[37]: plane-0` | `fb=62`、`allocated by = [fbcon]`、XR24、pitch 4320 | fbcon 的 framebuffer 挂在 plane 上 |
| 同上 `obj[0]` | `dma_addr=0xf8b00000`、`size=10371072` | 显示引擎要读的物理地址（无 IOMMU，直通物理地址）|
| `encoder-0/bridges` | `mtk_dsi_bridge_funcs` + `panel_bridge_bridge_funcs`（OF `/dsi@1400e000/panel@0:xiaomi,k10a-36-02-0a`）| DSI 与面板桥都接上了 |
| `/proc/fb` | `0 mediatek-drmdrmfb` | fbcon 确实绑在 DRM framebuffer 上 |
| `fbpaint` | **仍无彩条** | 所以不是"没人画"，是**硬件没把画面送到面板** |

两个附带发现：

- **`/sys/class/drm/card0-DSI-1/dpms` 写不了**（`Permission denied`）：连接器 sysfs 属性只允许
  DRM master 写，initramfs 的 shell 不是 master。**A/B 改用 unbind/bind**。
- `dmesg panel` 里没有回读那行（可能被 `drm.debug=0x06` 顶掉）⇒ 用 `dmesg readback` 精确过滤。

### 第四次上机（结果：证据没拿到，先修观测通道）

这一轮拿到两条有价值的线索：

1. **`grep dsi0 /sys/kernel/debug/mediatek-drm/disp-regs` → `(missing)`** ⇒
   上机用的镜像**早于**寄存器 dump（只带 `force_dsc_decoder` + 回读那份），
   所以寄存器证据根本没拿到。
2. `dmesg readback` **0 行** 与 `drm` 输出两次都在 `=== /proc/fb ===` 处断掉：
   原因是 `ignore_loglevel` + `drm.debug=0x06` 把 256 KiB 的 printk 环冲爆了，
   早期（DRM bind 时）的日志全部被顶掉；再加上递归 `ls`/目录 dump 把 115200 的
   USB CDC 灌满导致掉字符（日志里出现 `脱机`，命令行甚至被拼坏成
   `init: unknown command`）。
3. `irq` 表（名字被终端截断成 4 字符，但仍能读出数量变化）：

   | hwirq | 设备 | 计数 |
   | --- | --- | --- |
   | 285 | `14000000.disp_ovl`（OVL0，**在我们的路径里**）| 9 → 9（不动）|
   | 286 | `14001000.disp_ovl`（OVL0_2L，**不在路径里**）| 2007 → **2163**（在涨）|
   | 287 | `14003000.disp_rdma`（RDMA0，**在路径里**）| 0 → 0 |

   DSI0（hwirq 294）与 mutex（hwirq 283）那几行被截断，没看到 —— 但
   **路径里的 OVL0 与 RDMA0 都没有按帧产生中断**，而一个不在路径里的块在猛涨。
   这需要 `grep dsi /proc/interrupts` 之类的短输出来确认。

### 第五次上机：**DSI 寄存器解码 = 完全正确**

`dmesg enabled` 拿到了新加的那行（时间戳 12.43 s —— fbcon 打开 DRM 节点时才触发 modeset）：

```
enabled: START=0x1 MODE_CTRL=0x1 PSCTRL=0x30ca8 SIZE_CON=0x9600438 VSA=10 VBP=10 VFP=54 VACT=2400...
```

| 寄存器 | 值 | 含义 |
| --- | --- | --- |
| `START` | `0x1` | **视频流已启动** |
| `MODE_CTRL` | `0x1` | `VIDEOMODE_SYNC_PULSE`，正是本面板要的模式 ✓ |
| `PSCTRL` | `0x30ca8` | `PS_SEL=3`（RGB888）+ `PS_WC=0xca8=3240=1080*3` ✓ 非压缩 24bpp 完全对 |
| `SIZE_CON` | `0x9600438` | `(2400<<16) | 1080` ✓ |
| `VSA/VBP/VFP` | `10/10/54` | 与厂商一致 ✓ |

⇒ **DSI 的模式 / PS / SIZE / 时序编程完全正确**。"DSI 根本没发出去"这条在寄存器
层面已无疑点，剩下的分歧收敛为：(a) DSI 时钟或 PHY 实际没跑；(b) 面板不解码非压缩
流（**DSC 必需**）。

另外两点：`dmesg readback` 仍是 0 行（4 MiB 环已生效，不是被冲掉）；
`disp-regs` 仍是 `(missing)` ⇒ 本轮把 dump 改成 **debugfs 根目录下的单文件**
`/sys/kernel/debug/mediatek-drm-regs`，并**加了成功/失败日志**，下次 dmesg 里直接能看到。

### 第六次上机：DSI 中断"不存在" + 黑屏后 shell 卡死

两条新现象：

1. `grep dsi /proc/interrupts` → **无匹配**。但 `mtk_dsi_probe()` 里
   `platform_get_irq()` 与 `devm_request_irq()` **任一失败都会让 probe 直接失败**
   （`drivers/gpu/drm/mediatek/mtk_dsi.c:1260/1275`），而 card0 存在、DSI 桥接上了、
   DSI 寄存器也被正确编程 —— 所以中断**必然已经注册**。抓不到只可能是
   USB 控制台当时已经开始掉字符（同一个卡死问题的前兆）。
2. **黑屏后 USB 串口很快卡住、shell 无法使用** —— 这解释了前几轮所有的
   "输出被截断 / 命令被拼坏 / 出现 `init: unknown command`"：那不是刷屏，
   是**系统在显示接管后不久就卡死**。

已经卡死过一次的机器上没法用 shell 命令取证，所以这轮把取证做成**卡死也留痕**：

- shell 空闲循环里每 ~2 s 采样一次 `/proc/interrupts`，只要有 IRQ 在猛涨就写一行
  `irqwatch: #N top movers (2 s): <设备名> +<增量> ...` 到内核日志；
- 内核日志由 logstore 驱动留在 DRAM，**下次启动会被 initramfs 恢复**
  （你贴的日志开头就是 "init: recovered 87730 bytes of kernel log"）；
- 所以卡死 → 重启 → `dmesg irqwatch` 就能看到**是哪个中断在风暴**、以及最后一次
  心跳（"alive"）停在什么时候。
- DSI 驱动另外在 probe 成功时打一行 `ready: irq N, L lanes`，这样即使在
  /proc/interrupts 名字被截断的终端上也能对上号。

### 第七次上机：**根因锁定在 DSI 中断没送到 CPU**

`dmesg panel` 拿到了决定性的一行：

```
panel-k10a-36-02-0a 1400e000.dsi.0: prepare: init sequence, DSC decoder forced off
panel-k10a-36-02-0a 1400e000.dsi.0: failed to write command 0 (0xff): -62
panel-k10a-36-02-0a 1400e000.dsi.0: failed to initialize panel: -62
```

`-62 = -ETIME`，来自 `mtk_dsi_wait_for_irq_done()`，路径是：

```
panel prepare → mipi_dsi_dcs_write_buffer
  → mtk_dsi_host_transfer()          (视频模式已开)
     → mtk_dsi_stop()
     → mtk_dsi_switch_to_cmd_mode(dsi, VM_DONE_INT_FLAG, 500)
        → mtk_dsi_wait_for_irq_done(…, 500)  → 超时 → -ETIME
```

关键点：`mtk_dsi_poweron()`（在 **DSI 桥的 atomic_pre_enable** 里跑，早于面板 prepare）
已经把控制器切到**视频模式**，所以面板的第一条命令必须等"视频→命令"切换的
**VM_DONE 事件**；这个事件的中断**从未到达 CPU**，于是每条命令都超时、
面板永远初始化不了 ⇒ 黑屏。这与上一轮 `grep dsi /proc/interrupts` 无匹配、
以及 `mtk_dsi` 自己的 `ready: irq` 那行缺失，指向同一件事：**中断投递有问题**
（GIC 线接错，或显示中断 mux 没被编程），而不是控制器/PHY/面板本身。

同一轮的其它读数：`dmesg mmsys` 里 `mmsys routes read back:` 出现了 ✓（路由读回生效）；
`irqwatch: #30 alive, nothing moving` 说明采样器在跑但**解析有 bug**（漏了 IRQ 号后的
冒号，所有计数都算成 0），本轮已修。

### 第八次上机：采样器证明"显示相关中断一个都没动"

`irqwatch` 现在能用了（修掉了漏冒号的解析 bug），结果是：

```
irqwatch: #2  top movers (2 s): arch_timer +152, 11201000.usb +32, 11f00000.i2c5 +4
...（每一拍都是这三行，arch_timer +560 左右）
```

**top-3 里没有任何显示块**（若有，35 Hz 的面板帧率会是每 2 s `+70`，必然进前三）。
即：这一轮启动里 **OVL / RDMA / DSI / mutex 一个中断都没产生**。结合 `dmesg panel`
的 `-62`（面板初始化第一条命令就失败）可以推断：**面板 prepare 失败 → 原子提交失败 →
CRTC 从未 enable → DDP 根本没启动**，所以连 OVL 中断都没有。也就是说：

> 现在唯一挡在前面的就是 **DSI 命令通道**；DDP/中断/面板/背光都还没轮到。

另外两个本轮修掉的问题：

1. **采样器之前根本没写进内核日志**：`main()` 把 fd 1 dup 到 ttyGS0 之后，
   `kmsg_tx()` 的两条路径都指向同一个 tty，于是 irqwatch 只重复出现在控制台上、
   一个字节都没进 printk，也就没进 logstore 的 DRAM 环（这就是"日志到 3.2 s 就没了"）。
   现在改为**自己打开 `/dev/kmsg` 写**，所以卡死前的采样真的能被下次启动读回来。
2. 采样器**每拍都打一行**（arch_timer 恒在动）导致刷屏；现在只报：显示块注册时一行、
   显示块有变化时的行、每 ~30 s 一次心跳、以及任何每拍 >2000 的"风暴"行。

### 第九次上机：**挂住的原因是我自己写的轮询循环**（已修）

三条读数：

1. `dtprop /sys/firmware/devicetree/base/dsi@1400e000/interrupts` →
   `00 00 00 00 | 00 00 01 06 | 00 00 00 04` = **GIC_SPI 262、level-high** ✓
   设备树是对的（那个属性本来就是 3 个 cell），**DT 排除**。
2. `dmesg panel` 的最后一行是 `prepare: init sequence, DSC decoder forced off`（11.854 s），
   **既没有 `failed to write command`，也没有 `Timeout`，也没有 `polling`** ——
   说明那条命令**根本没返回**。而 `dmesg irqwatch` 里 34.28 s 的心跳证明
   **内核、printk、shell 都还活着** ⇒ 是**线程卡死**，不是整机死机。
3. 这个组合指向的唯一位置：我上一轮加进 `mtk_dsi_wait_for_irq_done()` 的轮询代码里，
   **我照抄了中断处理里那个没有边界的 `do { … } while (tmp & DSI_BUSY)`**。
   控制器若不清 `DSI_BUSY`，这个循环就在**进程上下文里永远转**：不超时、不打印、
   线程再也回不来 —— 与 1/2 的现象完全一致。

本轮修复（`076bad1`）：

- 轮询路径里的 RACK 握手**加 20 ms 边界**，并且**无论如何都清掉锁存的标志**
  （标志才是 transfer 要等的东西），握手仍忙时打一条 ratelimited 警告；
- 中断处理里的同一个循环也**加 50 ms 边界**（`b1c944b`）—— 它原来会在
  **关中断的 IRQ 上下文里**空转，那是"显示起来后整块板子不响应"的合理机制；
- 新增**首帧 transfer 步骤跟踪**（只打第一条命令，之后自动停）：
  `xfer[1] entry` / `xfer[2] stopped video mode` / `xfer[3] cmd mode ok|FAILED` /
  `xfer[4] lane ready` / `xfer[5] send cmd ok|FAILED`，每条都带
  `MODE_CTRL/INTSTA/INTEN` —— **缺哪一步，卡就卡在哪一步**；
- 采样器首帧把**整张 `/proc/interrupts` 原文**写进持久日志（不再按名字过滤，
  因为"没有 dsi/mutex 行"这件事已经骗过我们两次）。

### 第十次上机：`tasks` 抓到真凶形态 + printk 轨迹失效

**`tasks` 这次抓到了卡住的线程**：

```
 85   D  kworker/2:1+events            __flush_work           ← 有人在 flush 一个永不完成的工作
119   R  kworker/u32:4+events_unbound  0                      ← 在跑/自旋
126   D  kworker/3:2+events            drm_client_dev_hotplug ← ★ 执行初始 modeset 的工作项卡在这里
129-131 S card0-crtc0/1/2              kthread_worker_fn      ← CRTC 自己的 kthread（空闲）
```

`drm_client_dev_hotplug` 正是那条**执行初始 modeset（→ 面板 prepare）**的工作项 ⇒
modeset 线程确实卡在面板/DSI 这条路上，而且是 **D（不可中断）**状态 —— 而
`wait_event_interruptible_timeout` 是 **S**，所以它卡在一个**不可中断的等待**（mutex 类）里。

同时：**`dmesg xfer` 这次 0 行**（连 `xfer[1] entry` 都没有），但同一时刻 shell 写入
`/dev/kmsg` 的 34.4 s 心跳正常 ⇒ **printk 是活的、我们的 dev_info 轨迹却没出来**
⇒ 不能再拿 printk 当观测通道。

同一轮其它读数（都能对上）：
- `ovl started: EN=1 DATAPATH_CON=7000005`、`rdma started: GLOBAL_CON=101 SIZE0=438 SIZE1=960`
  ⇒ **CRTC 的 DDP 初始化跑过了**（SIZE0=1080、SIZE1=2400 ✓）；
- `irqwatch: #15 disp name=…` 三行（新格式 ✓）：OVL0=6、OVL_2L0=4、RDMA0=0 ⇒ 注册了但都不按帧中断；
- 6.0 s 的整表 dump 里没有显示中断，是因为显示设备 **11.8 s** 才 probe（我上轮期待错了）。

### 第十一次上机：完整日志把故事讲完了 —— **LK 把 DSI 以视频模式交给我们**

```
11.838319 panel-k10a: prepare: init sequence, DSC decoder forced off
11.881482 mtk-dsi: xfer[1] entry:             MODE_CTRL=0x1 INTSTA=0x17b2 INTEN=0xb
11.881493 mtk-dsi: xfer[2] stopped video mode: MODE_CTRL=0x1 INTSTA=0x17b2 INTEN=0xb
11.881501 mtk-dsi: xfer[3] wait: enter:        MODE_CTRL=0x0 INTSTA=0x17b2 INTEN=0xb
12.399467 mtk-dsi: xfer[4] wait: timed out:    MODE_CTRL=0x0 INTSTA=0x17b2 INTEN=0xb
（之后直到 31.7 s 的 regulator 消息之前，没有任何面板/DSI 日志）
```

要点：

1. **面板第一条命令之前，控制器就已经在视频模式**（`MODE_CTRL=0x1`），
   而且 `DSI_START` 还开着、状态锁存着：`INTSTA=0x17b2` —— 其中
   bit1 = `CMD_DONE`，其余（bit4/5/6/7/9/10/12）是 FIFO/错误位。
   ⇒ 这是 **LK 留下的、正在运行的 DSI 视频流**（它当年用的 framebuffer 早被 Linux 复用，
   所以挂着一堆 underrun/error 位）。**主线从没处理过这种情况**：它支持的 SoC 上
   DSI 复位后是空闲的（`MODE_CTRL` 为命令模式）。
2. 于是第一条 DCS 命令被迫走"视频→命令"切换，而**在这种残留状态下 `VM_DONE`（bit3）
   永远不置位**（1.5 秒轮询里一直是 0）⇒ 命令 -ETIME，随后 modeset 工作项被卡成 **D** 状态
   （`kworker/*+events` / `drm_client_dev_hotplug`，后面还压着一个 `__flush_work`）。
   `tasks` 两轮看到的 D 线程与锁链，根因就是这个"等不到的事件"。

**修复（本轮）**：不再等这个事件。

- 新增 `mtk_dsi_force_cmd_mode()`：`DSI_START=0` → `MODE_CTRL=命令模式` →
  清 `DSI_INTSTA` → 清陈旧 `irq_data` → 1~2 ms 稳定；
- `mtk_dsi_poweron()` 先把 LK 残留的 `MODE_CTRL/INTSTA/START` 记进无锁轨迹环，
  然后强制到已知状态 ⇒ 后续任何步骤都不再依赖 LK 留下的东西；
- transfer 路径与 `mtk_dsi_poweroff()` 改用它，去掉 500 ms 的 `VM_DONE` 等待；
- 轮询循环前后各加一条轨迹（若寄存器访问卡住，就会看到只有 "entering loop" 没有 "loop done"）。

### 第十二次上机：轨迹环打进 dmesg 后，**卡点精确到语句**

ringfix64 确认在跑 ✓（cmdline 里 `chopinport=ringfix64`），轨迹也进了 dmesg ✓：

```
mtkdisp 11.819791 dsi poweron: leftover MODE_CTRL=0x1 INTSTA=0x80      ← ★ LK 留下的是「视频模式」✓ 假设成立
mtkdisp 11.821801 dsi poweron: forced   MODE_CTRL=0x0 INTSTA=0x8000    ← 强制命令模式成功 ✓
mtkdisp 11.824024 k10a prepare: enter (prepared=0 reset_gpio=1)
mtkdisp 11.866028 k10a prepare: reset pulse done, pushing init table
mtkdisp 11.866029 k10a cmd 0 (0xff) sending
mtkdisp 11.866031 dsi entry:      MODE_CTRL=0x0 INTSTA=0x1080 INTEN=0xb  ← 已是命令模式（无需切换）✓
mtkdisp 11.866043 dsi lane ready: MODE_CTRL=0x0 INTSTA=0x1080 INTEN=0xb
mtkdisp 11.866054 dsi wait: enter: MODE_CTRL=0x0 INTSTA=0x80001080 INTEN=0xb
mtkdisp 13.888017 dsi wait: timed out: INTSTA=0x800...                  ← 2.02 s（CMD_DONE 从未置位）
mtkdisp 13.888028 dsi poll: entering loop (INTSTA=0x80005790)           ← ★ 之后什么都没有
```

结论：

1. **LK 残留状态被证实**（`leftover MODE_CTRL=0x1`），强制命令模式成功（`forced MODE_CTRL=0x0`）✓，
   所以"第一条命令前的模式切换"这条已经不再是问题 ✓；
2. `INTSTA` 在等待期间是 `0x80001080 → 0x80005790`：**bit1（CMD_DONE）始终为 0**，而
   bit4/7/8/9/10/12/14 等**错误位**陆续置起来 ⇒ 命令**发不出去/完不成**（DSI 处于错误态）；
3. `dsi poll: entering loop` 之后**完全没有输出** ⇒ 卡在轮询循环里 ⇒ 循环体只有
   `readl(DSI_INTSTA)`、`udelay(20)`、以及（status 非 0 时才走的）RACK 写 —— 而这条轨迹的
   *参数里*那次 `readl` 明明成功（打出了 0x80005790）⇒ **几微秒后同一个寄存器的访问就不再返回**：
   **DSI 寄存器窗口在命令发出后不久就"死"了**（时钟/电源被关，或块进入不可访问状态）。

`tasks` 仍是同一条 D 状态锁链（`drm_client_dev_hotplug` + `__flush_work`），`fbpaint` 仍无输出 ——
与"modeset 工作项卡死并握着 DRM 锁"完全一致。

### 第十三次：把 `INTSTA` 的位解出来 —— 命令**永远完不成**，而且视频引擎还在跑

厂商 `ddp_reg_dsi.h` 的 `DSI_INT_STATUS_REG` 位序（b0=RD_RDY, b1=**CMD_DONE**,
b2=TE_RDY, b3=VM_DONE, **b4=FRAME_DONE**, b5=VM_CMD_DONE, b6=SLEEPOUT_DONE,
**b7=TE_TIMEOUT**, b8/9/10=VM_VBP/VACT/VFP_STR）对上轨迹里的值：

| 时刻 | INTSTA | 解读 |
| --- | --- | --- |
| LK 残留 | `0x80` | 只有 b7（TE_TIMEOUT）|
| 强制后 | `0x8000` | 已清（b15 为只读位）|
| 命令发出前 | `0x1080` | b7 + b12 |
| 等待中 | `0x80005790` | **b4 FRAME_DONE + b7 TE_TIMEOUT + b8/b9/b10 VM_VBP/VACT/VFP_STR**，而 **b1 CMD_DONE 始终为 0** |

⇒ **视频引擎还在跑**（FRAME_DONE / VM_*_STR 事件持续产生），说明只"停流 + 写 MODE_CTRL"
并不能让它安静；控制器处于**混合状态**，命令队列永远完不成 ✓；之后**寄存器窗口连读都不返回**
（轨迹 `dsi poll: entering loop` 之后再无输出）。

**本轮修复**：在 `mtk_dsi_force_cmd_mode()` 里增加 **engine 复位**
（`DSI_CON_CTRL.DSI_RESET` 脉冲），顺序为 停流 → 复位 → 选命令模式 → 清 INTSTA → 稳定 2~3 ms。
标签升到 **ringfix65**。

另外记下一条**面板侧**的线索：`TE_TIMEOUT` 说明 DSI 在等面板的 **TE（撕裂效应）信号**；
厂商树里为此专门有 `dsi_te` 节点，本移植没有接。等命令通道通了再看是否需要接。

### 第十四次上机：★★ **DSI 命令通道通了**（engine 复位是那把钥匙）

`ringfix65` 的 engine 复位之后，`dmesg xfer` 第一次出现了"命令完成"：

```
xfer[3] wait: enter:    MODE_CTRL=0x0 INTSTA=0x80000000 INTEN=0xb
xfer[4] wait: timed out: MODE_CTRL=0x0 INTSTA=0x2      ← ★ CMD_DONE (bit1) 锁存了
xfer[5] poll: flag found: MODE_CTRL=0x0 INTSTA=0x0     ← ★ 轮询把它捞出来
xfer[6] send cmd ok                                    ← ★ 第一条命令成功！
xfer[7] entry: …（第二条命令开始）
```

同时 `clk` 证明**时钟从来不是问题**：

```
mm_dsi0_mm_clk      1 1 0 546000000 ... Y  1400e000.dsi
mm_dsi0_intf_clk    1 1 0 546000000 ... Y  1400e000.dsi
mipi_tx0_pll        1 1 1 834000000 ... ?  1400e000.dsi   ← ★ 正是厂商的 834 Mbps
mm_disp_dsc_wrap    0 0 0 546000000 ... Y              ← DSC wrap 时钟存在（未开）
```

`tasks` 也变了：`kworker/u32:4+events_unbound` 现在是 **S** 状态、wchan 为
**`mtk_dsi_host_transfer`** —— 不再是 D 卡死，而是**正在一条一条地发命令** ✓。

**唯一剩下的代价**：中断始终不投递（INTEN 使能了、INTSTA 锁存了、线也注册了，但计数恒 0），
所以每条命令要**等满 2 s 超时**才落到轮询 ⇒ 53 条初始化命令要好几分钟，
这期间 modeset 工作项握着 DRM 锁、`fbpaint` 被堵 ⇒ 看起来像"卡死"。

**ringfix66 修复**：`mtk_dsi_wait_for_irq_done()` **改为先轮询**（50~100 µs 一步），
waitqueue/interrupt 只作为后备 ⇒ 每条命令从 ~2.4 s 降到毫秒级，53 条命令秒级完成 ✓。

### 下一位上机者要做的：刷新 ringfix63，只做短输出

```bash
fastboot flash boot_a port/dist/chopin-ringfix88-livemirror-stockdt.img
# sha256 3b1132038717390f118e063a79dfbdab9bce1554fe2bee482f42c574eb207b5e
```

> ⚠️ **不要在 `/sys` 下用 `ls`**（我们的 `ls`/`lsr` 递归，会灌爆 USB CDC 并让
> 后续输入被拼坏）。本版已去掉 `drm.debug=0x06`、把 printk 环提到 4 MiB
> （`log_buf_len=4194304`），并在 DSI 启动时打一行寄存器日志。

**① 谁在按帧中断**（每条都只有 1～3 行，名字完整）：

```bash
chopin# grep dsi   /proc/interrupts
chopin# grep mutex /proc/interrupts
chopin# grep rdma  /proc/interrupts
chopin# grep ovl   /proc/interrupts
chopin# grep dsi0  /sys/kernel/debug/mediatek-drm/disp-regs
chopin# grep mmsys /sys/kernel/debug/mediatek-drm/disp-regs
chopin# dmesg readback
chopin# dmesg enabled
chopin# dmesg mmsys     # ★ 本版新增：路由寄存器读回（我手写的那些）
chopin# dmesg mutex     # ★ 本版新增：mutex EN/MOD/SOF
chopin# dmesg ovl       # ★ 本版新增
chopin# dmesg rdma      # ★ 本版新增
chopin# dmesg register  # ★ 本版新增：debugfs 单文件建成/失败
```

`dmesg enabled` 应出现（新加的一行）：

```
mtk-dsi ...: enabled: START=0x1 MODE_CTRL=0x1 PSCTRL=0x... SIZE_CON=0x9600438 VSA=10 VBP=10 VFP=54 VACT=2400 HSA=... HBP=... HFP=...
```

判读：

| 结果 | 结论 |
| --- | --- |
| `START=0x1` 且 `grep dsi` 计数在涨 | DSI 真在发 ⇒ 问题在面板侧 ⇒ **做 DSC** |
| `START=0x0` | DSI 没启动（`mtk_output_dsi_enable` 没跑到 / 被关掉）|
| `PSCTRL`/`SIZE_CON` 与 1080x2400 24bpp 不符 | 我们的 PS/SIZE 编程有问题 |
| `grep rdma` 恒 0 而 `grep ovl` 在涨 | RDMA 没在取数据（SMI/地址/使能问题），不是面板 |
| `mmsys +0xf54/0xf80/0xf50` 不是 `1/0/1` | 我的路由表在真机没生效 |

**② A/B：把面板的 DSC 解码器打开**（不用重刷；`dpms` 只允许 DRM master 写）

```bash
chopin# write /sys/module/panel_k10a_36_02_0a/parameters/force_dsc_decoder 1
chopin# write /sys/bus/platform/drivers/mediatek-drm/unbind mediatek-drm.7.auto
chopin# write /sys/bus/platform/drivers/mediatek-drm/bind   mediatek-drm.7.auto
```

花屏/噪点 ⇒ 链路 OK 只差 DSC；仍全黑 ⇒ 按①定位 DSI/路由，别急着写 DSC。

**③ 人眼**：屏幕是"完全漆黑"还是"有灰光的黑"？（背光走 LM36273，本移植没驱动它。）

**⑤ 本版新增：DSI 中断缺失时自动改用轮询 INTSTA**（既是取证也是绕行）

`mtk_dsi_wait_for_irq_done()` 超时后不再直接报错，而是轮询 `DSI_INTSTA`
（硬件锁存的标志与中断无关）最多 200 ms，用与中断处理相同的应答序列清除。
于是：

| 结果 | 结论 |
| --- | --- |
| `dmesg polling` 出现 `irq 0x.. was never delivered, found in INTSTA by polling` 且面板 `initialised, 53 commands sent` | **控制器/PHY 都是好的，坏的是中断投递** ⇒ 去查 GIC 线/显示中断 mux；同时面板已初始化，屏幕可能就有图了 |
| 轮询也超时（仍是 `Wait DSI IRQ ... Timeout`）| 看这一行里的寄存器：`INTSTA` 有标志但 `INTEN` 没有 ⇒ 控制器里被屏蔽；两者都没有 ⇒ 控制器根本没跑命令（时钟/电源/PHY）；都有却没中断 ⇒ GIC 线/路由 |

**④ 如果又卡死了：重启后读上一轮的日志**（这是关键步骤，卡死前的一切都在里面）

```bash
chopin# dmesg irqwatch     # ★ 风暴是哪个中断 + 首帧记录下来的显示中断行
chopin# dtprop /sys/firmware/devicetree/base/dsi@1400e000/interrupts   # ★ 期望 00 00 01 06 00 00 00 04（GIC_SPI 262）
chopin# clk mm_dsi          # ★★ 立即执行（不用重刷）：DSI 时钟开关与频率
chopin# dmesg mtkdisp       # 轨迹：应该很快就走到 push_table returned 0
chopin# dmesg panel        # ★★ 期待：panel initialised, 53 commands sent
chopin# dmesg readback     # ★★ 期待：panel readback 0x0a = 0x9c
chopin# fbpaint            # ★★ 彩条？
chopin# clk mm_disp         # ★★ 同上，显示主时钟
chopin# clk mipi            # ★★ MIPI TX PLL
chopin# trace              # 无锁轨迹环（也走 dmesg：dmesg mtkdisp）
chopin# dmesg panel        # 面板初始化到第几条命令
chopin# tasks              # ★ 哪个线程是 D、睡在哪个内核函数
chopin# dmesg xfer         # printk 轨迹（可能为空，属正常）
chopin# dmesg polling      # ★ 是否靠轮询才拿到 DSI 事件
chopin# dmesg RACK         # ★ 握手是否仍然卡住（新增）
chopin# dmesg panel        # 面板初始化序列走完了吗（LPM 修复的效果）
chopin# dmesg ready        # DSI 的 irq 号
chopin# dmesg mmsys        # 路由读回
```

判读：`irqwatch` 里增量最大的那个设备名就是风暴源（若持续是 `14001000.disp_ovl`，
说明是 LK 留下的 OVL0_2L 在按帧触发而我们没接管它；若是 `1400e000.dsi`，
说明 DSI 的状态位没被清掉）。最后一条 `alive` 的时间戳 = 卡死时刻。

---

## 一、先明确一件事：DSC 还没有做

chopin 的面板（`k10a_36_02_0a`）是 **DSC 视频面板**，厂商在 60/90/120 Hz 下都用
8 bpp 压缩（2 个 540 像素 slice）。主线的 mediatek-drm **完全没有 DSC 编码器**
（`mtk_dpp_comp.c` 里的 `mtk_dsc_config()` 只写 bypass 位，且只认 mt8195），
面板侧也没有 PPS 编程能力。所以本轮的策略是：

**先用「不压缩」的 30 Hz 模式把 DDP + DSI + 面板这条链走通，再谈 DSC。**

| | 厂商 60 Hz | 本轮首选（不压缩） | 本轮备选（不压缩） |
| --- | --- | --- | --- |
| 像素时钟 | 316325 kHz（厂商自定义值，≠ htotal×vtotal×fps）| **139000 kHz** = 1582×2474×35.5 | 117416 kHz = 1582×2474×30 |
| 每 lane 速率 | 834 Mbps（DSC 8bpp）| **834 Mbps** = 139000k × 24 / 4 | 704.5 Mbps |
| 刷新率 | 60 Hz | **35.5 Hz** | 30.0 Hz |
| 面板 DSC 解码器 | 开（`0xC0=0x03`）| 关（`0xC0=0x00`）| 关 |

**为什么首选 834 Mbps 而不是更低的 704**：两者都是不压缩，但首选模式把 DSI
链路速率**钉在厂商原样的 834 Mbps/lane** 上，于是「压缩 / 不压缩」成为与厂商
配置唯一的差异。若用 704 Mbps，一旦黑屏就同时有两个变量（速率 + 压缩）无法区分；
面板 DDI 的 PLL 若只认它标称的速率附近，704 还可能干脆锁不上。
两个模式都挂在 connector 上，fbdev 用带 `PREFERRED` 的那个（即 834 Mbps）。
`chopin# drm` 会列出 connector 的 modes 与 `/sys/kernel/debug/dri/0/state` 里
实际选中的 timing（clock 不同可区分）。

704 Mbps 低于面板标称的 834 Mbps，也低于 DSI PHY 的上限；面板初始化序列里
本来就写着 `/* DSC:03  NO DSC:00 */`，说明非压缩是面板支持的配置。**但这一点没有
先例可查**（全网没有 k10a 面板在主线跑通的记录），所以本轮最大的不确定性就是
「面板接不接受不压缩的视频流」。不接受的表现是：内核一切正常（DRM 起来、
connector enabled、DSI 在发数据），但**屏幕一直是黑的/花的**。

若如此，下一轮就要做 DSC：backport 上游 v9 系列（`mtk_disp_dsc.c` + DSI 侧 DSC）
并补 MT6893 的 PSC/DSC 寄存器与面板 PPS。

---

## 二、刷机

```bash
# 本轮镜像（DRM）
fastboot flash boot_a port/dist/chopin-ringfix88-livemirror-stockdt.img
#  sha256 3b1132038717390f118e063a79dfbdab9bce1554fe2bee482f42c574eb207b5e

# 回退（上一轮实测良品，无 DRM）
fastboot flash boot_a port/dist/chopin-ringfix61-host-settle-stockdt.img
#  sha256 84dd3a3ef88aa089da67f568fa62aad315ab4cafa1d4098d1fc3d6f0e4a3fdb0
```

刷完 `fastboot reboot`。**先插好 USB 线**，登录命令与以前一样：

```bash
sudo minicom -D /dev/ttyACM0 -b 115200
chopin# drm          # ← 本轮新增：一条命令给出显示状态 + 相关内核日志
```

镜像里已经把 `drm.debug=0x06`（DRIVER + KMS）写进内嵌 cmdline，所以
modeset/bridge 的每一步都会打到内核日志里；`chopin# drm` 会把相关行摘出来。

**为什么只能看日志、不能看寄存器**：arm64 上 `devmem_is_allowed()` 恒为 0，
`/dev/mem` 读 MMIO 必然 `-EFAULT`，所以用户态拿不到寄存器。本命令改用三个
间接但同样硬的证据：

| 证据 | 含义 |
| --- | --- |
| `clk_summary` 里 `mm_disp_*` / `mm_dsi0_*` / `mipi_tx0_pll` 的 enable 计数 | CRTC 到底有没有被 enable（时钟真的开了） |
| `/proc/interrupts` 里 disp/dsi/mutex 的计数 | **计数在涨** = 硬件真的在扫描输出；恒为 0 = 只配置没跑起来 |
| `/sys/kernel/debug/dri/0/state`（DRM 自带）| 原子状态：CRTC active、当前 mode、plane 的 framebuffer |

`chopin# irq` 和 `chopin# clk` 可以单独再看一遍这两项（连按两次 `irq` 比较计数增长）。

---

## 三、判读

### 3.1 好结果：屏幕亮起来（不管帧率）

`chopin# drm` 会给出：

- `/sys/class/drm/card0-DSI-1/status` = `connected`
- `/sys/class/drm/card0-DSI-1/enabled` = `enabled`
- `/sys/class/drm/card0-DSI-1/modes` = `1080x2400`
- `/proc/fb` 里出现 `0 mediatek-drmdrmfb`（简单 framebuffer 被 DRM 顶掉了）
- 日志里有 `mediatek-drm ... Adding component match for /disp_ovl@14000000` 等，
  以及 `mtk-dsi`/`panel-k10a` 的成功 probe

同时面板上应该出现 **fbcon 的文字**（因为 DRM 接管后 fbcon 画在 DRM 的 framebuffer 上，
而不再是 LK 那块 0x78e0c000）。哪怕只有 30 Hz、哪怕有轻微闪烁/撕裂，也说明
**路由 + DSI + 面板时序这条链是通的**，本轮目标达成。

### 3.2 黑屏但要分清「哪一步没通」

`chopin# drm` 的输出按下面的顺序看：

| 现象 | 结论 | 下一步 |
| --- | --- | --- |
| 连 `/sys/class/drm/card0*` 都没有 | `mediatek-drm` 没 bind：某个组件没起（时钟/PHY/DSI/panel）| 看 `drm` 命令里 "mediatek-drm / mtk-mmsys / mediatek-mipi-tx" 的 bound 情况与 dmesg |
| 有 card0，但 connector 是 `unknown`/无 modes | DSI 或 panel 没 attach 上 | 看 `mtk-dsi`、`panel-k10a` 的 probe 日志（典型：`-EPROBE_DEFER`、PLL `clk_set_rate` 失败）|
| connector `connected` + `enabled`，但屏幕黑 | **最可能：面板不接受不压缩流** | 记录 `drm` 全部输出，进入 DSC 阶段 |
| 花屏/条纹/局部图像 | 路由或像素格式不对 | 记录现象（哪一半、什么颜色），比对 OVL/RDMA 配置 |
| 内核 panic / 看门狗复位 | 寄存器写坏或时钟没起 | 用 `port/tools/read-boot-log.sh` 取 expdb；回退 ringfix61 |

### 3.3 已经换掉的「屏幕日志通道」

以前屏幕的可见内容是 logstore 驱动把 fbcon 的缓冲（`0x78e0c000`）镜像到面板正在扫的
那一块。**DRM 一 bind，simplefb 会被 DRM 的 aperture 逻辑摘掉**，`0x78e0c000` 那块就
不再更新，屏幕可能停在最后一帧旧内容上——这是预期行为，不是故障。
所以本轮的现场通道以 **USB gadget 串口** 为准（它不依赖显示）。

### 3.4 卡死时的自恢复

如果显示代码在 **initramfs 起来之前** 就把内核挂住（initcall 阶段），USB 串口
不会出现，现场通道就没有了。此时：

- **看门狗会自动复位**（`CONFIG_MEDIATEK_WATCHDOG=y`；initramfs 起来后才开始喂狗，
  所以「卡在 initcall」这种状态没人喂，约 30 秒后硬复位）；
- 复位后 LK 的 `fastboot` 仍然可用（在内核之前），所以**不必依赖串口也能重刷**：
  按住音量下 + 电源进 fastboot，刷回 `ringfix61` 即可。

这就是为什么本轮的「回退良品」镜像必须放在手边。

---

## 四、本轮改了哪些东西

内核（`git/chopin-linux-7.2.8`，提交 `99df977`）：

| 文件 | 内容 |
| --- | --- |
| `drivers/soc/mediatek/mt6893-mmsys.h` | **新增**：MT6893/MT6885 MMSYS 路由表（15 条 `MMSYS_ROUTE`），含 TOVL0、RDMA0/RSZ0 三级 mux，以及把 DSI0 输入从 DSC wrap 切回来的写 |
| `drivers/soc/mediatek/mtk-mmsys.c` | `mediatek,mt6893-mmsys` + `clk-mt6893-mm`；**这是 clk-mt6893-mm 能被实例化的唯一途径** |
| `drivers/soc/mediatek/mtk-mutex.c` | MT6893 显示 mutex（mt6885 位分配：DITHER0=8、DSI0=9、POSTMASK0=25）+ mt8183 的 SOF/EOF 表 |
| `drivers/gpu/drm/mediatek/mtk_drm_drv.c` | mt6893 主路径 `OVL0→RDMA0→COLOR0→CCORR→AAL0→GAMMA→POSTMASK0→DITHER0→DSI0` |
| `drivers/gpu/drm/mediatek/mtk_dsi.c` | `mediatek,mt6893-dsi`（mt8183 寄存器布局，但不写 shadow 位）|
| `drivers/gpu/drm/panel/panel-k10a-36-02-0a.c` | **新增**：面板驱动（复位时序、50 条初始化命令、30 Hz 不压缩模式）|
| `arch/arm64/kernel/chopin-board.dts` | 显示节点改用主线 compatible 与主线时钟 ID、MIPI PHY 用 mt8183 路径、新增 DSI port@1 与 panel 节点 |
| `arch/arm64/configs/chopin_defconfig` | `DRM/DRM_MEDIATEK/MTK_MMSYS/PHY_MTK_MIPI_DSI/PANEL_K10A` 全部 `=y`；**`MTK_CMDQ` 关掉**，否则 `MTK_MMSYS` 只能到 `=m`（initramfs 不加载模块）|

移植工程（`port/`）：`initramfs/init.c` 增加 `drm` 命令；`dist/` 增加本镜像。

**没有做的事**（有意为之，见下轮计划）：DSC 编码器、LM36273/WL2866D 电源与背光驱动、
IOMMU/SMI 挂载（显示走物理地址，和 LK 一样）、双 pipe、DP/HDMI。

---

## 五、已知风险（按可能性排序）

1. **面板不接受不压缩流**（最高）。面板是 DSC 面板，且厂商三档刷新率都用 DSC。
   缓解：下一轮做 DSC；本轮若黑屏但 DRM 状态全绿，基本就是这条。
2. **DDP 子块寄存器差异**。OVL/RDMA/COLOR/CCORR/AAL/GAMMA 用的是 mt8192/mt8183/mt8173
   的驱动数据，MT6893 属于 mt6885 世代，`mt6885-disp-ovl` 等厂商节点确实存在且
   寄存器基址一致，但**逐寄存器是否完全一致未证实**。缓解：先点亮，再按现象修。
3. **PHY 上限**。厂商驱动把 MT6893 的 DSI PLL clamp 在 1250 Mbps/lane，主线 mt8183
   数据 clamp 在 1600。本轮 704 Mbps 两边都满足，所以不影响本轮；但它决定
   「能不能不做 DSC 跑 50 Hz（1174 Mbps）」。
4. **`iommus` 残留**。板级树里显示节点仍指向厂商 IOMMU 节点（无主线驱动），
   DMA 会退回物理地址。LK 自己就是 `m4u off` 跑的（日志里有 `m4u off`），
   与之一致；但若 LK 的 SMI 仍开着地址转换，显示会取错地址。缓解：现象会是
   「DSI 在发数据、屏幕乱」；可在板级树里把厂商 m4u 节点 `status = "disabled"`。
5. **背光/电源**。DVDD（WL2866D）与 AVDD/AVEE（LM36273）本轮完全不碰，
   沿用 LK 的状态（LK 已点亮面板）；`regulator_ignore_unused`/`clk_ignore_unused`/
   `pd_ignore_unused` 在 cmdline 里，无人会去关它们。若屏幕亮但无背光，
   就需要移植 LM36273（可复用主线 LM36274 的 ti-lmu/lm363x 驱动）。

---

## 六、复现构建

```bash
cd linux-7.2.8
make O=../build/chopin ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- chopin_defconfig
make O=../build/chopin ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j16 Image
cd .. && ./port/tools/mkinitramfs.sh && ./port/tools/build-images.sh
cp port/out/boot-stockdt.img port/dist/<新名字>.img
```

自检：

```bash
port/tools/verify-drm.sh            # ★ 一条命令跑完下面全部本地校验
```

`verify-drm.sh` 覆盖：defconfig/`.config` 的 `=y` 符号与 `MTK_CMDQ=n`、
内嵌 cmdline 的 `ringfix62` 标签、板级 DTB 里的显示节点与 DSI↔面板 graph 连线、
路由表回归、面板序列回归、仓库一致性（`audit-repo.py`）、boot.img 四个分区与
内嵌 board DTB 的逐项比对、dist 镜像与 `SHA256SUMS` 的一致性。

也可以单独跑：

```bash
port/tools/audit-repo.py            # 应为 OLD COPY IN REPO: 0
port/tools/test-mmsys-routes.py     # 路由表回归（见下）
port/tools/test-panel-seq.py        # 面板初始化序列 vs 厂商驱动（见下）
dtc -I dtb -O dts build/chopin/arch/arm64/kernel/chopin-board.dtb | \
    grep -E 'mt6893-mmsys|mt6893-dsi|k10a-36-02-0a'
```

### 路由表回归测试

`tools/test-mmsys-routes.py` 直接解析 `mt6893-mmsys.h`，按主线
`mtk_mmsys_update_bits()` 的语义（`reg = (reg & ~mask) | (val & mask)`，
逐条按表顺序）在一个**以 LK 的 DSC 状态为初值**的寄存器模型上跑一遍
`mtk_crtc` 的连接序列，然后断言：

- DSI0 必须从 DSC wrap 切回 DITHER0（`0xf80: 1→0`、`0xf54: 3→1`）——
  这正是「mask 开窄了」会静默失败的地方（mask 写成 `0x1` 时 bit1 留 3）；
- `0xf50` 的 PQ0 支路被清掉、`0xf4c` 的 OVL0_2L 背景喂入被清掉；
- RDMA0→COLOR0 的三级 mux 全部到位；
- 表不碰的寄存器（`0xf88`/`0xff8`/`0xf68`）保持原值。

它还带一个自检：把 `0xf54` 的 mask 故意改成 `0x1`，测试必须报错
（否则测试本身没有意义）。当前结果：全部 PASS。

**它证明的是** mask/取值组合正确、不会互相踩；**不能证明**偏移正确
（偏移的证据在 vendor 源码，见 `audit/drm-mmsys-routes-zh.md`），
也不能证明硬件行为与文档一致。

### 面板序列回归测试

`tools/test-panel-seq.py` 直接解析厂商
`panel-k10a-36-02-0a-dsc-vdo.c` 与新驱动的表，逐条比对：

- init：**53 条对 53 条**，只允许两处 0xC0（DSC 03→00）；
- suspend：7 条对 7 条，只允许厂商多送的那个 0x00 字节（它的
  `push_table()` 对空参数表仍拷 `count` 字节）；
- 时序：htotal/vtotal 与厂商 60 Hz 档完全一致，且 `.clock` **必须**
  等于 `htotal × vtotal × fps`（厂商自己的 `.clock` 不是这个值，
  照抄会让主线按 24 bpp 请求 ~1.9 Gbps/lane）。

这个测试在本轮抓到了一个真实缺陷：`.clock` 曾写成 117398 kHz
（应为 117416），已修正。

本镜像已做过的打包自检（证明刷进去的确实是这次编的东西）：

| 检查 | 结果 |
| --- | --- |
| boot.img 里的 kernel 段 == `out/Image.gz` | ✅ |
| 该段 gunzip 后 == `build/chopin/.../Image` | ✅（42953216 字节） |
| boot.img 里的 ramdisk == `out/initramfs.cpio.gz` | ✅ |
| boot.img 尾部的 DTB == `out/stock-base.dtb`（LK 用）| ✅（201784 字节 @ 0xeb9000） |
| Image 内嵌 board DTB（`fdt-initrd.py` 改过 initrd 字段，故非逐字节相同） | ✅ 含 `disp_ovl@14000000` / `mediatek,mt6893-mmsys` / `k10a-36-02-0a` |
| 内嵌 cmdline | ✅ `chopinport=ringfix62 drm.debug=0x06` |

---

## 七、如果本轮点亮了，下轮做什么

1. **DSC**：把 DSI 速率与刷新率恢复到厂商值（60/90/120 Hz）。需要
   `drivers/gpu/drm/mediatek/mtk_disp_dsc.c`（可 backport 上游 v9 系列）、
   MT6893 的 DSC wrap 寄存器（0x14124000）、mmsys 的 DSC 插入路由
   （`0xF50` bit3、`0xFF8=1`、`0xF88=1`、`0xF80=1`、`0xF48=0`、`0xF54=3`）、
   面板 PPS 编程。**风险最高的是 RC 参数**：厂商驱动没给 `rc_buf_thresh` /
   `rc_range_params`，只有解码后的 PPS 字段。
2. **背光与电源**：LM36273（ti-lmu/LM36274 复用）+ WL2866D2，让面板能自己上电/休眠。
3. **IOMMU/SMI**：接主线 `mt6893-iommu-mm` + `mt6893-smi-larb`，把 `iommus` 变成真的。
4. **显示子系统的其它部分**：双 pipe（1080 宽拆 540+540，和 LK 一样）、DP/DP_INTF。
