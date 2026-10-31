# Chopin 自动旋转 / 传感器：当前状态交接（2026-10-25 第二轮）

新会话从这里开始。目标：让 chopin 的加速度计（自动旋转）和环境光（自动亮度）
可用，数据来自 **SCP sensor hub**。

上一轮（同一天早些时候）的结论是"固件在启动早期自旋死锁，卡点在固件侧"。
这一轮把厂商内核的启动代码逐行对上以后，**结论要改**：卡点在我们自己写的
AP 侧启动代码里，有两个明确的 bug，都能解释"固件在第一次打印时就停住"。

---

## 0. 一句话结论

我们以前用 helper（`drivers/misc/mediatek/scp-chopin.c`）手动启动 SCP 时，
**给固件的 `DRAM_RESV_ADDR` 写的是 0x9fa00000（scp.img 在 DRAM 里的位置），
而厂商写的是 0x8f000000（`reserve-memory-scp_share`，运行时 AP↔SCP 共享窗口）**；
同时**固件要的 DRAM 日志控制块（share + 0x1ca700）从来没被初始化过**（上一轮
dump 出来的"全 0xff"就是这个）。固件的第一次 printk 走日志路径，于是停在那把
log 互斥量上，两核都卡住。

本轮已改：

1. helper 的启动序列改成厂商 `scp_sys_reset_ws()` / `reset_scp()` 的忠实复刻
   （正确的共享窗口、SRAM 上电、只放 core0、整段序列一条命令）；
2. **默认改回厂商驱动启动**：`mediatek,scp` 节点 `core_0="enable"`，新增
   `mediatek,no-auto-start` —— 驱动在 boot 时把 region info、reserve memory、
   **logger 控制块**、recovery 映射全部准备好，核留在复位状态，由你在串口里
   一条命令放行。这样 SCP 再卡死也只是一次重启，不会拖死启动。
3. 新增只读诊断 `cat /sys/kernel/debug/scp-chopin/audit` 和 board 脚本
   `/usr/bin/board-scp-audit.sh`（已打进新镜像）。

还有一个必须一起解决的疑问：**LK 很可能本来就把 SCP 启动了**
（`SENSOR-HANDOFF.md` 里实测：未加载任何 SCP 驱动时 `0x10730070=0`（core0 未
gated/halt）、`0x10500000=0x03c0006f`（TCM 里是 loader 的第一条指令））。
厂商内核启动时做的事其实很轻：`reset_scp()` 只写共享窗口 + 放 core0，
**不清 TCM**。而我们以前每次"启动"都是先 assert 复位、清 TCM、重载固件
——那是**崩溃恢复**路径，对一个本来活着的 SCP 做冷重启。所以本轮把
"轻启动"（= 厂商 boot 行为）和"重启动"（= 恢复路径）分开成两条命令，
先走轻的。

镜像：`out-scp-v7/boot-stockdt.img`（md5 `d04cba977a8f111c9d17f05e005d0d89`）；
`out-scp-v6` 是拿到 `SCP A is ready` 的那一版（md5 `53cc7d0110bf781e0bf9ce91d8cc7790`）。

**下一步就一件事**：刷这个镜像，按 §5 跑一遍，把 4 个数字贴回来
（`status` 的 `pc0`、`devmem 0x107a5060`、`audit` 里的 `dr[1ca700]`、
`tcm[07d5e0]`）。

---

## 0.1 实测结果（2026-10-25 09:15，本节的结论已经变了三次，以这段为准）

**`SCP A is ready` 已经拿到。** 从 `SENSOR-HANDOFF.md` 的"固件死了"到今天的
"固件活着、READY 到达"，中间真正的 blocker 不是固件，也不是共享窗口，而是
**mbox 通道表（pin 布局）**：

- mbox 的"pin"不是随便分配的号：`mbox_setup_pin_table()` 对每个 mbox 先排
  **send** 再排 **recv**，把每条 pin 的 `msg_size` **累加**起来，累加值同时当
  slot offset 和中断位（is64d 下按半字算）。所以 DT 里的
  `send_table`/`recv_table` 必须和固件编译时那张表**逐项、逐 mbox、逐 size** 一致。
- 我们原来的表是从 **MT6895** 移植的。固件发 READY 在 **mbox3 bit15**
  （= 厂商布局的 `IPI_IN_SCP_READY_1`, id 22），我们的表把这个 id 放在 mbox1
  pin12，于是 `dmesg` 里只有一行 `[MBOX ISR] mbox=3 unclaimed irq 8000`。
- 改成厂商 `mt6885/scp_ipi_table.h` + `scp_mbox_layout.h` 的布局后（v6），
  同一个 bit 命中 id22 → `scp_ready=1` → `[SCP] recovery success` →
  `SCP A is ready`。

实测得到的、已验证的 pin 表（`dmesg | grep "recv pin:"` 可以复核）：

| id | mbox | pin | 说明 |
| --- | --- | --- | --- |
| 9 | 1 | 20 | `IPI_IN_SCP_READY_0` |
| 22 | 3 | **15** | `IPI_IN_SCP_READY_1` ← 就是那个 0x8000 |
| 12 | 2 | 17 | `IPI_IN_SCP_MPOOL_0`（sensor 走这条 legacy 通道） |
| 25 | 4 | 17 | `IPI_IN_SCP_MPOOL_1` |

其它本轮修掉的坑：

- `mtk_mbox_isr()` 只 ack"有 pin 的位"，没 pin 的 pending 位永远清不掉 →
  电平中断风暴 + 每 pin 三行 dump，看起来像死机。现在 ack 全部读到的位，只留一行限流日志。
- `scpreg.scpsys` 是 NULL（DT 没有 `mediatek,scpinfra` 节点），
  `INFRA_IRQ_SET = scpsys + 0xB14` 与 `scp_do_regdump()` 第 14 条都会空指针；
  ready 之后 `scp_logger_notify_ws → mtk_ipi_send → scp_awake_lock` 就崩在这里。
  v7 在 awake lock/unlock 里对 NULL 直接返回，regdump 里跳过。
- `reset_scp()` 里 `add_timer` 了还在 pending 的 ready timer（v4 的 WARNING），改 `mod_timer`。
- 之前 helper 里那张 "LK 寄存器表" 其实是 `scp_excep.c: reg_save_list` 的
  (地址, 转储长度)，被当成 (地址, 写值) 写进活寄存器（往 `SW_RSTN_CLR` 写了
  0x120），已从 helper 删除。

**当前卡点**：sensorhub 注册 sensor IPI 失败 ——
`ipi_comm register ipi 30 failed -3`，因为按厂商表布局后 DT 里没有
29/30/31/32 这些 pin（厂商表里也没有，它们走 `SCP_MPOOL` legacy 通道）。
IIO 设备（`xaga-als`/`xaga-accel`）在，但 `in_accel_*_raw` 读不出值。

**操作上的坑（实测）**：ready 之后再敲一次 `666 0 1` 会**把状态打回 not ready**。
`reset_scp()` 一进来就 `scp_extern_notify(SCP_EVENT_STOP)`（sensorhub 立刻置
`sensor_ready=0`），而已经在跑的固件**不会重新发 READY**（它只在启动时发一次）。
所以：**每次开机只轻启动一次**；要重测就重启，别再敲那条命令。

**v8 结果（09:24）**：sensor pin 放到 mbox0 后注册成功 ——
`ipi_comm sensor IPI handlers registered (ctrl 30, notify 32)`（40 失败但非致命）。
IIO 的几个 `in_accel_*_raw` **文件在、读返回 `-ENODEV`**（`cat` 报"没有那个设备"），
`in_accel_scale` 正常 = 驱动还没把 sensor 拉起来，不是文件缺失。

**剩下的唯一内核卡点**：sensor hub 的应答。
`[ 58.599] sensor_ready sensor hub gave no ready ack; rescue reset disabled`
——READY 通知发出去了，固件没回。同时 `[SCP] scp_logger_notify_ws: id=17` 说明
**固件在要求 AP 取日志**，所以下一步先 `cat /sys/class/misc/scp/scp_A_get_last_log`
（logger 通道现在是通的），看固件自己说 sensor 侧缺什么（很可能是 sensor 共享内存/
sensor list 没被 AP 初始化，或者 notify 走的是 `SCP_MPOOL` 而不是直连 IPI）。

**下一步（需要试）**：把 sensor 的 4 个 pin 放到 **mbox 0**（厂商表里 mbox0 只有
VOW 的 send id0 / recv id2，我们不用音频 VOW，可以让位）：

```
send: 29 (16 slots), 31 (7)         -> pin 0, 8   (各占 8/4 个半字)
recv: 30 (2), 32 (7)                -> pin 12, 13 (占 1/4)
```
合计 17/32，不溢出，也不动 mbox1/3 上的 READY。若固件其实把 sensor 发在别的
mbox，`[MBOX ISR] ... unclaimed irq` 这一行会直接告诉我们它落在哪一位 ——
READY 就是这么找到的。

---

## 0.2 固件日志：本轮最强工具（2026-10-25 10:40）

固件把自己的日志写进 share 里的 logger 环，**不需要任何 IPI 协议就能读**：

```sh
python3 - <<'EOF'
import subprocess
base = 0x8F000000 + 0x1CA700      # = 0x8F1CA700
buf = b""
for off in range(base, base + 0x1000, 4):
    v = int(subprocess.check_output(["devmem", "0x%X" % off]).strip(), 16)
    buf += v.to_bytes(4, "little")
print(buf.decode("latin1"))
EOF
```

格式 `[秒.毫秒](核号) 消息`。读出来的关键内容：

- **固件自己打印了 mbox pin 表，和 v6 按厂商表算出来的完全一致**
  （`mbox3 pin20 index:07 offset:14`、`pin22 index:15 offset:30` = READY_1 的 bit15），
  所以 pin 布局这条彻底结案；
- `[0.080](1) send ready IPI` —— 固件确实发 READY；
- `[0.088](0) ICM40607 detected`、`alsps,startTask`、`[SarAlgorithm]SarAlgorithmStart`、
  `accGyro: app start` —— sensor hub 起来了，加速度计是 **ICM-40607**；
- **`acc map[0]:1, map[1]:0, map[2]:2, sign[0]:1, sign[1]:-1, sign[2]:1`**
  —— 这就是 mount-matrix，直接可用，不用再上机转方向猜；
- 最后一行：**`initSensors: Accelerometer not ready!`** ← 当前真正的卡点。

### 当前唯一卡点：固件认为加速度计"未就绪"

`acc spi_num: 2, eint 12` —— 固件在等 **EINT 12** 上的 data-ready。
下一步方向（按成本排序）：

1. 确认这颗 accel 的 EINT（GPIO12？）在 SCP 侧的 eint/pio 配置里是否使能
   （LK 通常配置；我们的内嵌 DTB 里的 pinctrl/eint 节点要对照厂商 `chopin.dts` 的 accel 节点）；
2. 若 eint 没问题，就是固件内部的 sensor 任务顺序/超时问题，需要看更长的日志
   （环只到 0.096s 就停了，因为 AP 侧 logger 没有持续 drain —— `scp_A_logger_inited` 一直是 0）；
3. 并行可做（不依赖数据）：把 `map={1,0,2}` / `sign={1,-1,1}` 写进 IIO 的
   `in_accel_mount_matrix`，`xaga-accel` 改名 `chopin-accel`。

### 本轮镜像与提交

```
out-scp-v18  md5 9bec10ab6393b10d6aee0525f671c193   ← 基线（timesync 已加）
内核：4c3bc0894(pin表/ISR/awake) 48a44c356(sensor pin) fc34d919a(recv_opt+notify)
      7bd25d27d(prime) 3770ca9c9(config 不阻塞) 33005c5da(notify 窗口) 00038dff4(timesync)
```

**v9~v18 的因果**（交接给下一个会话）：`boot_work` 只在 ready 后跑 → `scp_status`
闸门挡住 ctrl/notify（`comm_preready` 开窗）→ 编译坑（`make | tail` 吞掉失败，
导致三版镜像其实是旧内核）→ 固件 bit8 坐标 → `recv_opt=1` 应答语义 →
**固件执行但不应答，判断"它有没有动"要看 `audit` 里的 `dr[...]`，不要看 ack**。

---

## 0.3 最后一处：样本派发源（2026-10-25 11:00）

v20/v21 之后的状态（已上机验证）：

```
SCP A is ready                         ✅
in_accel_*  = -EAGAIN                  ← 不再是 -ENODEV，active 已置
shared mem config ack -110, continuing ← 固件不回 ack（已知行为，不再阻塞）
sensor_list wait completion timeout    ← 同上
```

`-EAGAIN` 的含义很具体：`sensorhub_iio_read_raw()` 已经放行、`st->active` 已置，
但 `st->has_data` 仍是 0 —— 而**只有** `sensorhub_iio_sample_cb()`
（`sensorhub_iio.c:112`，经 `sensorhub_register_sample_cb()` 注册，:460）
会把它置 1：

```c
	st->value[i] = s->value[i];
	st->has_data = true;
```

