# chopin 自动旋转 / SCP 传感器 bring-up

目标：让 Debian/GNOME 能自动旋转——需要 AP 侧拿到加速度计数据，经 IIO 交给
`iio-sensor-proxy`。

## 结论：加速度计在 SCP 后面，不在 AP 总线上

- `chopin_defconfig`：`CONFIG_MTK_ACCELHUB=y`、`CONFIG_NANOHUB=y`、
  `CONFIG_NANOHUB_MTK_IPI=y`
- `accelhub.c` 的数据来自 `SCP_sensorHub`（IPI），真正的传感器由 SCP 固件
  （`scp_a`/`scp_b` 分区）驱动
- Android 的 AP 设备树里只有 `scp@10700000`，没有任何 accel 节点

## 已确认：SCP 已经在跑（LK 启动）

devmem 实测（mainline 下、未加载任何 SCP 驱动）：

```
0x10730070 = 0x00000000   CORE0_STATUS：未 gated、未 halt
0x10500000 = 0x03c0006f   RISC-V "jal x0, +0x3c" —— SRAM 里有固件
0x10724000 = 0x00000200   SCP CFG
```

SCP region info（SRAM+0x4）：

| 字段 | 值 |
| --- | --- |
| ap_loader_start | `0x9fa00000`, size `0x2000` |
| ap_firmware_start | `0x9fa02000`, size `0x143c00` |
| ap_dram_start | `0x9fc00000`, size `0x5a880` |
| 保留区 | `0x9fa00000` + `0x500000`（DT `mblock-22-SCP-reserved`） |

## 传输栈（厂商实现，mainline 都没有）

```
sensor 事件  <->  SCP_nanoHub.c (sensorFIFO，44B 记录，1MB)
                   |
             scp_ipi_send(IPI_SENSOR)          <-- legacy mpool
                   |
             mtk_tinysys_ipi.c / mtk-mbox.c
                   |
             mbox 寄存器 (0x107fb000..0x107ff100)
```

- **mbox**：每个 mbox 0x100 字节的槽位窗口 + `set`(0x…100)/`clr`(0x…10c)/
  `init`(0x107a5020 起) 寄存器；`mtk_mbox_write_hd()` 把
  `struct mtk_ipi_msg_hd { id, len, options, reserved }` + 负载写进槽位，
  `mtk_mbox_trigger_irq()` 往 `set` 寄存器写 `1 << pin_index` 触发 SCP。
- **mbox 窗口**：mbox0 `0x107fb000`、mbox1 `0x107fc000`、mbox2 `0x107fd000`、
  mbox3 `0x107fe000`、mbox4 `0x107ff000`（来自 AP DT 的 scp 节点）。
- **legacy mpool**（IPI_SENSOR = 3 走这里）：消息是
  `{u32 id; u32 len; u8 data[]}`，收发 pin 是 `IPI_OUT_SCP_MPOOL_0`（mbox2）
  与 `IPI_IN_SCP_MPOOL_0`（mbox2）。
- **nanohub**：`IPI_SENSOR` 的 48 字节消息（4B 头 + 44B packet）；传感器
  数据由 SCP 写进 AP 预先放在保留内存里的 `struct sensorFIFO`
  （`wp`/`rp`/`FIFOSize`/`data[]`，记录 44 字节）。AP 先把 FIFO 物理地址
  用 `sensor_send_dram_info_to_hub()` 告诉 SCP，再发 config event 使能
  accel，然后轮询 FIFO。

## 分阶段计划

1. **mbox + legacy IPI 传输层**（mainline 驱动或先 userspace devmem 验证）：
   - 读 mbox 的 set/recv 寄存器，确认 SCP 是否已经在发消息（SCP_READY 等）；
   - 发一条 legacy mpool 消息，确认 SCP 能收到（mbox busy 位/回包）。
2. **nanohub 最小客户端**：
   - 在 `0x9fc60000` 附近（SCP 保留区、避开 firmware/ap_dram）放
     `sensorFIFO`，发 dram-info；
   - 发 config event 使能加速度计（`ConfigCmd{evtType=0x300, rate, sensorType, cmd=ENABLE}`）；
   - 轮询 FIFO，解出 44 字节记录（timestamp/type/x/y/z）。
3. **IIO + 用户态**：暴露 `iio:device`（加速度计），Debian 装
   `iio-sensor-proxy`，GNOME 自动旋转。

## mbox 读数的异常（待确认）