所以最后要补的是**样本派发源**：sensorhub 核心的 ring reader 必须在 prime
阶段就启动，**不判断 `sensor_ready`**（这颗固件永远不发 sensor ready），
从 SENS 环（`share+0x0CA700`，固件日志证明它在 512Hz 写）取样本后调用
派发函数，把 `sensorhub_sample` 送给已注册的回调。

改动位置：负责“读环 → 组装 `sensorhub_sample` → 派发”的那段代码
（`share_memory.c` 的 ring handler / `sensor_list` 的读取路径），
把它的启动条件从 `sensor_ready` 改成“handler 已注册 + 环已配置”。

做完这一步：`in_accel_*` 从 `-EAGAIN` 变成数字 → 内核侧完成。
之后按 §7：mount matrix `[1,0,0; 0,-1,0; 0,0,1]`、改名 `chopin-accel`、
`iio-sensor-proxy`。

镜像：`out-scp-v21`（md5 `4d00cd9515461a546ff978d2ad0a508a`）。

---

## 0.4 交接快照（2026-10-25 11:20，v24）

### 一句话

SCP 与 sensor hub 的**软件栈已全部打通**，固件已在 512Hz 采样（ICM40607）、
ALS(BU27030) 已启动；**只差加速度计的 data-ready 中断**（固件日志
`jitter check failed even 4 retries`、固件 IRQ 表 `irq 12 count=0`）。

### 已经确定、不必再验证的事实

| 结论 | 证据 |
| --- | --- |
| SCP 启动方式：轻启动 `666 0 1`，一次开机只用一次 | `SCP A is ready` 稳定复现 |
| READY 在 **mbox3 bit15**（`IPI_IN_SCP_READY_1`，id 22） | 固件自打印 pin 表与 `scp_ipi_table.h` 一致 |
| 固件**执行但不回 ack**（config/list 都 `-110`） | `dr[000ca700]` 被固件写成 `0 0 0fffd8 24` |
| 数据走 **mpool 通道 11/12、24/25**，不是直连 29-32 | `[MBOX Error]null ptr ipi_id=12` → 加 `legacy_table` 后消失 |
| 固件 sensor hub 运行 | `ICM40607 detected`、`acc: enable done`、`FIFO TH=0x10`、`alsps,startTask` |
| mount matrix | 固件日志 `acc map[0]:1, map[1]:0, map[2]:2, sign 1,-1,1` → `[1,0,0; 0,-1,0; 0,0,1]` |
| 加速度计无数据 | `jitter check failed even 4 retries`、`irq 12 count=0` |
| GPIO12 电气没问题 | `pinconf` 显示 pull up + input enabled；`pinmux-pins` 显示 MUX/GPIO 均 UNCLAIMED |
| 厂商内核不配置 SCP eint | 厂商 `scp/`、`sensorhub/` 里 grep `eint` 无结果 → 由 LK/固件负责 |

### 下一步（按顺序）

1. 读固件日志里 `ICM4X6XX` 段之后的寄存器写，确认 **INT1 是否真的使能**
   （`0x2D` 系列，`INT1_CTRL`/`FIFO_CTRL3`）：
   ```sh
   python3 - <<'EOF'
   import subprocess
   b=0x8F000000+0x1CA700; buf=b""
   for o in range(b,b+0x2000,4):
       buf+=int(subprocess.check_output(["devmem","0x%X"%o]).strip(),16).to_bytes(4,"little")
   t=buf.decode("latin1"); print(t[t.find("ICM4X6XXInitConfig"):][:1500])
   EOF
   ```
2. 若 INT 已使能 → 查 SCP eint 12 与这颗 accel 的对应关系（厂商/LK 的 scp eint 配置）。
3. 数据一通：`in_accel_*_raw` 出数 → 改 DT 的 mount matrix、改名 `chopin-accel/als`、
   装 `iio-sensor-proxy`（+ udev `ACCEL_MOUNT_MATRIX`）。
4. 自动亮度同一条链（ALS 已在固件里跑）。

### 镜像 / 提交

```
out-scp-v24/boot-stockdt.img   md5 688786bba8407bc66d1226a8857d6e82   ← 基线
内核：4c3bc0894 48a44c356 fc34d919a 7bd25d27d 3770ca9c9 33005c5da
      00038dff4 933fa9b5b 39b8014d0 574a03f38 96d970f24 14038c502 e6d267c86
```

### 调试约定（这次会话踩过的坑，别再踩）

- **绝不要 `make ... | tail`**：管道会吞掉 make 的失败，我们因此有三个镜像其实是旧内核。
  一律 `make ... >/tmp/b.log 2>&1; echo rc=$?; grep -E "error:" /tmp/b.log`。
- 判断“镜像是不是新的”看 `uname -a` 的**构建时间**（提交哈希取的是构建时 HEAD，会滞后一版）。
- 判断“固件有没有真的动”看 `audit` 里的 `dr[...]`，**不要看它回不回 ack**。
- 固件日志：`0x8F000000+0x1CA700`，4 字节小端转 ASCII，格式 `[秒.毫秒](核号) 消息`。
- 贴日志一律先 `grep`/`tail` 过滤，不要把整段 dump 贴进来（上下文会被吃掉）。

---

## 0.5 加速度计 INT：主线的 EINT 驱动把 SCP 的线 mask 了（2026-10-25 11:30，v25）

### 结论（等上机确认，但证据链很硬）

把“厂商内核 vs 主线内核”的 EINT 初始化逐行对了一遍，差别只有一处：

| | 厂商 `mtk_eint_hw_init()` | 主线 `mtk_eint_hw_init()` |
| --- | --- | --- |
| `DOM_EN`（+0x400） | 写 `0xffffffff` | 写 `0xffffffff` |
| **`MASK_SET`（+0x0C0）** | **不写** | **写 `0xffffffff`（全部 mask）** |

而 LK 自己的启动日志（`n10p/exp15.txt`，每次开机都有）写的是：

```
Unmask all EINT event mask
```

所以这颗板子的真实启动顺序是：

```
LK 把所有 EINT 打开
  → SCP 固件在 AP 起来之前就配置好加速度计的 data-ready 线（ICM40607 → EINT 12）
  → Linux 的 pinctrl 驱动 probe，把全部 EINT 又 mask 一遍   ← 只有主线这么干
  → 固件永远等不到 EINT 12（固件 IRQ 表 irq 12 count=0、jitter check failed、
    initSensors: Accelerometer not ready!）
```

这条线是 **GPIO12 = EINT12**（固件日志 `acc spi_num: 2,eint 12`；厂商 DTS 的
`gpio_init_default` 里 pin 12 = mode 0、input、pull-up，和板子上实测的 pinconf 一致），
它是 SCP 的线，不是 AP 的线——AP 侧从来没有任何驱动 request 过它
（所以 `pinmux-pins` 显示 UNCLAIMED 是正常的）。

**注意 mask 位语义**：EINT `MASK` 寄存器 bit=1 表示被屏蔽；`MASK_CLR`（+0x100）
写 1 是解除屏蔽。eint12 在 port0，所以就是 bit12 = `0x1000`。

### 先验证（不刷机，在 v24 上直接做）

EINT 寄存器 AP 侧可以直接 `devmem` 读写，所以这一步等价于 v25 内核在 probe 时做的事：

```sh
# 1) 只读：EINT12 现在是什么状态
devmem 0x100053C0        # GPIO12 MODE，bits[3:0]（0 = GPIO/EINT 可用）
devmem 0x1000B080        # EINT MASK[0..31]：bit12=1 就是被 mask 了  ← 关键
devmem 0x1000B000        # EINT STAT[0..31]
devmem 0x1000B140        # SENS[0..31]（0 = 边沿，1 = 电平）
devmem 0x1000B300        # POL[0..31]
devmem 0x1000B400        # DOM_EN[0..31]
devmem 0x1000B50C        # DBNC ctrl，eint12 在最低字节
devmem 0x100052C0        # GPIO12 输入电平（bit12），连读 3~4 次看有没有脉冲

# 2) 解除 EINT12 的屏蔽（这一步就是 v25 的改动）
devmem 0x1000B100 32 0x00001000
devmem 0x1000B080        # 再读，bit12 应该变成 0
sleep 2
for f in /sys/bus/iio/devices/iio:device*/in_accel_x_raw; do echo "$f: $(cat $f 2>&1)"; done
dmesg | tail -30         # 特别看有没有 "nobody cared" / "Disabling IRQ"
```

判定：

| 现象 | 结论 |
| --- | --- |
| `in_accel_*_raw` 出数字（不再是 `-EAGAIN`） | **就是它**，刷 v25 落地 |
| `0x1000B080` 的 bit12 本来就是 0 | 不是 mask 问题，按 fallback 看 SENS/POL/GPIO12 电平 |
| bit12 原本是 1，放开后仍无数据 | EINT 没到 SCP，或芯片 INT1 没使能（回到固件侧） |
| dmesg 出现 `nobody cared` / `Disabling IRQ` | AP 抢了这条线，需要再做 domain 隔离（见下） |

**副作用要盯**：这条线放开后 AP 的 GIC 212 也会看到它，而 AP 上没有它的 handler，
`handle_level_irq()` 会 ack+mask+unmask 一遍，`note_interrupt()` 计数到 10 万次
（512 Hz 约 3 分钟）会 `Disabling IRQ` 把它重新 mask 掉。所以验证时要先看数据有没有出来，
再看 `dmesg` 有没有 `nobody cared`；如果确实被 AP 抢，下一步是在
`mtk_eint_irq_handler()` 里对 keep 名单里的 eint 只 ack、不派发给 Linux。

### v25 改动

```
out-scp-v25/boot-stockdt.img   md5 dcf6e052511da377ca95790c4300d3e2
内核：eda9a9d79  pinctrl: mediatek: keep the SCP's EINT lines unmasked at probe
```

- `drivers/pinctrl/mediatek/mtk-eint.c`：新增 DT 属性 `mediatek,eint-keep-unmasked`，
  `mtk_eint_hw_init()` 写完 `MASK_SET=0xffffffff` 之后，把名单里的 EINT 用
  `MASK_CLR` 放回去；`mtk_eint_do_resume()` 同样处理（resume 会用 AP 的
  `cur_mask` 重写整个 mask，不然一次 suspend 就回到原样）。
- DT（`mt6893-xiaomi-chopin.dts`）：`&{/pinctrl} { mediatek,eint-keep-unmasked = <12>; };`
  （板子的 pinctrl 节点是厂商路径 `/pinctrl`，不是 `mt6893.dtsi` 里的 `pio` 标签）。

### 上机结果：mask 修好了，但它不是卡点

v25 开机 0.254s 就打出 `mt6893-pinctrl 10005000.pinctrl: EINT 12 left unmasked
(owned by another core)`，`0x1000B080` 里 bit12=0；但 `in_accel_x_raw` 仍是 `-EAGAIN`，
GPIO12 恒低、EINT12 的 STAT 恒 0。顺带确认的事实：

| 结论 | 证据 |
| --- | --- |
| **固件是轻启动放出来的，不是 LK 起的** | 重启后、敲 `666 0 1` 之前固件日志区是空的（`FWLOG: (empty)`），敲完才有完整启动日志 |
| 第二次轻启动不重启固件 | 前后日志第一条时间戳都还是 `[0.026]`，`dmesg` 只有 `sensor ready state changed to 0` |
| SENS 环在 **share+0xca700**（不是 0x100000） | DT `scp_mem_tbl` 按 id 累加：SECDUMP 0 → VOW size 0xca700 → SENS 0x100000 → LOGGER 0x1ca700（与实测日志位置一致） |
| **AP 把环配好了，固件一个字节都没写** | 环头 `rp=0 wp=0 buf=0xfffd8 item=36`（= `share_mem_init()` 算出来的），数据区全 `0xff`，`wp` 一秒不动 |
| GPIO12：mode0 / 输入 / IES=1 / 上拉 R1R0=01 | `0x10005310`=0、`0x10005000` bit12=0、`0x11c20020` bit11=1、`0x11c20040` bit2=0、`0x11c20060/70` bit2=1/0；`pinconf` 也是 `pull up (101 ohms), input enable` |
| 线一直被拉低 | 41 万次采样 `di12_changes=0`；有上拉还读 0，只能是芯片在驱动低 |
| EINT12 = **level + 高有效**（bit12=1/1） | `sens=0xfef7ffff`、`pol=0x00081000`；对照触摸 GPIO19 是 rising、pol bit19=1，mt6360 的 edge/level 与 sens 位也一一对上 |
| **POL（0x300）是只读状态**，改极性要写 `POL_CLR`/`POL_SET`（0x380/0x340） | 直接写 `0x1000B300` 无效，读回仍是 `0x81000` |

⇒ 判断：**芯片没有产生 data-ready**（“高有效、空闲低”，恒低就是没有中断），固件 boot 时
读不到样本 → `jitter check failed` → `Accelerometer not ready!`，之后 AP 的 enable 只有 `-110`。
现在要回答的是“为什么 ICM40607 不出数据”，而不是继续折腾 EINT。

### 下一步（按顺序）

1. **先看固件日志的尾部**（环有 1MB，之前只读了前 8KB）：
   ```sh
   python3 - <<'EOF'
   import mmap, os
   f = os.open("/dev/mem", os.O_RDONLY | os.O_SYNC)
   m = mmap.mmap(f, 0x100000, mmap.MAP_SHARED, mmap.PROT_READ, offset=0x8F1CA000)
   b = bytes(m[0x700:])
   last = max(i for i, c in enumerate(b) if c != 0xFF)
   print("last written offset:", hex(0x700 + last))
   print(b[max(0, last-1600):last+1].decode("latin1"))
   EOF
   ```
2. 尾部只有 boot log、之后全静 → 固件不处理 AP 命令，重点回到 sensor comm（IPI 通道/预启动窗口）。
3. 尾部有 `ICM4X6XXSample, rx dataInfo error` / `spi` 错误 → SPI 读 FIFO 的问题，查 SCP SPI2。
4. 想单独验证“芯片会不会出数据”可以试 `devmem 0x1000B380 32 0x1000`（POL_CLR 改成低有效）：
   线被拉低，改完应立刻 STAT bit12=1；SCP 若随即读 FIFO，`wp` 就会动。这只能当诊断。

### 结论（固件日志尾部，已存 `port/audit/fw-log-scp-boot-2026-10-25.txt`）

日志环只有 11 KB（`[1.050]` 之后就再也没有输出），但足够说明：

- 固件自己在 `[0.600]` **把加速度计打开**（`sensorPowerAcc on:1`、`ICM4X6XXgyEnable`），
  配了 `hwrate 512000`、`delay 2499584`(≈400Hz)、`watermark 1`、`FIFO TH=0x10`；
- 从 `[0.911]` 到 `[1.031]`，每 3 ms 一条 `accGyro CHISC pending by:36`（在等样本）；
- `[1.033] jitter check failed even 4 retries:0/1` → `[1.043] ICM4X6XXAccPowerOff`
  → `[1.045] acc: disable done` —— **固件自己把加速度计关掉了**，然后彻底安静。

另外 `[0.100..0.106] dispatch_to DEPUTY fail [13,13] / [53,13] / [71,13] / [54,13]`
值得注意：传感器任务往 DEPUTY 派发失败了四次。

**AP 侧的命令完全没被处理**：写 `in_accel_sampling_frequency` 之后，环形缓冲的最后写入
位置仍是 `0x2b68`、没有新日志、GPIO12 依旧没有翻转。也就是说 `-110` 不是“不回 ack”，
而是**命令没进到固件的传感器服务里**。

### 所以现在真正要回答的两个问题

1. **这颗 ICM40607 到底会不会出数据？** 固件打开它 400ms 一个样本都没拿到。
   需要排除：传感器供电轨/时钟（固件用 PWRAP 操作 PMIC，`[PWRAP]init done` 出现过）；
   以及 SCP SPI2 的 burst 读。最省事的对照是：**刷回原厂 boot，看 Android 里自动旋转是否正常**
   —— 如果原厂也不转/传感器列表里没有加速度计，那就是硬件或 sensor 供电的问题，
   跟我们的移植无关。
2. **AP→SCP 的 sensor comm 为什么没进去？** 环已配好、IPI 也发出去了，固件却不执行。
   下一步按 §0.4 的 IPI 表逐条核对 ctrl/notify 的 mbox/pin，并确认固件确实收到了
   （最直接的证据就是日志里出现新的 `sensorPowerAcc`/`ICM4X6XX` 行）。

---

## 0.6 sensor 命令通道：pin 放错了 mbox（v26，2026-10-25 12:10）

**根因候选**：`sensor ctrl/notify`（29/30/32）被放在 **mbox0**，但厂商 sensorhub 布局
（`n10p/vendor/.../scp/mt6853/scp_mbox_layout.h`）把它们放在 **mbox2**，紧跟在
`SCP_MPOOL_0` 后面：out 尺寸 34/16/7、in 尺寸 30/2/7。mbox0 在那套布局里是音频 VOW 的。

为什么确信是 mt6853 那套：我们的 mbox3 表（send 14/15/16/17/18 = DVFS_SET_FREQ_1/
C_SLEEP_1/TEST_1/LOGGER_CTRL/SCPCTL_1，recv 20..23 = ERROR_INFO_1/LOGGER_CTRL/
READY_1/RAM_DUMP_1）和固件自己打印的 pin 表逐项一致，而那正是 mt6853 的编号。
之前“厂商表里没有 29/30/32”的结论是从 **mt6885** 的头文件得出的（那份没有开
CONFIG_MTK_SENSORHUB），mt6853 那份是有的——所以它们不是走 legacy，而是 mbox2 上的普通 pin。

这解释了全部现象：AP 把命令发到 mbox0，固件在 mbox2 上听 → 命令永远不到；
固件在 mbox2 上的应答落到我们没有 pin 的位上 → `-110`；`sensor_list`、
shared-memory config 全部超时 → 固件的 sensor 服务没被喂过任何东西。

v26 改动（`mt6893-xiaomi-chopin-scp-dbg.dtsi`）：
```
send_table: <11 2 34>, <29 2 16>
recv_table: <12 2 30 0>, <30 2 2 0>, <32 2 7 0>
```
镜像：`out-scp-v26/boot-stockdt.img` md5 `8576109210fabc68f75c08c2938180dc`，
内核 `adb041541`。

验证要点：刷 v26 + 重启 + 轻启动后
1. `dmesg` 里 `sensor_list wait completion timeout` / `shared memory config ... -110` 应该消失或变少；
2. SENS 环（`0x8F0CA700`）的 `wp` 应该开始前进；
3. 固件日志尾部应出现 100s 之后的**新行**（`sensorPowerAcc`/`ICM4X6XX`）。

---

## 0.7 v26 错了、v27 回退 + 用户态就绪（2026-10-25 12:45）

### v26 的结论被推翻

把固件日志环（1MB）里所有 `id=29/30/31/32` 的行 grep 了一遍：**一条都没有**。
固件自己打印的 pin 表只有 `mbox2 id=11 mg_sz=34`、`mbox4 id=24 mg_sz=34`、
`mbox3 id=18`、`mbox0 id=0` 这些。v26 把 29 挪到 mbox2 后，AP 在固件没有 pin 的
位上报文 → 固件 ISR 报 `[MBOX ISR]pin table err, status=20000` 并**每 3ms 打印整张表**，
日志环被刷满、旧日志被覆盖。所以 v26 的 DT 改动是**错误方向**，已 revert。

⇒ 与 §0.4 的结论一致：**这份固件不带 SENSOR_CTRL/NOTIFY（29-32），sensor 命令只能走
SCP_MPOOL（11/24）**。AP 侧 `ipi_comm.c` 现在用的是 `mtk_ipi_send(&scp_ipidev, 29, ...)`
（直连 mbox），需要改成走 legacy mpool 封装：

- 发送：`mtk_tinysys_scp_ipi_send(id, buf, len, wait)` —— 它会把
  `[u32 id][u32 len][payload]` 打进 `IPI_OUT_SCP_MPOOL_0/1`（11/24，34 slots）；
- 接收：`scp_legacy_handler()` 解包头后按 `pkt.id > IPI_MPOOL` 分派到 `mpool[]` 表，
  所以要按它那套 id 注册（`scp_ipi_registration()`）。

### v27 内容

```
out-scp-v27/boot-stockdt.img   md5 fa4a82ff362aa2e90a1d3472f25d29e9
内核 6f1a78257
```

1. revert v26 的 DT 改动（不再往 mbox2 放 29/30/32，`pin table err` 风暴消失）；
2. IIO 设备改名：`xaga-als`/`xaga-accel` → **`chopin-als`/`chopin-accel`**；
3. mount matrix 用固件自己打印的值（不是猜的）：
   固件 `map[0]:1 map[1]:0 map[2]:2, sign[0]:1 sign[1]:-1 sign[2]:1`
   ⇒ 屏幕 x = +设备 y，屏幕 y = −设备 x，屏幕 z = +设备 z：
   ```dts
   mount-matrix = "0",  "1", "0",
                  "-1", "0", "0",
                  "0",  "0", "1";
   ```

刷 v27 后**数据仍不会出来**（`-EAGAIN`），这次的目的只是：止住风暴、把名字和方向做对、
为最后一步（mpool 传输）准备好。

---

## 0.8 v27 上机通过 + 一个能把 0x20000 对上账的细节（2026-10-25 13:00）

v27 实测（镜像 `fa4a82ff...`）：

| 检查 | 结果 |
| --- | --- |
| `EINT 12 left unmasked` | 有（0.25s） |
| IIO 名字 | `chopin-als` / `chopin-accel` ✅ |
| `in_accel_mount_matrix` | `0, 1, 0; -1, 0, 0; 0, 0, 1` ✅ |
| `pin table err` / `[MBOX]recv pin` 计数 | **0 / 0**（风暴消失），环尾 0x269a |
| `in_accel_x_raw` / SENS `wp` | 仍是 `-EAGAIN` / 0（预期，数据链还没通） |

**新细节**：`mbox_setup_pin_table()` 在 `is64d` 下是按半字算的
（`last_sz = DIV_ROUND_UP(msg_size,2)`，`pin_index = last_idx + last_sz`）。
所以 mbox2 上「mpool 34 slots → sensor ctrl 16 slots」两条 pin 的**中断位分别是 0 和 17**，
不是 0 和 34。v26 里那句 `[MBOX ISR]pin table err, status=20000`（bit 17）**正好就是
我们发出去的 IPI 29 那一位** —— 固件在 mbox2 的 bit17 上没有 pin。
这条把 §0.7 的结论钉死了：**固件确实没有 29/30/32，命令只能走 mpool 11/24**。

### v28 要做的（下一步）

`drivers/misc/mediatek/sensor/2.0/sensorhub/ipi_comm.c`：

- 发送改成 `mtk_tinysys_scp_ipi_send(IPI_OUT_SENSOR_CTRL, buf, len, wait)`：
  它把 `[u32 id][u32 len][payload]` 打进 `IPI_OUT_SCP_MPOOL_0`（11，34 slots），
  固件那边由 legacy mpool 的分派器按包头里的 id 找自己的通道；
- 接收：固件若用 30 回，AP 侧目前没有对应 pin（`scp_ipi_legacy_id[30]` 是 IPI_NO_USE），
  要么按 mpool 虚拟 id 注册（`IPI_SENSOR`，> `IPI_MPOOL` 的那几个），要么先只验证
  「固件收不收到」——判据就是固件日志里出现 100s 之后的新行。

---

## 0.9 v28：sensor 命令改走 SCP_MPOOL 传输（2026-10-25 13:20）

```
out-scp-v28/boot-stockdt.img   md5 499d14aac153d2046d10b17f1d7c8e75
内核 b3dbedf9f
```

`ipi_comm.c` 的发送从「直连 mbox 的 29 号 pin」改成 legacy mpool：

```c
/* 前 */
ret = mtk_ipi_send(&scp_ipidev, id, 0, tx, ipi_len(tx_len), 0);
/* 后：打 [u32 id][u32 len][payload] 进 IPI_OUT_SCP_MPOOL_0/1（11/24） */
ret = mtk_tinysys_scp_ipi_send(id, tx, tx_len, 0, SCP_A_ID);
```

并加了一行 marker：`sensor channel %d is sent over SCP_MPOOL`（`pr_info_once`），
用来在 dmesg 里确认这条路径真的被走到了。

**接收侧没动**：固件若在 30 上回包，AP 仍然收不到（`scp_ipi_legacy_id[30]` 是
IPI_NO_USE），所以 `-110` 还会出现。v28 的唯一目标是回答一个问题：
**固件会不会对这条命令有反应**（判据：固件日志环尾从 0x269a 往前长、出现
100s 之后的新行；以及 SENS 环 `wp` 是否开始动）。

---

## 0.10 v28 的 -5 与 v29（2026-10-25 13:25）

v28 上机：marker 出现了（`sensor channel 29 is sent over SCP_MPOOL`），但所有传输都是 **-5**：

```
sensorhub share mem config ack -5, continuing
sensor_list notify fail -5 ...
sensor_ready notify ready to scp fail -5
[SCP] recovery success            ← 2s 后
```

原因：`mtk_tinysys_scp_ipi_send()` 在 `is_scp_ready()` 为 0 时直接返回
`SCP_IPI_NOT_READY`，而 sensor 的 prime 本来就跑在 READY 通知之前 → 一个包都没发出去。
（同日还出现一次屏幕花屏，时间点紧跟 `[SCP] recovery success`，即 SCP 被复位；
重启应能恢复，要单独确认不是显示回归。）

v29 改成**自己封包**，绕开 ready 检查：

```
out-scp-v29/boot-stockdt.img   md5 cff22d9ebad648aad0269e0f946511a0
内核 2c3edf8d9
```