mainline 下读 5 个 mbox 的 `set`/`clr` 全是 0，但 **mbox2 base（0x107fd000）
读出来是 0x03C0006F、+4 是 0x9FA00000**——正好等于 SCP SRAM 的
`[0]`/`region_info.ap_loader_start`。也就是说 `0x107fd000` 这个窗口看起来
**镜像到了 0x10500000**，而不是真正的 mbox 槽位。

两种可能：

1. mainline 没有给 SCP 岛上电/开时钟（厂商是 `scp_mtcmos_ctrl` + SCP 时钟
   之后才访问 mbox），mbox 窗口在岛关闭时被总线路由成 SRAM 镜像 / 返回脏数据；
2. mbox 的寄存器其实不在 AP 视角的 0x107fb000（需要经 scp_cfgreg 或另一层
   地址映射），DT 里的地址只在厂商的地址翻译下成立。

下一步先做地址镜像验证，再决定是“先补 SCP 上电/时钟”还是“重新定位 mbox”。

## 已打通：SCP 启动序列（2026-10-24 实测）

之前判断“SCP 已经在跑”是错的：SRAM 里的固件是上次 Android 启动的残留，
mainline 下 SCP **时钟关着、复位没释放**，所以 mbox 窗口被总线路由成 SRAM 镜像
（读 0x107fd000 得到 0x03c0006f）。

按厂商 `scp_sys_reset_ws()` 的顺序用 devmem 实测成功的启动序列：

```sh
# 1) 开 SCP 核心时钟（CLK_SYS_EN | CLK_HIGH_EN）
devmem 0x10721004 32 0x3

# 2) 复位 core0/core1
devmem 0x10730004 32 0x1
devmem 0x10740004 32 0x1
sleep 0.2

# 3) DRAM 保留区（SCP 的 ap_dram）
devmem 0x10730044 32 0x9FA00000
devmem 0x10730048 32 0x00500000

# 4) CORE_REBOOT_OK + 释放复位
devmem 0x10730054 32 0x1
devmem 0x10730000 32 0x1
devmem 0x10740000 32 0x1
```

结果：

- `MON_PC` 从 0x2A0（loader）推进到 0xF76C，再重启后进入固件（0x1AD80 / 0x22088）；
- **mbox 窗口恢复正常**（0x107fd000 读 0x0，不再是 SRAM 镜像）；
- 之后 `CORE0_STATUS` 回到 `0x3`（gated+halt），PC 停在固件里：SCP 在 idle/WFI
  等 AP 的 IPI，五个 mbox 的 set 寄存器全 0（还没有 IPI 交互）。

关键寄存器（SCP 地址空间）：

| 名称 | 地址 |
| --- | --- |
| CLK_ENABLE / CLK_SW_SEL | 0x10721004 / 0x10721000 |
| CPU0/1 SRAM PD | 0x10721080 / 0x10721084 |
| CORE0/1 SW_RSTN CLR/SET | 0x10730000 / 0x10730004（core1: 0x10740000/4） |
| DRAM_RESV_ADDR/SIZE | 0x10730044 / 0x10730048 |
| CORE_REBOOT_OK | 0x10730054 |
| CORE0_STATUS / MON_PC | 0x10730070 / 0x10730080 |
| mbox0..4 base/set/clr | 0x107fb000…0x107ff10c |

## 2026-10-24 进展与卡点

已经做到：

- **SCP 能稳定启动**：时钟、复位、DRAM 窗口、`CORE_REBOOT_OK`、复位释放。
  驱动 `drivers/misc/mediatek/scp-chopin.c`（提交 `42d7cc385`）在 debugfs 里提供
  `poweron`/`restore`/`status`/`events`/`dump`。
- **卡死问题已解决**：`ipc0`/`ipc1` 是 SCP 的**看门狗中断**（电平触发），
  不处理会让 AP 在 spurious IRQ 里死循环。现在驱动会清 WDT 位并计数，系统稳定。
- SCP 启动后 `pc` 进入固件地址（0x1ad80/0x3ef9a），`wdt=0`，但**不主动发 IPI**。

**当前卡点：AP 侧 mbox 数据窗口不可用。**

- 官方 DT 给的 `mbox*_base`（0x107fb000…0x107ff000）在 AP 上读写都会落到
  **SCP TCM**：写 0x107fb000 会把 TCM[0] 改成写入值（实测 TCM[0] 从
  `0x03c0006f` 变成 `0x00000003`），读回的也是 TCM 内容。
- `set`/`clr`/`init` 寄存器（0x107fb100、0x107a5020…）看起来是真的
  （init 里是 SCP 侧 mbox 地址 0x7d600/0x7d700/…）。