```c
uint32_t hdr[2] = { id, tx_len };          /* [u32 id][u32 len] */
memcpy(pkt, hdr, 8); memcpy(pkt + 8, tx, tx_len);
mtk_ipi_send(&scp_ipidev, IPI_OUT_SCP_MPOOL_0, 0, pkt, ipi_len(tx_len + 8), 0);
```

判据仍然是：固件日志环尾是否变长、是否出现 100s 之后的新行；以及 SENS `wp`。

---

## 0.11 v29 命中：固件收到了 mpool 包，但 pkt_id 不对（→ v30）

v29 的固件日志第一次出现了这条：

```
[0.808](0) legacy_handler id 11,pkt_id:29 handler is null!!! prdata:0x0 data:0xbe1cc len:136
[0.912](0) legacy_handler id 11,pkt_id:29 handler is null!!! prdata:0x0 data:0xbe1cc len:136
```

意思很明确：**包进了 SCP_MPOOL（id 11）、固件的 legacy 分派器也跑了，但 `pkt_id=29`
在它那张虚通道表里没有 handler**，于是丢弃。

看 `scp_rv.h` 的 id 空间就懂了：`enum ipi_id { IPI_MPOOL, IPI_CHRE, IPI_CHREX,
IPI_SENSOR, IPI_SENSOR_INIT_START, SCP_NR_IPI }` —— 这是**另一套从 0 开始的虚 id**，
`scp_legacy_handler()` 用 `mpool[pkt.id - IPI_MPOOL - 1]` 取 handler。所以传感器流量
必须打 **`IPI_SENSOR`** 这个虚 id，而不是 29。

v30 改动（`ipi_comm.c`）：

```c
uint32_t hdr[2] = { (uint32_t)IPI_SENSOR, (uint32_t)tx_len };   /* 原来是 id=29 */
...
scp_ipi_registration(IPI_SENSOR, ipi_comm_mpool_handler, "sensor");  /* 收回包 */
```

```
out-scp-v30/boot-stockdt.img   md5 3501a924e6bd7e65dbd3248e942630bc
内核 667194ff5
```

判据：固件日志里若出现 **不再是 “handler is null” 的新行**（例如
`sensorPowerAcc`/`ICM4X6XX`/`share mem`），说明命令被执行；然后再看 SENS `wp`。

---

## 0.12 v30 打通了发送、回包只是慢（→ v31）

v30（`pkt_id = IPI_SENSOR`）上机：

- `handler is null` 只剩 **1** 次（v29 是每包一次），说明分派器现在找得到 handler；
- 固件日志环从 0x27d1 长到 **0x2b32**；
- dmesg 出现 **`ipi_comm dropped transfer`** —— 这行来自 `ipi_comm_complete()` 里
  `hw->context == NULL` 的分支，含义是**回包到了，但我们 100ms 的超时已经放弃**。

⇒ 传输层通了，只是固件的应答比 100ms 慢。v31 把 `wait_for_completion_timeout()`
从 100ms 提到 1000ms（和厂商 ipi_comm 一致）：

```
out-scp-v31/boot-stockdt.img   md5 664383d5c10190844f44fabc06acc02d
内核（见 git log）
```

判据：`ipi_comm timeout` / `-110` / `dropped transfer` 是否消失，`sensor_list`
是否拿到真实列表，固件日志是否出现新的 `sensorPowerAcc`/`ICM4X6XX`。

---

## 0.13 v31：命令已经进到固件的 command 解析器（2026-10-25 13:10）

v31（超时 1s）上机，固件日志首次出现**对命令本身的反应**：

```
[2.110](0) contextHubHandleIpiRxEvent cannot find cmd
[2.110](0) contextHubHandleIpiRxEvent cannot find cmd
[3.130](0) ... 每 1s 复现（正好是我们的重试节奏）
```

也就是说：包进了 mpool → 分派器找到 handler（v30 起 `handler is null` 基本消失）→
**context hub 的 IPI 接收函数被调用了**，但它在 payload 里找不到认识的 cmd，于是丢弃、
不回 ack（所以 `-110` 还在）。

一个关键细节：v29 那行固件日志里 `data:0xbe1cc **len:136**` —— 136 = 8 字节包头 + 128
字节 payload，说明固件把**整个包（含 [u32 id][u32 len] 头）**交给 handler 自己解析，
不像我们 AP 侧 `scp_legacy_handler()` 那样先剥 8 字节。所以 cmd 的读取偏移很可能与我们的
布局不一致（它可能从包头里读到了 cmd=0/1 之类）。

**下一步（离线可做）**：反汇编固件里的 `contextHubHandleIpiRxEvent`——
它从哪个偏移读 cmd、和哪张表比较。定位方法：固件字符串里搜
`contextHubHandleIpiRxEvent cannot find cmd`，反查引用它的代码。
拿到偏移后改 `ipi_comm.c` 的封包即可，其余（传输、用户态）都已就绪。

当前状态：

| 环节 | 状态 |
| --- | --- |
| EINT/IIO 用户态（v25~v27） | ✅ 完成 |
| sensor 命令传输（mpool + IPI_SENSOR） | ✅ 打通到固件 command 解析器 |
| payload 的 cmd 布局 | ❌ 最后一个不匹配（就是这句话） |
| 数据（SENS 环 `wp`、`in_accel_x_raw`） | 仍 0 / `-EAGAIN`，等上一条 |

---

## 0.14 v32：三种 payload 形态的探针（2026-10-25 13:35）

固件说 `cannot find cmd`，说明传输对、**cmd 的字节位置不对**。手工反汇编没能
定位 `contextHubHandleIpiRxEvent`（PIC 代码、无绝对指针，831 个 `auipc` 里只能稳定
解出 71 个目标、字符串附近无命中），所以改用经验探针。

`ipi_comm.c` 每次传输轮换三种封包并在 dmesg 里打印模式：

| mode | 封包 |
| --- | --- |
| 0 | `[u32 IPI_SENSOR][u32 len]` + `struct sensor_comm_ctrl`（现状） |
| 1 | `[u32 IPI_SENSOR][u32 len]` + ctrl，但 sensor_type/command 两个字节交换 |
| 2 | `[u32 IPI_SENSOR]`（无 len 字段）+ ctrl |

dmesg 里会出现 `mpool framing 0/1/2: channel 29, N bytes`；固件日志里
`cannot find cmd` 消失（或开始出现 `contextHubHandleIpiRxEvent` 之外的新行）的那一版
就是正确的布局。

```
out-scp-v32/boot-stockdt.img   md5 d055e9cd0c2c5a538ddfe92c0aac4213
内核 6fd3ffc6a
```

---

## 0.15 v32 结果与 v33（可运行时切换封包）

v32（自动轮换三种封包）上机：

- **`cannot find cmd` 计数 = 0**（v31 是每包一次）—— 说明固件不再报 cmd 解析失败；
- 但每次传输仍然 `-110` 超时、固件日志没有新增 100s 之后的行。

也就是说：**某一种封包被解析了，但还不能确定是哪一种**（一轮开机里三种都试过，
超时和探测混在一起分不开）。v33 把 `mpool_mode` 做成模块参数，一版镜像就能逐个试：

```
out-scp-v33/boot-stockdt.img   md5 dff6670ca35abdae9a1c6437690db2c4
内核（见 git log）
/sys/module/sensorhub/parameters/mpool_mode    # 0/1/2
```

| mode | 封包 |
| --- | --- |
| 0 | `[u32 IPI_SENSOR][u32 len]` + ctrl |
| 1 | 同 0，但 sensor_type/command 交换 |
| 2 | `[u32 IPI_SENSOR]` + ctrl（无 len） |

试法：`echo N > /sys/module/sensorhub/parameters/mpool_mode`，
然后 `echo 50 > in_accel_sampling_frequency` 触发一次，看固件日志是否出现新行/是否回包。

---

## 0.16 v34/v35：卡在 mbox ack，不是封包（2026-10-25 13:40）

- v33 已排除 mode 2（唯一还会触发 `cannot find cmd` 的形态）；mode 0/1 都能被解析、不报错。
- v34：`init-start` 用“超时后补发”，那行标记从未出现 → 说明超时路径没走到预期分支。
- v35：改成**第一条命令前无条件发 `IPI_SENSOR_INIT_START`**，并打印每次发送的返回值：

```
[27.869729] ipi_comm init-start sent (0)
[27.869743] Error: scp_ipidev IPI 11 timeout at ... (last done is IPI 11)   ×18
[27.870222] ipi_comm mpool framing 0: ch 29, 44 bytes, send ret 0
ring end: 0x2593 | cannot find cmd: 0 | handler null: 1
```

含义很清楚：

1. `send ret 0` → **包写进了 mbox**（`mtk_ipi_send` 成功）；
2. 紧接着的 `IPI 11 timeout` 来自 `mtk_tinysys_ipi.c: ipi_timeout_dump()` →
   **固件没有回 mbox ack**；
3. 固件日志里对这批包**一条记录都没有**（连 `legacy_handler ... handler is null` 都不出现），
   说明它这次根本没走到分派器。

⇒ 当前唯一的硬阻塞是：**固件不 ack SCP_MPOOL_0 这条 pin**。
这和封包格式无关（v33 已证明 mode 0/1 会被解析），更像是固件在某个状态下不再消费该 pin
（例如 sensor 栈在 `jitter check failed` 之后进入“放弃”状态，或在等 AP 先完成
`SENSOR_INIT_START` 之外的握手）。

两个子代理正在离线挖：
- A：`contextHubHandleIpiRxEvent` 从 payload 哪个偏移读 cmd、接受哪些 cmd；
- B：legacy/mpool 接收路径的 ack 时机、pkt_id→handler 表、以及固件在什么状态下才消费 pin。
结论会写进 `work/accint/reA.md` / `reB.md`。

---

## 0.17 v36：读数把卡点定到「SCP 不服务 mbox2 接收中断」（2026-10-25 13:30）

v36 每次发送前后打印 mbox2 的中断寄存器：

```
before: recv irq 0x0  send irq 0x0
after : recv irq 0x0  send irq 0x1
```

| 读数 | 含义 |
| --- | --- |
| `recv irq` 恒 0 | SCP 从没往 AP 发过东西（符合预期） |
| `send irq` 发送后置 1 | AP 的消息确实写进 pin 了，`send ret 0` 也印证 |
| 这条 bit 一直没被固件清 | **SCP 侧的 mbox2 接收中断没被服务** |
| 固件日志对这批包一条都没有 | 连 `legacy_handler` 都不打，说明 ISR 根本没跑 |

⇒ 结论：**不是封包、不是 pin 号、不是命名，是 SCP 没有对这条 mbox 中断做出反应。**
这解释了 v29（刚起来时还能看到 `legacy_handler`）到 v35/v36（彻底沉默）的变化，
也说明“先发 init-start”反而把固件带进了不处理的状态。

## 0.18 v37：发送时断言 awake 握手（2026-10-25 13:45）

按上面的结论，最可能的原因是 **SCP 处于 idle/WFI，mbox 中断没能把它拉起来**——
我们这条链路在 v23 把 awake 握手改成了“advisory”（不复位、不阻塞），等于没人给核发唤醒。

v37 在每次 mpool 发送前后调用 `scp_awake_lock()/unlock()`（advisory 版：只写
`INFRA_IRQ_SET`，不等 ack），并把返回值打进日志：

```
out-scp-v37/boot-stockdt.img   md5 6fb80c9f9d9f74386b438d12c9a332f4
内核 1b89734ee
ipi_comm mpool framing 0: awake 0 send ret 0, after: recv irq 0x? send irq 0x?
```

判据：`send irq` 这次是否被清、固件日志是否出现 `legacy_handler`/新行、`-110` 是否变化。

---

## 0.19 固件反汇编结论：这条固件根本没有 MTK sensor 通道（2026-10-25 13:55）

子代理对 `scp.img` 的完整分析见 **`work/accint/reB.md`**（附带 `rv.py`/`walk.py`/`xr.py`/`scp4.asm`）。
关键事实（代码地址 = `scp_code.bin` 偏移）：

| 事实 | 地址/证据 |
| --- | --- |
| mbox ISR **内联**调用 pin handler，**返回之后**才写硬件 ack | `0xe5fa jalr a4` → `0xe622 sw s5,0(a0)` |
| IPI 11 的 handler = legacy 分派器 | `0x1b2d0`（注册于 `0x1b338`，缓冲 `0xbe1cc`） |
| `pkt_id 3` → `chre_fw_ipi`，里面有**无上限**的拷贝-校验自旋 | `0x3b3ee`：`0x3b408 memcpy` → `0x3b414 memcmp` → `0x3b418 bnez` |
| 只有一个 136 字节共享槽，就地消费 | `.bss 0xbe1cc`，payload = buf+8 |
| legacy 表 4 项，全部 boot 时注册 | `0xc007cc`，index = pkt_id-1：1 chre / 2 chrex / 3 **chre_fw_ipi** / 4 elliptic；**29 是 NULL** |
| AP 侧 `mpool[4]` 越界 | `scp_legacy_handler` 用 `mpool[pkt.id-1]`，以前没有边界检查 |

**因果链**：我们把 sensor_comm 的包按 `pkt_id=3` 发进 mpool → 落到 CHRE 解析器 →
payload 不是 CHRE 消息 → 它在 ISR 里自旋 → ack 永远不写 → SCP 被锁死
（固件自己的日志停在 `t=1.050s`，正好对上）。**这不是封包偏移问题，是打错了子系统。**

⇒ 结论：**这份固件没有 MTK 的 sensor_comm 通道**（29/30/32 在它表里不存在），
它对外只暴露 CHRE（`chre_fw_ipi`）、CHREX、elliptic。想在这颗 SCP 上拿传感器数据，
必须实现 **CHRE 主机侧协议**，而不是继续调 MTK sensorhub 栈。

## 0.20 v38：止损（2026-10-25 14:00）

```
out-scp-v38/boot-stockdt.img   md5 8303affdd4e2daa86935bfee3b8fe69a
内核 59a6ea088
```

1. `ipi_comm` 的 mpool 发送改成**直接返回 -ENODEV** 并打一行
   `no MTK sensor channel on this firmware; SCP_MPOOL sends disabled (CHRE-only)` ——
   不再把 SCP 锁死；
2. 修掉 AP 侧 `scp_legacy_handler` 的越界读（`pkt_id` 超出 4 项表时直接返回）。

v38 之后系统不会再被我们自己弄卡；IIO 设备、EINT、命名、mount matrix 都还在。

### 下一步的两条路（需要选）

| 方案 | 说明 | 量级 |
| --- | --- | --- |
| **CHRE 主机侧** | 在 Linux 实现 CHRE host↔CHRE 消息协议，经 `IPI_IN/OUT` 的 CHRE 通道（pkt_id 1/3）与固件 nanoapp 通信；Android 的 CHRE HAL 是用户态，协议（`chreHostMessage`）有公开文档 | 大（需要新驱动/守护进程），但不是死路 |
| 暂停 SCP 路线 | 保留现有成果（SCP 启动、EINT、IIO 框架），加速度计另找 AP 侧可达的 IMU（本机是否有备用 IMU 需确认） | 小 |

---

## 0.21 reA 的结论：CHRE 通道是真的，只是我们不认识它的命令集（2026-10-25 14:10）

完整报告 `work/accint/reA.md`。要点：

- 固件**绝对链接在 0 基址**（`lui`+`addi`，不是 PIC），所以之前 auipc 扫描和指针搜索都失败；
- `contextHubHandleIpiRxEvent cannot find cmd` 的两个引用点：`0x3aa12`(fn `0x3a86a`)、`0x3b31e`(fn `0x3b2c4`)，
  两次日志是 0x3a86a 打完仍继续调用 0x3b2c4 造成的；
- 封包 `[u32 pkt_id][u32 len][payload]` **是对的**；`pkt_id=3` → `0x3b3ee` 拷 **48 字节** payload 到
  `0xa9ee0` 并投事件 0x401；
- **命令字节在 payload[1]**，与表 `0x75ac8` 比较，只接受 5 个命令：
  **2=FLUSH、4=CONFIG、5=SELF_TEST、8=MASK_NOTIFY、10=TIMESYNC**（`payload[2]=0x2c`）；
  MTK sensor_comm 用的 0(disable)/1(enable)/11(share_mem)/12(debug) **全部不支持** → 无应答；
- pkt_id 表 `0xc07c8`，只有 1/2/3/4 有 handler；29 是 NULL；无范围检查；
- 应答不在 IPI 里发：handler 只投事件 0x401，真正处理在 sensor task 里；连续两包会因
  固件内部计数 (`0xa9edc`) 失配互相作废。

⇒ **方向修正**：CHRE 通道可用，但只能发那 5 个命令；传感器“使能/配置”必须走 **CONFIG(4)**，
且一次只能一包。

## 0.22 v39：一次 CONFIG 探针（2026-10-25 14:15）

```
out-scp-v39/boot-stockdt.img   md5 74dab74a3a3372b540ea4c0c5a53026e
```

- 每次开机**只发一包**：`pkt_id=3`、48 字节、`payload[0]=1`（加速度计）、
  `payload[1]=4`（CONFIG）、`payload[2]=0x2c`；
- 其余 MTK sensor_comm 传输全部快速失败（`-ENODEV`），避免再次触发 ISR 快照循环；
- dmesg 标记：`chsensor: one-shot CONFIG probe, pkt_id 3, 48 bytes` / `chsensor: probe send ret N`。

判据：固件日志是否出现新行（`contextHubIpi sequence not match` 或 sensor task 的
`sensorPowerAcc`/`ICM4X6XX`），以及 SENS 环 `wp` 是否开始前进。

---

## 0.23 v39 上机 + 一个测量方法的坑（2026-10-25 14:25）

v39（单发一次 CHRE CONFIG 探针）结果：

```
ipi_comm chsensor: one-shot CONFIG probe, pkt_id 3, 48 bytes
ipi_comm chsensor: probe send ret 0
```

- **没有 `IPI 11 timeout`** → 单包是安全的，不会再触发 ISR 快照循环；
- 固件日志无新行、`sequence not match` = 0、`cannot find cmd` = 0。
  按 `reA.md`，**成功路径本来就不打日志**，所以“沉默”既可能是接受、也可能是静默丢弃。

想用 `reA/reB` 给的 SCP 内部地址做旁证（payload 副本 `0xa9ee0`、mpool 槽 `0xbe1cc`、
序号 `0xa9edc`），但这次读数**全部为 0**，连对照点 `0x10500004`（region info）也是 0：

| 读取 | 结果 |
| --- | --- |
| `0x10500004`（region info） | 0（文档说这里应是 `9fa00000` 一类） |
| `0x105A9EE0`（payload 副本） | 0 |
| `0x105BE1CC`（mpool 槽） | 0 |
| `0x105AAA40` 附近 | 0，且 `+0x0c` 起出现总线错误 |

⇒ **这轮对 SCP 内部内存的读取不可信**（要么 TCM 视图/基址不在 `0x10500000`，
要么 SCP 被 recovery 复位过、region info 被清）。所以“handler 没跑”这个结论**不成立**，
暂时不能凭这些 0 下判断。

### 下次要做的第一件事（很便宜）

1. 先校准 TCM 视图：读文档里已知非零的点——`0x10502000`（loader 搬进去的固件体）、
   `0x1057d5e0`（log 互斥量）、`0x1057d700`（mbox 收件槽）。若这些也是 0，
   说明 TCM 映射/基址要先解决，之后才能用内部地址做旁证。
2. 校准后重跑：轻启动 → 立即读 `0x105BE1CC`（应含 `pkt_id=3,len=48`）与
   `0x105A9EE0`（应含 `01 04 2c 00`）：
   - 槽里有、副本没有 → 分派/中断没到 CHRE 解析器（查 SCP INTC 的 mbox2 使能）；
   - 两者都有 → 命令已被接受，下一步补序号（`0xa9edc`）；
   - 槽里没有 → mbox 写地址或映射问题。

---

## 0.24 v39 探针实测：被 ack、不锁死；TCM 别名不可用（2026-10-25 14:35）

```
[92.865633] ipi_comm chsensor: one-shot CONFIG probe, pkt_id 3, 48 bytes
[92.865651] ipi_comm chsensor: probe send ret 0
（没有 IPI 11 timeout）
```

⇒ 单包 CHRE 路径是安全的：**mbox 被 ack、SCP 没被锁死**。固件侧无日志，与 `reA.md`
“成功路径不打日志”一致，但无法确认 CONFIG 是否被接受。

**重要教训（别再踩）**：`0x10500000` 这个 AP 侧别名只在 loader 改 remap 之前等于 TCM。
这次 `0x107a5060 = 0x00070600`（remap 已设），所以 `0x10502000` / `0x10500004` /
`0x105BE1CC` 全读 0，**不能拿它当固件内存的旁证**。判断 SCP 状态只用：
`0x10730070`（bit0 gated / bit1 halt）、`0x10730080`（PC）、固件日志环、`scp_A_status`。

**恢复手法**（本轮验证过）：helper 的 `fullreset/cleartcm/loadloader/start` 是破坏性的，
会把 TCM 清掉且装不回去；正常恢复靠**重启 + `666 0 0` / `666 0 1`**。
`echo 1 > .../sram` 可以清 gated 位（`0x10730070` 从 3 → 0），但它不放固件。

下一步存在 `work/accint/reC.md`：反编译 CONFIG 的两个处理函数（0x3b00a/0x3b074），
拿到 payload 结构体布局与序号规则，再出 v40。

---

## 0.25 v40/v41：FIFO 注册成功；pkt_id 1 的 ENABLE 会把 SCP 卡住（2026-10-25 14:45）

- **v40（只发 SET_CONFIG，pkt_id 3 / action=4）**：`send ret 0`、无 IPI 超时、FIFO 头保持我们写的值
  （`rp=0 wp=0 size=2832`），**安全**。`wp` 不动是预期的——SET_CONFIG 只登记 FIFO，不设速率。
- **v41（再发一包 pkt_id 1 的 CHRE ENABLE）**：立刻出现 `IPI 11: seqno=1, state=0 ...`
  + `Error: scp_ipidev IPI 11 timeout (last done is IPI 11)`，然后 `ENABLE send ret 0`，
  FIFO `wp` 仍为 0。⇒ **pkt_id 1 的 handler 也会把 SCP 卡住**（或至少长时间不 ack）。

结论：在 `reD.md` 给出 pkt_id 1 的精确 hostintf 结构之前，**不要再发 pkt_id 1**。
当前最安全、且已经验证的镜像是 **v40**（只做 SET_CONFIG + FIFO 注册）。

---

## 0.26 v42：精确 enable 帧仍然拿不到 ack（2026-10-25 15:05）

v42 按 `reD.md` 的逐字节帧发 `pkt_id 1`（seq=1、100Hz、accel、ENABLE、crc 0x31ea7032），
结果与 v41 相同：

```
IPI 11: seqno=1, state=0 ... / Error: scp_ipidev IPI 11 timeout (last done is IPI 11)
chsensor: ENABLE send ret 0
FIFO rp=0 wp=0 size=2832 (delta 0)
hostintf lines: []          ← 固件没有走到应用速率的那一步
```

**注意**：同一轮里 `pkt_id 3 / action=4`（SET_CONFIG）是**有 ack** 的，只有 `pkt_id 1` 没有。
所以最可能不是“帧校验失败”（那样 handler 会快速返回、写 ack，并打 `hostIntfFindHandler: 1/2/3/8`），
而是 **handler 里发回包（`scp_ipi_send(1,…)` → SCP→AP mpool）时阻塞**，ack 因此永远写不出来。

### 下次的第一件事

查 SCP→AP 的 mpool 回包路径为什么会阻塞：

1. AP 侧 `IPI_IN_SCP_MPOOL_0`(12) 的 pin/buffer 是否已注册、是否被填满（`[MBOX Error]null ptr ipi_id=12` 的旧问题）；
2. `scp_legacy_handler` 是否在 enable 之前就绪（我们已注册 `IPI_CHRE` → `mpool[0]`）；
3. 固件 `scp_ipi_send(1,…)`（`0x29f6`/api[2]）是否要求 AP 先清某个 IRQ/槽位。

安全基线仍是 **v40**（只做 SET_CONFIG，不碰 pkt_id 1）。

---

## 0.27 reE：真正会卡死的是 SCP→AP 那条方向（2026-10-25 15:20）

`work/accint/reE.md` 的结论（很重要，纠正了之前的判断）：

- 回包**不在 ISR 里发**：`0x29a8` 只是把 {pkt,len,cb} 塞进深度 10 的队列（非阻塞），
  由 uSnd 任务（`0x12bcc`）执行 `while (scp_ipi_send(1,pkt,len,0))`（`0x2a0a`，**无上限**）；
- 底层 mbox 发送（`0xe244`）有两处**无限等待**：
  `0xe0c8` 的 `xSemaphoreTake(..., portMAX_DELAY)`，以及
  `0xe29e` 之后在 **`wait==0` 时无限自旋**，轮询的是 **AP 侧的 pin 状态位**
  （`minfo=0x97340+mbox*0x34`，`IPI 12`/mbox2 pin17）；
- ⇒ **只要 AP 有一条没排空的 SCP→AP 消息，整个 IPI-12 方向就永久卡住**，
  之后所有发送都堵在信号量上，没有超时、没有日志；
- ISR 本身**会**写 ack（`0xe622`），所以 AP 侧那对
  `IPI 11 timeout` + `send ret 0` **不是“卡死”的证据**（`mtk_ipi_send(timeout=0)`
  只做一次 trysend，遇到瞬时 BUSY 就打日志，从不等待 SCP 的 ack）。

**正确的观测点**（不用改代码就能看）：

```
devmem 0x107fd10c     # bit17 (0x20000) = SCP 有回包待取 / AP 没排空
devmem 0x107fd100     # bit0            = SCP 还没 ack
```