- 因此 AP→SCP 的 IPI 发不出去，nanohub 的 "dram info"/配置消息也无法送达。

### 槽位实验（2026-10-24）

init 寄存器给出的 SCP 侧 mbox 地址是 `0x7D600 + 0x100*N`；对应 AP 侧
`0x10500000 + 0x7D600 = 0x1057D600` 起，实测该处全为 0（像是槽位区）。
往 `0x1057D800`（mbox2）写 dram-info 消息、ring `0x107fd100` 后：

- 槽位保留了我们写入的值；
- `0x107fd100` 的 set 位**未被清**；
- mbox0 无回包，`events` 为空，SCP 的 `wdt=0`、PC 稳定。

结论：消息没有被 SCP 消费。可能是 (a) 槽位不在这个地址（只是零填充的 TCM），
(b) SCP 固件的 mbox 服务没起来/没使能对应中断，(c) set 寄存器没有真正送到 SCP。

### 固件里的 mbox 描述表（scp_a 反汇编，文件偏移 0x97544）

```
mbox0: 0x0007d600  0x700fb104  0x700fb108  0x700a5020  ...
mbox1: 0x0007d700  0x700fc104  0x700fc108  0x700a5024  ...
```

- 槽位基址是 **SCP 地址 0x7d600 + 0x100*N**（对应 AP 的 `0x10500000+0x7d600`，
  与我们猜的 TCM 槽位一致）；
- SCP 侧的 set/clr/init 是 `0x700fb104 / 0x700fb108 / 0x700a5020` 这一组
  （0x70000000 是 SCP 视角的外设别名，AP 对应 0x107fb100 等）。

也就是说：**槽位内存是共享的（在 SCP TCM 里），AP 的 mbox base 窗口本应翻译到
TCM+0x7d600，但实际别名到了 TCM+0**——这是 AP 侧访问 mbox 的关键缺陷；
而 set 寄存器是否真的能触发 SCP 中断仍未验证（写 0x107fd100 后 SCP 没反应）。

可能的下一步（未验证）：

1. 找 AP 访问 SCP CFG/mbox 的**使能位**（可能在 SPM 的 SCP_PWR_CON 或
   infracfg_ao 的某个 secure 寄存器），或 LK 在启动 Android 时额外做的配置；
2. 移植厂商 `mtk-mbox`/`mtk_tinysys_ipi`，看它的 probe 是否隐含了必要的初始化
   （目前看只映射 + 请求 IRQ，没有额外写寄存器）；
3. 从 `scp_a` 分区里的固件/loader 反汇编，确认它是否在等某个 AP 侧握手
   （GPR/region info 的扩展字段）。

## 2026-10-24 晚：mbox 解码成功，RTOS 在等待一个内存标志

按 LK（`lk.img`，`platform/mt6893/mt_scp.c`）的寄存器表写入后：

```
0x10721000 <- 0x120   0x10724000 <- 0x170   0x10730000 <- 0x120
0x10732000 <- 0x260   0x10733000 <- 0x120   0x10740000 <- 0x120
0x10742000 <- 0x260   0x10743000 <- 0x120   0x10750000 <- 0x330
0x10751000 <- 0x10    0x10751400 <- 0x70    0x10752000 <- 0x340
0x107a5000 <- 0x110   0x10001b14 <- 0x10
```

**AP 侧的 mbox 窗口恢复正常**（0x107fb000/0x107fd000 读出 0，而不是 TCM 镜像）。
驱动 `scp_power_on()` 已内置这张表（提交 `32657e1fe`）。

同时为了不再卡死，驱动**不注册任何中断**（mbox 与 ipc0/ipc1 全部保持屏蔽），
改为 debugfs `poll` 轮询；`send` 只写 TCM 槽位（`0x1057D600 + 0x100*N`）。

固件反汇编显示 RTOS 启动后在 `0x3ef96` 处自旋：

```
0x3ef96: lr.w a0, (s0)
0x3ef9a: bnez a0, 0x3ef96     ; 等 *(s0) == 0
```

即它在等一个**共享/内存标志**被清零（不是崩溃，也不是 WFI）。

栈回溯（MON_SP=0xC1F68，MON_LR=0x3EF82）显示自旋函数的入口附近有
`lui a0, 0x25b` / `mv s0, a0`，即 s0 可能是 **0x25B000**（超出 DT 的 TCM
0x180000，说明 SCP 还有一段我们没映射的地址空间，或是反汇编未对齐造成的误读）。
栈上其他候选指针（0xC1000、0x9FFC4、0x22354）读出都是 0 或代码字节，不是标志。