**AP 要做的事**：把 SCP→AP 的 pin（IPI 12）**排空并清中断**
（`mtk_mbox_isr` → `mtk_mbox_clr_irq`，要求 `scp_legacy_ipi_init` 已注册 pin_buf）。
建议顺序：先发 `SET_CONFIG` 并**等到 pkt_id 3 的回包**，再发 `pkt_id 1`；一次只压一个命令。
pin17 卡住时只能在 AP 侧重置 SCP，因为那个无限循环在固件里（`0xe29e`/`0x2a0a`）。

---

## 0.28 v43~v47：协议全部打通，剩下的是“固件不给样本”（2026-10-25 15:00）

这几版把 CHRE/nanoHub 这条路走完了，每一步都有实测：

| 版本 | 做了什么 | 结果 |
| --- | --- | --- |
| v40 | `SET_CONFIG`（pkt_id 3 / action 4）注册 DRAM FIFO | 回包 `{0, 4, 0}`，errCode=0 ✅ |
| v43 | enable 延后 5s（避开 `gRxBuf 0x0` 窗口） | 不再被入口丢掉 ✅ |
| v44 | 打印所有 mpool 接收 | 证明 SCP→AP 回包通路正常 ✅ |
| v45 | enable seq1 + seq2 重发 | 每次 2ms 内都有回包 ✅ |
| v46 | 反复 kick SCP logger（5 次 IPI 17） | **固件日志仍不增长**（0x2610 不动）❌ |
| v47 | DISABLE(seq1) → ENABLE(seq2) → ENABLE(seq3) | 三条都有回包，**FIFO `wp` 仍为 0** ❌ |

⇒ **软件/协议侧到此为止**：命令被接受、FIFO 已注册、回包正常，但**固件从不产生样本**。
这与固件 boot 时自己打的 `jitter check failed even 4 retries` + `acc: disable done`
完全一致——**ICM40607 没有 data-ready**。

### 硬件侧排查清单（下一轮从这里开始）

1. **EINT12 极性**：`0x1000B300` 的 bit12 是 1（level + 高有效），而 GPIO12 被芯片拉低。
   写 `POL_CLR`(0x380) 不生效（读回仍 0x81000），下一步试 `POL_SET`(0x340) 看是否反向；
   也可对照触摸 eint19（rising → pol=1）确认这对寄存器的真实语义。
2. **GPIO12 到底是不是 accel 的 INT**：固件说 `acc spi_num: 2,eint 12`；可用“临时把该 pin
   配成输出高再放开”确认是否有外部驱动。
3. **ICM40607 的 INT1 配置**：固件 boot 时 `ICM4X6XXInitConfig` 之后有
   `gy_stall_readstatus`/`RegisterCore`，但从未出现 `ICM4X6XXSample` —— 怀疑 INT1_CTRL(0x2D)
   或 FIFO 未真正使能（需要 STOCK 固件/示波器对照）。
4. **供电/时钟**：固件用 PWRAP 操作 PMIC（`[PWRAP]init done`），但 sensor 的 LDO/CLKIN
   是否被 AP 侧改动过需要对照厂商 PMIC 配置。
5. **对照原厂**：原厂 Android 上自动旋转正常（用户确认），说明硬件本身没问题——
   建议在原厂上抓 `ICM4X6XXSample` 阶段是否有 `eint` 活动（可用 LK/SCP 日志或示波器）。

### 已交付、可复用的东西

- 镜像 v25（EINT 修复）、v27（IIO 命名+矩阵）、v40（SET_CONFIG 安全基线）、v47（完整 CHRE 序列）；
- 固件逆向报告 `work/accint/reA.md`~`reE.md` + `rv.py`/`walk.py`/`xr.py`/`xrf2.py`/`nanohub_pkt.py`/`scp4.asm`；
- 本文件 §0.19~§0.28：从“MTK sensor_comm 错误假设”到“CHRE 协议全通”的完整证据链。

---

## 0.29 v49/v50：FIFO 被自己覆盖已修；EINT12 极性是开关（2026-10-25 15:10）

- **v49 修掉一个真 bug**：FIFO 原先放在 `0x8F0CA700`，与 `share_mem_config()` 的 SENS 环
  **同一地址**，被它覆盖成 `{buf=0xFFFD8, item=36}`。移到 **`0x8F400000`** 后
  读回 `size=2832`（正确）。v40~v48 的 FIFO 一直是坏头，这解释了“注册了却从无样本”。
- **v48/v50 的 EINT12 探针给出决定性数据**：
  ```
  eint12 pol=00080000 stat=00000000        ← 固件配的极性：状态永不置位
  after pol_set pol=00081000 stat=00001000 ← 改成 1：状态立刻置位
  ```
  ⇒ `pol` bit12 = 1 才是“能看见这条线”的值，而**固件配的是 0**。
  v50 起内核固定写 `POL_SET(0x340) bit12`。
- 但随后读回 `pol=00081000 stat=00000000`：**状态被清掉了** → SCP 确实在收、在 ack EINT12；
  只是它没有去读 accel 的 FIFO（`wp` 仍 0）。

⇒ 现在的阻塞点已经非常具体：**SCP 收到 EINT12 但 accel 侧不产生/不读取样本**。
下一步优先级：① 确认 SCP 侧 IRQ12 的处理是否 arm（在 accel 被 disable 后可能没挂上）；
② 结合 EINT 极性，重发一次 disable→enable，观察 `stat` 与 `wp` 的联动。

镜像：v49 `ba0f174679921ae7088eb49aa6a8be59`，v50 `59146ef2b991fd4c5d8cf9149ce8a5ff`。

---

## 0.30 固件补丁工程：DRAM 注入打通，6 个补丁生效（2026-10-25 16:45）

### 为什么不能改分区

`scp_a` = `/dev/block/259:8`（6MB，头 `88168858…tinysys-scp-RV33_A`），但 **LK 对镜像做 RSA 验签**
（`reG.md`：cert2 里存 SHA-256(头)+SHA-256(载荷)，RSASSA-PSS，MediaTek CA）。改任意字节 →
LK 在 `mt_scp.c:127` panic，表现就是“卡在开机第一屏、没有内核日志”（已实测复现）。
分区在 Linux 侧还是只读的（UFS/SCSI DATA PROTECT）。

### 可行路径：改运行时 DRAM 副本（已在 v54~v58 实现并验证）

```
scp_a（原厂签名，永不修改）
  → LK 加载到 DRAM 0x9fa00000（代码基址，offset==代码地址）
  → /sys/kernel/debug/scp-chopin/patchscp  （内核 ioremap 写 1~6 条指令）
  → fullreset + sram + start（loader 把 DRAM 重新拷进 TCM，不再验签）
  → SCP 运行补丁固件
```

`patchscp` 接受补丁组：`1`=DRDY+FIFO STREAM，`2`=再加两处 FSM 闸门+cmd6 上电+registry READY，
`3`=再加芯片侧 INT1 极性。每处**先校验原值再写**，天然幂等（第二次会报 `expected … got …` 并跳过）。

### 已验证生效的 6 个补丁（DRAM = 0x9fa00000 + 地址）

| 地址 | 原 → 新 | 作用 | 依据 |
| --- | --- | --- | --- |
| `0x26EB2` | `93e56500` → `93e5e500` | `INT_SOURCE0 \|= UI_DRDY_INT1` | reF |
| `0x286DE` | `11c5` → `0100` | 强制 FIFO=STREAM | reF |
| `0x23944` | `639e2505` → `13000000` | FSM 闸门 1 → nop | reF |
| `0x23AE0` | `63905507` → `13000000` | FSM 闸门 2 → nop | reF |
| `0x25FAA` | `29CE` → `0100` | cmd6 强制走“上电”分支（唯一写 `PWR_MGMT0` 的路径） | reH |
| `0x0A968` | `0589` → `0545` | 传感器注册表恒 READY | reH |

### 结果（v58，6 补丁全生效）

```
SCP A is ready
poll 0..19: stat=00000000  wp=0
GET_DATA: 回包 errCode=0 但 d=0
```

⇒ **accel 仍不上电、不出中断、不出样本**。说明真正的电源条件还在更深处：

- `0x25F52`（cmd 6 入口）之前还有一道 `ctx+0x68|0xA0` 的判断（reF），我们只 nop 了出口分支；
- 或芯片侧（`PWR_MGMT0` 写了但未进采样 / SPI burst 失败）；
- 或 PMIC 的 sensor 电源轨未使能（固件 PWRAP 侧，我们看不到）。

### 下一步：补丁 7

在 `0x25F52`（cmd 6 入口）把“前置条件判断”整段 nop 掉，让 `0x284BE set_power_mode(2)`
**无条件执行**——比逐条 nop 更激进，也最可能真正写出 `PWR_MGMT0`。

### 工程资产

- `patchscp`（分组补丁，v55+）、`scp_a_stock.img`（6MB 原厂备份，`work/accint/`）
- 主机⇄设备 SSH 已打通（`work/accint/sshcfg`，公钥认证），调试可全程远程完成
- 逆向报告 `reA.md`~`reH.md`，工具 `rv.py`/`walk.py`/`xr.py`/`xrf2.py`/`dmp.py`/`drv.py`/`nanohub_pkt.py`
- 镜像：v52（无补丁基线）、v54（首个 patchscp）、v57（4 补丁）、**v58（6 补丁，当前最全）**

---

## 0.31 加速度计打通：ICM40607 出数（2026-10-26 15:00）

**结果**：`chopin-accel` 的 `in_accel_{x,y,z}_raw` 输出真实数据
（`390/55/9943`、`612/-354/9935`，z≈9.94 m/s²，单位 mm/s²，`in_accel_scale=0.001`）。

完整链路（全部实测）：

```
SCP 固件（7 个 DRAM 补丁：DRDY / FIFO STREAM / 2×FSM 闸门 / 强制上电跳转 /
          registry READY / boot 时 PWR_MGMT0=0x03）
  → SET_CONFIG 注册 DRAM FIFO @0x8F0CA700（FIFOSize 0xFFFC8，44 的倍数）
  → SET_CUST + SET_TIMESTAMP + batch/enable（nanoHub，pkt_id 1）
  → 固件每 10ms 一条 action=6 通知（+8 是 currWp，字节偏移）
  → 内核每 50ms 轮询 FIFO 头 + 24 字节/44 字节记录
  → sensorhub_push_sample() → IIO chopin-accel
```

关键坑（都已修）：

| 现象 | 原因 | 修法 |
| --- | --- | --- |
| `in_accel_*_raw` 恒 `-EAGAIN` | 记录第 0 字节不是 sensor_type（u32 读作 0x00010000），`if (type==1)` 把样本全丢 | 无条件按 accel 推送 |
| `buffer/enable` 报 `EINVAL` | `postenable` 去启动已失效的 MTK 控制通道，失败即拒绝 | 控制失败只告警、继续 |
| 读不到样本、扫描全 0 | 记录在 `base+0x10+wp`（`wp` 是字节偏移），不是固定区 | 按 `wp` 读；`fifo` debugfs 节点自动解码 |
| 系统卡死、串口无响应 | 每 20ms `ioremap(1MB)`+`iounmap` + 日志刷屏 | 缓存映射、50ms 轮询、日志降级 |
| `scale` 为 1.0 | 固件给的是 mm/s² | accel 固定 `gain=1000` → `scale=0.001` |
| 间歇性无数据 | SCP 自己的 accel 初始化竞态 | v79 起每 5 秒重跑全套序列（SET_CONFIG/SET_CUST/SET_TIMESTAMP/batch/enable） |

镜像：**v80-alpine `e0abefd2c6fd090fe5bea077b62df67a`**（稳定出数）。
`v81` 起 alpine rootfs 内含 `iio-sensor-proxy 3.9-r2`（体积涨到 96MB / 压缩 35MB）。

### 间歇性：实测规律（重要）

加速度计**能出数**，但固件只在部分 boot/部分重试后开始采样。实测数据：

| 观察到的情况 | 结论 |
| --- | --- |
| 同一序列连发 5 次：`try1 EAGAIN, try2 EAGAIN, try3 9964, try4 9979, try5 9959` | 固件需要 **3~6 次**启动序列才开始采样，一旦开始就持续出数 |
| `wp` 从 0 变为非 0 后不再变化 | 判定“已开始”的唯一可靠信号是 **FIFO 写指针 `wp`** |
| 每次发送都伴随 `IPI 11 timeout`（约 15 条/轮） | 每次重试都有 mbox 代价，**重试次数必须严格有界** |
| 重试 5 次以上 → 内核崩溃 | IPI 风暴会拖垮系统（非传感器本身） |

已实现并验证的机制：

* `echo 1 > /sys/kernel/debug/chsensor/retry` —— **手动触发一次启动序列**（v88 起）；
* v89 起自动重试最多 6 次、间隔 4 秒，`wp` 一旦非 0 即永久静默。

仍待解决：部分 boot 即使重试 6 次也不采样（可能与固件内部 accel 初始化的一次性失败有关）。

### 用户态（下一步）

```sh
# alpine:
rc-service dbus start            # 或手动 dbus-daemon --system
/usr/libexec/iio-sensor-proxy &  # 然后 monitor-sensor 看方向
monitor-sensor
```

---

## 1. 为什么必须走 SCP

- 加速度计 / 环境光挂在 **SCP 自己的 I2C** 上，AP 侧 DT 里没有这些节点，
  Linux i2c 驱动碰不到；
- AP 需要的是：启动 SCP + 共享 DRAM 里的 FIFO + mbox/IPI 收事件；
- 所以"驱动"其实在 SCP 固件里（nanohub/accelhub），AP 只做传输和 IIO 前端。

---

## 2. 证据链（都是可复核的事实）

| 结论 | 证据（文件 / 日志） |
| --- | --- |
| SCP 共享窗口 = **0x8f000000 / 0x500000** | 内核启动日志：`[SCP]scp_reserve_mem_of_init 0x000000008f000000 0x0000000000500000`；`OF: reserved mem: 0x8f000000..0x8f4fffff reserve-memory-scp_share`（`work/boot-log.txt`） |
| 厂商把 `scp_mem_base_phys` 写进 `DRAM_RESV_ADDR` | `n10p/vendor/.../scp/mt6885/scp_awake.c: scp_sys_reset_ws()`；`scp/v02/scp_helper.c: reset_scp()`（注释：*"to let scp setup MPU"*） |
| 0x9fa00000 是**镜像位置**，不是共享窗口 | LK 日志：`mblock-22-SCP-reserved 0x9fa00000..0x9fefffff`；region info: `ap_loader_start 0x9fa00000 / ap_firmware_start 0x9fa02000` |
| LK 留下的 region info（TCM+4，13 字 = 0x34） | 见 §3.1；来自 LK 自己的日志 `work/expdb*/exp*.txt`：`scp_region_info->ap_dram_start 0x9fc00000`、`ap_dram_size 0x5a880`、`ap_dram_backup_start 0x9fcb5800` |
| 固件日志控制块在 **share + 0x1ca700** | `scp_reservedmem_define.h` 顺序：SECDUMP(0)→VOW(0xca700)→SENS(0x100000)→**SCP_A_LOGGER**，累计 0x1ca700；`scp_logger.c: scp_logger_init()` 往这块写 `base=0x504c5402`、`enable=0`、r/w pos=0 |
| 固件的那把锁 = **log 互斥量** | 见 §3.3：`0x3fc82` 是 SCP 的 printk（vsnprintf → 加锁 0x7d5e0） |
| 厂商只放 core0 | `reset_scp()` 只写 `R_CORE0_SW_RSTN_CLR`；`mt6893.dts` 只有 `core_0="enable"`，没有 `core_1` |
| 厂商在启动前一定先建日志控制块 | `scp_init()`：`scp_region_info_init()` → `scp_reserve_memory_ioremap()` → `scp_logger_init()` → `reset_scp(SCP_ALL_ENABLE)` |
| 厂商 boot 的启动是**轻**的 | `reset_scp()` 只有 `scp_reset_wait_timeout()` + 写两个寄存器 + `add_timer(ready)`，**不清 TCM、不重载固件**；wiping 只在 `scp_sys_full_reset()`（`scp_sys_reset_ws()` 恢复路径）里 |
| LK 很可能已经在跑 SCP | `SENSOR-HANDOFF.md`：无驱动时 `0x10730070=0`（core0 未 gated/halt）、`0x10500000=0x03c0006f`（TCM 里是 loader） |

### 2.1 上一轮"已排除"里，哪些其实没被验证过

- **DRAM 预留地址**：完全没被怀疑。helper 里 `scp_start()` 直接写
  `SCP_DRAM_BASE`（0x9fa00000），而这个宏的真实含义是"LK 放 scp.img 的 mblock"。
  宏名骗了我们三天。
- **dramrestore / dramzero**：这两个**确实跑了**，而且和厂商
  `scp_sys_full_reset()` 等价（zero 2×size 再 copy size；起始地址是 region info
  的 0x9fc00000，落在 helper 的 0x9fa00000..0x9fefffff 映射内）。这条排除是有效的。
- **"logger 区全 0xff" 被解释成"固件停在极早期"**：其实是**AP 从没写过那块**。
  厂商路径下它应该是 `504c5402` 开头；0xff 说明 `scp_logger_init()` 一次都没跑过
  （因为 `core_0="disable"` 让 `scp_init()` 在 logger 之前就 `goto err` 了）。

### 2.2 由此得到的最大行为改变：先别再"冷启动" SCP

`0x10730070` 读 0 表示 core0 没被 gated、没 halt —— 也就是**核是放出来的**。
如果它一开始就在跑，那么我们过去的每一次 `start`（assert → 清 TCM → 重载 →
release）都是在**中断一个正在运行的 SCP，然后从零重来**，而且重来时用的还是
错误的共享窗口、没有 logger 控制块。固件在第一次打印就卡住，和这个场景完全一致。

所以本轮的默认改成：**boot 时不动核**（`mediatek,no-auto-start`），
由人决定走哪条：

| 命令（`/sys/class/misc/scp/scp_reset`） | 做什么 | 用在哪 |
| --- | --- | --- |
| `echo "666 0 0" > .../scp_reset` | 只把自动恢复预算清零（`scp_reset_counts=0`） | 先执行，防止后面提前 3s 超时自动升级成重启动 |
| `echo "666 0 1" > .../scp_reset` | **轻启动**：写共享窗口 + 放 core0 + armed 3s ready timer，TCM 不动 | = 厂商 boot 行为，默认先试这个 |
| `echo "666 1 1" > .../scp_reset` | **重启动**（恢复路径）：assert 两核 → 清 TCM/重载 loader/写 region info/恢复 DRAM → 写共享窗口 → 放 core0 | 轻启动失败、或要复现冷启动时的行为 |

---

## 3. 固件分析（上一轮成果 + 本轮修正）

### 3.1 region info（TCM + 4，固件/loader 与 AP 的契约）

```
+0x00 ap_loader_start        0x9fa00000    +0x04 ap_loader_size          0x00002000
+0x08 ap_firmware_start      0x9fa02000    +0x0c ap_firmware_size        0x00143c00
+0x10 ap_dram_start          0x9fc00000    +0x14 ap_dram_size            0x0005a880
+0x18 ap_dram_backup_start   0x9fcb5800    +0x1c struct_size             0x00000034
+0x20 scp_log_thru_ap_uart   0             +0x24 TaskContext_ptr         0
+0x28 scpctl                 0             +0x2c regdump_start           0
+0x30 regdump_size           0
```

（上一轮把 `TaskContext_ptr` 认成"TCM 0x7d214 连续 16 字全 0"是错的：0x7d214 在
固件自己 `memset(0x7cc14, 0, 0x600)` 的尾巴上，和 region info 无关。）

**谁写它**：LK。所以 helper 在 probe 时读出来就是 LK 的值，`reginfo` 再写回去。
`cleartcm` 会把它清零，所以 `start`/`fullreset` 必须先 `reginfo`。

### 3.2 loader 怎么把固件搬进 TCM（重要）

`cleartcm` 后只需拷 8 KiB loader（TCM 0..0x2000），**固件体是 loader 自己搬的**：

```asm
; 0x310  loader 主流程
jal  0x2d0            ; 写 remap 寄存器 0x700a5060 = (ap_loader_start>>28)<<16 | 0x600
                      ;   我们的值 = 9<<16|0x600 = 0x00090600
lw   t4, 0xC(t0)      ; src  = ap_firmware_start  = 0x9fa02000
lw   t1, 0x10(t0)     ; size = ap_firmware_size   = 0x143c00
lw   t0, 0x8(t0)      ; dst  = ap_loader_size     = 0x2000
0x2fc:                ; src = (src & 0x0fffffff) | 0x20000000  -> 0x2fa02000
copy: lw t4,0(src+i); sw t4,0(dst+i)
```

两点推论，都可用于上机验证：

1. **`ap_loader_start` 必须正确**，否则 loader 算出的 remap 窗口是错的，
   固件体就搬到垃圾里。⇒ `devmem 0x107a5060` 在 start 之后应为 `0x00090600`
   （AP 侧 alias，SCP 侧是 0x700a5060）。
2. **`tcm[0x2000]` 在 start 之前是 0、之后应该非 0**（固件体第一段）。
   helper 的 `audit` 里直接有 `tcm[002000]`（另外 `tcm[000004]` 是 region info）。

### 3.3 固件停在哪：SCP 的 printk

```
0x3fc82  scp_print(...)             ; 真实函数，非符号
  ...
  jal 0x12704                       ; 读 logger 全局 *(0x9e628) / *(0x9e604)
  bnez a0, 0x3ef88                  ; 走日志
  jal 0x115ae                       ; 或计数
0x3ef88:
  lui a0,0xad ; amoadd.w (0xad2e8)  ; 引用/计数
0x3ef96:
  lr.w  a0, (s0)                    ; s0 = 0x7d5e0（调用者传入，见 0x3fce4）
  bnez  a0, 0x3ef96                 ; 自旋直到 *s0 == 0
  sc.w  a0, -1, (s0)                ; 置 -1 = 持有
```

`0x3fce4` 处 `lui a0,0x7d; addi s4,a0,0x5e0`，紧接着 `jal 0x3ef6c` —— 所以
**0x7d5e0 是 log 互斥量**，不是 mbox 锁。它下面 0x7d600 / 0x7d700 是两核的
mbox 收件槽（`0x3eb8c`：`csrr mhartid` 后选 0x7d600 / 0x7d700）。

⇒ core0/core1 同时停在 0x3ef96/0x3ef9a = **两核都在打日志、抢同一把日志锁**。
这和"固件在启动极早期"是相符的：它一进来就想打第一行 log。

### 3.4 反汇编工具（别丢）

```sh
llvm-objcopy-19 -I binary -O elf32-littleriscv \
    --rename-section .data=.text,code,alloc scp_code.bin scp_code.elf
llvm-objdump-19 -d --triple=riscv32 --mattr=+m,+a,+c,+zicsr scp_code.elf
```
（`--mattr` 不加会满屏 "unknown"。产物在 `work/scp2.asm`。）

---

## 4. 本轮代码改动

### 4.1 `drivers/misc/mediatek/scp-chopin.c`（helper，重写）

| 改动 | 说明 |
| --- | --- |
| **共享窗口单独解析** | `scp_find_share()`：`mediatek,scp-share-base` DT 属性 → `of_reserved_mem_lookup("mediatek,reserve-memory-scp_share")` → 当前 `DRAM_RESV_ADDR`（合法范围）→ 0x8f000000/0x500000。日志会打印来源 |
| **`DRAM_RESV_ADDR/SIZE` 用共享窗口** | 不再用 0x9fa00000。0x9fa00000 只留给 `loadloader` 当镜像源（从 region info 读更优先） |
| **`start` = 厂商整段序列** | LK 寄存器表 → `scp_enable_sram()` → 只置 SYS_EN（不动 HIGH_EN，那需要活 PLL）→ 两核置复位 → `GPR_REBOOT_OK` → 清 TCM / 拷 loader / 写 region info / 恢复 DRAM → 写共享窗口 → 只放 core0 → 打印状态 |
| **`scp_enable_sram()`** | 位走查照抄厂商 `mt6885/scp_awake.c`（CPU0/CPU1 SRAM_PD、L1、TCM TAIL）。注意：这个函数在厂商树里**定义了却从未被调用**——mt6893 的 SRAM 是 LK 上电的。这里保留成显式步骤（`echo 1 > sram`），不作为"修好"的理由 |
| **core1 默认不释放** | 和厂商一致；`echo 1 > core1` 可打开 |
| **`audit`** | 只读：clk / sram_pd / core0 全寄存器 / core1 关键寄存器 / TCM 关键 32 字（含 0x7d5e0、0x7d600、0x9e600、0xad2e8）/ share 关键 10 处（含 **0x1ca700 日志控制块**） |
| **`dramrestore`** | 用 region info 的 ap_dram_start/backup 动态 ioremap（不再依赖固定的 0x9fa00000 窗口），跳过条件写清楚 |

### 4.2 DT：`mt6893-xiaomi-chopin-scp-dbg.dtsi`

```dts
core_0 = "enable";
mediatek,no-auto-start;
```

### 4.3 `drivers/misc/mediatek/scp/rv/scp_helper.c`

1. 新增 `mediatek,no-auto-start`：`scp_init()` 走到 `reset_scp(SCP_ALL_ENABLE)` 时
   跳过，只打一行提示。其余初始化（region info / reserve mem / logger / recovery /
   IPI 注册 / sysfs）全部照厂商跑完。
2. `scp_reset` sysfs 的第二字段现在有意义了：
   `666 1 n` = 重启动（恢复路径，原有行为），
   `666 0 n` = **轻启动**（`reset_scp(SCP_ALL_ENABLE)`，厂商 boot 行为），
   `n` = `scp_reset_counts`（自动恢复预算；轻启动时给 0 可以避免 3 s ready 超时
   自动升级成重启动）。

### 4.4 `port/tools/board-scp-audit.sh`（新）