下一步需要完整反汇编 RTOS 的启动路径（无符号表，工作量大），或找到 SCP 侧
0x25B000 对应的 AP 可见地址。

## 2026-10-24 下午：移植 MT6895 主线 SCP 栈（进行中）

按用户建议放弃逆向，改为移植 MT6895-Mainline 的成熟实现：

- `drivers/misc/mediatek/scp/rv/`（RV SCP 驱动）
- `drivers/soc/mediatek/mtk-mbox.c` + `mtk_tinysys_ipi.c`
- `drivers/rpmsg/mtk_rpmsg_mbox.c`
- `drivers/misc/mediatek/sensor/2.0/`（sensorhub + IIO）
- 缺头文件（emi.h/clk-fmeter.h/mtk-afe-external.h）与 AFE 信号量桩

**关键教训：内核用的是内嵌 DTB**（`arch/arm64/kernel/chopin-board.dtb`，由
`chopin-dtb.S` incbin），不是 boot 镜像里的 DTB。改完 DTS 必须：
`make dtbs` → 复制到 `build/.../arch/arm64/kernel/chopin-board.dtb` → 重新 `make Image`。
刷 `boot-stockdt.img`（LK 用原厂 DTB，避免 LK 崩）。

scp 节点补齐的属性（全部来自 MT6895 的 DT）：

- `core_nums = <1>; twohart = <1>; mbox_count = <5>;`
- `send_table` / `recv_table`（含 sensor hub 的 29/30/31/32）
- `scp_mem_key = "mediatek,reserve-memory-scp_share"`
- `scp_mem_tbl`（vow/sensor main/logger/audio/... 子块）
- `scp_feature_tbl`、`memorydump`
- `scp_dvfs` 节点关闭（缺 vlp/clk regmap，开只会 WARN + 拖 3s）

结果：

- 驱动 probe 完整通过，mbox/IPI 注册 43 个 IPI，sensorhub 注册出 **4 个 IIO 设备**
  （`xaga-als`/`xaga-accel` 名字待改）；
- `core_0="enable"` 自动启动会卡死重启（厂商 `scp_start` 路径）；
- 手动启动（LK 表 + 时钟 + 复位 + DRAM 窗口 + 释放）SCP 能跑起来，但 PC 停在
  `0x3EF9A` 的 `lr.w/bnez` 自旋（s0 疑似 0x25B000），打补丁跳过会总线挂死；
  GPR0、L1C flush、DRAM 标志清零都试过，均未解除。

## 2026-10-24/25：SCP 驱动栈移植完成，卡在固件互斥锁

**已完成**

- 移植 MT6895 的 rv SCP 驱动 + mtk-mbox + tinysys-IPI + rpmsg + sensorhub
  （提交 `2d9208892`），驱动 probe 通过，sensorhub 注册出 4 个 IIO 设备；
- scp 节点补齐所有 DT 表（core_nums/twohart/mbox_count/send_table/recv_table/
  scp_mem_key/scp_mem_tbl/scp_feature_tbl/memorydump），scp_dvfs 关闭；
- mbox 中断里 `BUG_ON(null pin_buf)` 改为“应答 + 丢弃”（否则 SCP 一启动就 panic/重启）；
- 新增 `scp-chopin` 辅助驱动（`/sys/kernel/debug/scp-chopin/`）：
  `status`、`cleartcm`、`loadloader`、`reginfo`、`dramrestore`、`dramzero`、
  `fullreset`、`fullreset_zero`、`start`（手动完整复位 + 启动，已跑通、不卡死）。

**关键教训**

- 内核用的是**内嵌 DTB**（`arch/arm64/kernel/chopin-board.dtb`，由
  `chopin-dtb.S` incbin）。改 DTS 后必须重新 `make dtbs`、把生成的 dtb 复制到
  `build/.../arch/arm64/kernel/chopin-board.dtb`，再 `make Image`；刷
  `boot-stockdt.img`（LK 用原厂 DTB，避免 LK 崩）。
- 厂商驱动的自动启动（`core_0=enable`）会静默卡死（likely 总线挂死），暂时只能用
  `core_0=disable` + 手动启动。
- 手动启动后 SCP 进入固件，但停在 **0x3EF96/0x3EF9A** 的循环。按编码解析为：
  `lr.w t0,(tp)` / `bnez t0` / `li a0,-1` / `sc.w t0,a0,(tp)` / `bnez t0` / `fence`
  —— 即对一个由 **tp(x4)** 指向的地址做 test-and-set（写入 -1）。
- 尝试过：TCM 全清、loader 重拷、region info 重写、DRAM 备份恢复/清零、只释放
  core0、L1C flush、改 GPR0、补丁捕获 s0/tp —— 均未解除；固件补丁在 2 字节对齐
  地址上的 32 位指令可能与实际指令流不对齐（PC 常变为 0），需要更准确的固件 ISA/对齐分析。

**当前可用镜像**：`out-scp-current/boot-stockdt.img`
（md5 `f840a84469ea9dc126c3d53d2b8cc9e0`，稳定启动、IIO 设备存在、SCP 手动可启动）。

**2026-10-25 追加发现**：固件补丁（把 `0x3EF96` 的 `lr.w` 换成 `sw`）捕获到
**tp(x4) = 0**（TCM[0] 被写 0），而同一时刻 s0 = 0x7D5E0。即自旋循环
`lr.w t0,(tp)` 是在 **tp=0** 的情况下读 TCM[0]（loader 的跳转字 0x03c0006f，恒非 0），
永远等不到 0——这不是正常的互斥锁，而是**线程指针 tp 未初始化**导致的空指针循环。
换句话说 RTOS 的线程/TLS 初始化没有完成。下一步应查：RTOS 早期初始化在等什么
（很可能是 SCP 启动参数 `ap_params_start` / `SCP_DRV_PARAMS_MEM_ID` 区域，
vendor 的 `params_to_scp()` 在我们这里因 `CFG_SENSOR_PARAMS_TO_SCP_SUPPORT`
未打开而是空实现）。

时钟线已排除：把 SCP mux 从 26M 切到 univpll_d5 后行为不变（重启后回到 0x3ef9a）。

**2026-10-25 结论**：dump `TaskContext_ptr`（TCM 0x7D214，AP 0x1057D214）连续 16 字
**全为 0** —— RTOS 的任务上下文根本没有被初始化，和 tp=0 相互印证：固件卡在
“任务/TLS 尚未建立”的极早期，自旋是空指针等待而不是真正的互斥锁。

已排除的因素：时钟（26M vs univpll_d5 行为不变）、`scpctl` 字段、GPR0、
L1C flush、DRAM 备份恢复/清零、只释放 core0、DRAM 参数区（ap_params_start 为 0）。

**行为不稳定**：同样的 `fullreset + start`，有时 SCP 停在 0x3ef9a（自旋），有时会让
AP 直接重启——说明固件早期初始化处在竞态/总线访问敏感阶段，继续盲试风险高。

另外查了 SCP 的 logger 共享区（0x8F1CA700，SCP_A_LOGGER_MEM_ID），头部
`base/size/enable/...` 全是 **0xFFFFFFFF**（从未初始化）——说明固件还没走到 logger
初始化，同样印证“卡在极早期”。

要继续只能是：完整的固件/RTOS 逆向（需要更准确的 ISA 分析），或原厂 v02 驱动 +
PMIC/DVFSRC 的整套移植。当前稳定镜像仍为 `out-scp-current`。

## 2026-10-25 死锁定位（重要）

用正确的 ISA 反汇编（`llvm-objdump --mattr=+m,+a,+c,+zicsr`）后确认：

- 固件在 `0x3EF88` 的“原子引用计数 + 自旋锁”函数里死锁：
  `amoadd.w` 对 0xAD2E8 计数，`lr.w/sc.w` 锁 **s0**（调用者传入的对象）；
- 启动后 **core0 PC=0x3EF96、core1 PC=0x3EF9A，两个 hart 都在这把锁的循环里**；
- `TCM[0x7D5E0] = 0xFFFFFFFF`（已被 `sc.w` 置为持有）→ 持有者又重入同一把
  非递归锁，自死锁；该地址紧挨 mbox 槽位区（0x7D600），是 mbox 初始化对象的一部分；
- **把锁清 0 → 固件继续跑，但随即 AP 重启**；initramfs 恢复的日志在 3.39s 之后
  没有任何内核消息（解锁后是**静默挂死**，多半是 SCP 继续运行后把 AP 总线锁死，
  看门狗复位），因此拿不到 panic 栈。

## 待办

- [ ] 读 5 个 mbox 的 set/clr 寄存器，确认 SCP→AP 有待处理消息
- [ ] 实现最小 mbox 写/触发（先 userspace devmem 脚本，再内核驱动）
- [ ] 从厂商 `SCP_nanoHub.c` 抄出 dram-info 与 config event 的精确字节
- [ ] FIFO 记录格式（`struct data_unit_t`，44 字节）解码