打镜像时由 `mkinitramfs.sh` 安装到 `/usr/bin/board-scp-audit.sh`：

```sh
sh /usr/bin/board-scp-audit.sh          # 只读报告
sh /usr/bin/board-scp-audit.sh start    # 用 helper 启动，再报告
sh /usr/bin/board-scp-audit.sh start2   # 连 core1 一起放
```

---

## 5. 上机顺序（照着做，把 4 个数贴回来）

### 5.0 刷机

```sh
fastboot flash boot_a out-scp-v2/boot-stockdt.img
fastboot reboot
```

串口 **921600**（不是 115200；old doc 里的 115200 是旧的）：
`minicom -D /dev/ttyACM0 -b 921600`。

### 5.1 开机后先看驱动有没有把东西准备好，以及 SCP 是不是本来就活着（只读）

```sh
dmesg | grep -i "\[SCP\]" | head -40
cat /sys/kernel/debug/scp-chopin/status
cat /sys/kernel/debug/scp-chopin/audit
devmem 0x10730044          # DRAM_RESV_ADDR（LK 留的应该是 0x8f000000）
devmem 0x10730048          # 0x500000
devmem 0x10730070          # core0 status：bit0 gated，bit1 halt（0 = 在跑）
```

期望看到：

| 位置 | 期望 | 含义 |
| --- | --- | --- |
| dmesg `[SCP] LK region:` | `loader=9fa00000/2000 firmware=9fa02000/143c00 dram=9fc00000/5a880` | 驱动读到了 LK 的 region info |
| dmesg `[SCP] auto start disabled` | 有 | 没有在 initcall 里启动 |
| status `share` | `8f000000/500000 (reserved_mem)` | 共享窗口解析正确 |
| audit `dr[001ca700]` | **`504c5402 00000018 00000000 ...`** | **logger 控制块这次真的建了**（上一轮是 ffffffff） |
| audit `tcm[07d5e0]` | **`00000000`** | 冷启动前固件没卡在 log 锁上（上一轮这里是 ffffffff） |
| `devmem 0x10730044` | `08f00000` | LK 留在寄存器里的正确值（若为 0 说明靠 DT/fallback，也无妨） |
| `devmem 0x10730070` | `0`（在跑）或 `2`（halt） | **这一步决定走 5.2 还是 5.3**：0 = core0 是放出来的（LK 已经启动过），2 = 停着 |

### 5.2 轻启动（推荐先做：就是厂商 boot 做的事，不清 TCM）

```sh
dmesg -c >/dev/null
echo "666 0 0" > /sys/class/misc/scp/scp_reset   # 先解除自动恢复，避免 3s 后自己升级成重启动
echo "666 0 1" > /sys/class/misc/scp/scp_reset   # 轻启动：写共享窗口 + 放 core0
sleep 3
dmesg | tail -60
```

期望依次出现：

```
[SCP] scp_reset_store 666 0 1
[SCP] reset timeout, still reset scp      <- 核本来在跑时会有，正常
[SCP] notify blocking call                <- 固件发回 READY（关键！）
sensor_ready: sensor hub reported ready
```

然后：

```sh
cat /sys/kernel/debug/scp-chopin/status     # pc0 / pc1 / tcm[07d5e0]
cat /sys/kernel/debug/scp-chopin/audit
```

### 5.3 重启动（轻启动不行时；= 崩溃恢复路径，会清 TCM 重载）

```sh
dmesg -c >/dev/null
echo "666 1 1" > /sys/class/misc/scp/scp_reset
sleep 3
dmesg | tail -80
devmem 0x107a5060          # 期望 0x00090600：loader 读了 region info 并设好窗口
devmem 0x10502000          # 期望非 0：固件体被 loader 搬进 TCM 0x2000
```

判定树（5.2 / 5.3 通用）：

| 现象 | 结论 / 下一步 |
| --- | --- |
| 出现 `notify blocking call` + `sensor hub reported ready` | **SCP 活了**，直接进 §7（IIO 拿数据） |
| 没有 ready，但系统还活着 | 看 `status`：`pc0=0x3ef96` 且 `tcm[07d5e0]=ffffffff` = 还是 log 锁。把 `audit` 全贴出来，再试另一边（5.2 ↔ 5.3） |
| 出现 `notify blocking call` 但 sensor hub 不应答 | 卡点移到 IPI/sensor 层，查 mbox 表（§3.3）、`cat /sys/class/misc/scp/scp_A_status`、`scp_A_reg_status` |
| **整机静默卡死（串口无输出）** | 总线被 SCP 挂住。重启后会由 initramfs 打印上一次内核日志（`init: kernel log recovery` + `last 80 lines`），把它贴出来，再走 §5.4 |

### 5.4 用 helper 路径启动（第二条独立的路，用来对照）

```sh
sh /usr/bin/board-scp-audit.sh          # 只读
sh /usr/bin/board-scp-audit.sh start    # 冷重载（清 TCM + 拷 loader + 正确共享窗口）
sh /usr/bin/board-scp-audit.sh start2   # 连 core1 一起放
```

helper 和驱动路径的差别：helper **不建 logger 控制块**，也**不 arm ready timer**
（那是 `mediatek,scp` 驱动的活）。所以 helper 能把固件跑起来，但接不上 sensor
handshake——它只用来回答"启动寄存器序列本身对不对"。

### 5.5 都不行时，按这个顺序加信息

1. `audit` 里 `dr[001ca700]` 到底是不是 `504c5402`（logger 契约）；
2. `devmem 0x107a5060` 是不是 `0x00090600`（loader 有没有跑对）；
3. `devmem 0x10502000`（TCM 0x2000，固件体有没有搬进去）；
4. `status` 的 `pc0/pc1` + `tcm[07d5e0]`（停在哪、锁状态）；
5. 两核 PC 用 devmem 再看一遍（`0x10730080` / `0x10740080`）并记录
   `0x10730070` / `0x10740070`。

---

## 6. 构建 / 打包（固定流程，别省步）

内核用**内嵌 DTB**：`arch/arm64/kernel/chopin-dtb.S` 用 `.incbin` 嵌入
`arch/arm64/kernel/chopin-board.dtb`。改 DTS 后必须三连：

```sh
make -C git/chopin-linux-7.3 O=$PWD/build/chopin-7.3 ARCH=arm64 \
     CROSS_COMPILE=aarch64-linux-gnu- -j16 dtbs
cp build/chopin-7.3/arch/arm64/boot/dts/mediatek/mt6893-xiaomi-chopin.dtb \
   build/chopin-7.3/arch/arm64/kernel/chopin-board.dtb
make -C git/chopin-linux-7.3 O=$PWD/build/chopin-7.3 ARCH=arm64 \
     CROSS_COMPILE=aarch64-linux-gnu- -j16 Image
```

打包（`mkinitramfs.sh` 会顺带把 `board-scp-audit.sh` 装进 initramfs）：

```sh
KERNEL=$PWD/build/chopin-7.3/arch/arm64/boot/Image \
DTB=$PWD/build/chopin-7.3/arch/arm64/boot/dts/mediatek/mt6893-xiaomi-chopin.dtb \
STOCK_DTB=$PWD/port/out/stock-base.dtb CHOPIN_ROOTFS=userdata \
port/tools/build-images.sh out-scp-v2
```

只刷 `boot-stockdt.img`（LK 用原厂 DTB）；**不要刷 `boot-owndtb.img`**（LK 会崩）。

---

## 7. SCP 活了之后的剩余工作（自动旋转本体）

1. **确认 IIO 有数据**：现在设备名还是 xaga 的（`xaga-accel` / `xaga-als`，
   见 `sensor/2.0/sensorhub/sensorhub_iio.c:451`）：
   ```sh
   cat /sys/bus/iio/devices/iio:device*/name
   cat /sys/bus/iio/devices/iio:device*/in_accel_x_raw
   ```
   三轴静止时 z 应为 ±9.8 量级（`in_accel_scale` 换算）。
2. **改名 + mount-matrix**：改成 `chopin-accel`/`chopin-als`，并实测量出
   加速度计朝向（贴一层轴 = 设备坐标 → 屏幕坐标）。DT 里已有
   `sensor-hub { mount-matrix = "1","0","0", ... }` 的挂钩，但要确认
   `sensorhub_iio.c` 真的读它并作用到 `_raw` 上（现在多半没读）。
3. **userspace**：`iio-sensor-proxy`（读 IIO，暴露 orientation/als）→ 合成器
   （GNOME 直接吃 iio-sensor-proxy；wlroots/Phosh 需要 sensor proxy 的 DBus 接口；
   纯 Xorg 没有现成的，得自己写个小 daemon 读 IIO 再 `xrandr --rotate`）。
   这一步跟内核无关，但缺了它"自动旋转"不成立，别到最后才发现。
4. 自动亮度：`xaga-als` 的 `in_illuminance_raw` → `iio-sensor-proxy` →
   `gnome-settings-daemon`/`iio-sensor-proxy` 的 brightness 支持，或自写小 daemon。

---

## 8. 镜像 / 命令速查

```
out-scp-v2/boot-stockdt.img        md5 73317455a00cc629d1a14686262e3466
out-scp-current/boot-stockdt.img   md5 f840a84469ea9dc126c3d53d2b8cc9e0（上一轮）
```

```sh
# 厂商驱动路径（推荐）
echo "666 0 0" > /sys/class/misc/scp/scp_reset   # 解除自动恢复（可选）
echo "666 0 1" > /sys/class/misc/scp/scp_reset   # 轻启动（厂商 boot 行为）
echo "666 1 1" > /sys/class/misc/scp/scp_reset   # 重启动（恢复路径，清 TCM）
cat /sys/class/misc/scp/scp_A_status
cat /sys/class/misc/scp/scp_A_reg_status
cat /sys/class/misc/scp/scp_A_get_last_log

# helper
cat  /sys/kernel/debug/scp-chopin/status
cat  /sys/kernel/debug/scp-chopin/audit
echo 1 > /sys/kernel/debug/scp-chopin/start      # 整段厂商序列
echo 1 > /sys/kernel/debug/scp-chopin/sram       # 只给 SRAM 上电
echo 1 > /sys/kernel/debug/scp-chopin/fullreset  # 清 TCM + 拷 loader + region info + DRAM
echo 1 > /sys/kernel/debug/scp-chopin/core1      # 之后 start 会连 core1 一起放

# 寄存器（注意：是 /usr/bin/devmem，busybox 没有这个 applet）
devmem 0x10730044   # DRAM_RESV_ADDR（应为 0x8f000000）
devmem 0x10730048   # DRAM_RESV_SIZE（0x500000）
devmem 0x107a5060   # loader 的 remap 窗口（应为 0x00090600）
devmem 0x10730080   # core0 PC
devmem 0x10740080   # core1 PC
devmem 0x10730070   # core0 status（bit0 gated，bit1 halt；0 = 在跑）
devmem 0x1057d5e0   # log 互斥量
devmem 0x1057d600   # mbox 收件槽 core1
devmem 0x1057d700   # mbox 收件槽 core0
devmem 0x10502000   # 固件体起点（start 前 0，之后非 0）
```

---

## 9. 风险 / 未验证

- §4.2/4.3 让 `mediatek,scp` 在 boot 时跑完除启动以外的全部初始化。以前
  "一启动就挂板"的那次是**最小 chopin 驱动**（`53fded01a`：只写时钟/DRAM 窗口/
  reboot_ok/放核，没有 logger、窗口还是 0x9fa00000），和现在的路径不是一回事；
  但驱动初始化本身（logger/recovery/IPI）仍然没在这块板上跑过。如果 boot 就挂，
  把上一次日志贴出来，退路是 `core_0="disable"` + helper。
- 轻启动会 arm 3 s 的 ready 超时，超时后**自动**升级成重启动（清 TCM）。所以
  先 `666 0 0` 把 `scp_reset_counts` 清零再看，是不被"自动升级"打断的关键。
- 共享窗口的解析依赖 `reserved-memory` 节点在**内嵌 DTB 替换之后**还能查到。
  我们的内核是"LK FDT 先跑 reserved-mem，再换成内嵌树"，所以
  `of_reserved_mem_lookup()` 可能查不到，helper 会退回 fallback 0x8f000000
  （就是正确值）；`status` 里的 `(fallback)` 字样就是提示，不代表错误。
- 0x8f000000 这个值本身来自本轮启动日志和厂商代码，**没有在实机上验证过
  SCP 侧是否真的按它建映射**。§5.1 的 `devmem 0x10730044` 就是用来确认
  LK 是否也这么写。

---

## 10. 备注

- `sensorhub` 的 IIO 设备名还是 xaga 的，拿到数据后要改成 chopin 并校准
  mount-matrix；
- 之前删掉的 `scp-v02` 实验目录没有恢复，树保持可编译；
- 本轮内核仓库改动：`scp-chopin.c`、`scp/rv/scp_helper.c`、
  `mt6893-xiaomi-chopin-scp-dbg.dtsi`；port 仓库改动：
  `tools/board-scp-audit.sh`、`tools/mkinitramfs.sh`、本文档。
