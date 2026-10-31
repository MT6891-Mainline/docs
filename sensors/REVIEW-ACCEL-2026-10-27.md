# 加速度计驱动逻辑复核 — 2026-10-27

> 前置：`port/HANDOFF-ACCEL-2026-10-27.md`（协议/固件事实仍然有效）
> 证据依据：`work/accint/reK.md`、`work/accint/reJ.md`、`port/AUTO-ROTATION-STATUS.md` §0.29
> 本文件只讨论**内核侧逻辑**，不重复固件逆向。

---

## 0. 一句话结论

驱动逻辑**确实有问题**，而且"不稳定"和"崩溃"是两个独立的 bug 链：

```
不稳定 = DRAM FIFO 的 16 字节头被我们自己的 share_mem_init() 覆盖成 36 字节几何
         → 第一次回绕后 44 字节记录全部错位 → x/y/z 是"两条相邻样本的混合"
         → 数值仍像重力、仍在变，但永远是错的（所以看起来"能读数但不稳"）

崩溃   = 三个独立的 UAF/竞态，跟重试次数正相关（重试越多，命中概率越高）
```

**关键推翻**：`IPI 11 timeout` 不是"重试次数"的函数，也不是崩溃主因，见 §3。

---

## 1. 头号问题：FIFO 头和 SENS 共享环是同一块内存（未修，需你决策）

### 事实链

| 事实 | 出处 |
| --- | --- |
| `SENS_FIFO_BASE = 0x8F0CA700`，即 `phys(SENS_MEM_ID)` | `ipi_comm.c:154`（改动前 145） |
| 同一地址同时被 `share_mem_init()` 当成 SENS 环形缓冲的头部 | `share_memory.c:204-210` |
| `share_mem_init()` 写的是 `{rp=0, wp=0, buffer_size, item_size}` | `share_memory.c:204-210` |
| `item_size = sizeof(struct share_mem_data) = **36**` | `share_memory.h:24-31` |
| `buffer_size = ((blocksize-16)/36)*36 = **0xFFFD8**`（1 MB 块） | `share_memory.c:207-210` |
| 而驱动的 FIFO 需要 `FIFOSize = 0xFFFC8`（44 的倍数） | `ipi_comm.c:249` |
| `share_mem_config()` 由 `sensorhub_prime()` **无条件调用** | `sensorhub_main.c:557` |
| 实测被覆盖后的头：`{buf=0xFFFD8, item=36}` | `AUTO-ROTATION-STATUS.md:1070`（v49） |

### 为什么这必然产生"看起来正常但是错误"的数据

固件的写入路径（reK.md §2，`0x3accc`–`0x3ad76`）：

```
wp_new = remu(wp_old + len, FIFOSize)      ; 0x3ad72
```

固件**用 44 字节步进**，但 `FIFOSize` 被换成了 `0xFFFD8`：

```
0xFFFD8 = 1048536 = 44 × 23830 + 16       ← 余 16
驱动侧绕回点   = 0xFFFC8 = 1048520 = 44 × 23830
固件侧绕回点   = 0xFFFD8
```

所以**第一次回绕之后**：

* 固件把 `wp` 从 `0xFFFD8` 折回 0，但它在 `0xFFFD8-44` 处又写了一条记录，于是最后那 16 字节是半条记录；
* 驱动按 44 步进，绕回点差 16 字节，`0x10 + wp` 从此永久错位 16 字节；
* 错位 16 字节 = 一条记录里 **`x` 取到了上一条的 `y`、`z` 取到了下一条的 `x`**。

这就是"能读数、z 仍约 9940（因为重力是模长最大的分量，混合后量级不变），但方向乱跳/不稳定"的机制。1 MB / 4400 B/s ≈ **4 分钟后**开始错位，所以短测试看起来是好的——这也解释了为什么"有时候能出数、跑一会儿又不对"。

### 两个可选修法

| 方案 | 做法 | 风险 |
| --- | --- | --- |
| **A（推荐，改动最小）** | 让 MTK 的 SENS 环别再初始化：`sensorhub_main.c:416 sensorhub_shm_cfg()` 直接返回 `-ENODEV`，`share_mem_config()` 会跳过它（`share_memory.c:311-314`）。这条路径本来就不通（固件对 MTK `sensor_comm` 从不回 ack），跳过它没有任何功能损失，而 FIFO 头从此只由我们写 | 低。唯一影响是固件不再收到 SENS ring 的公告；如果实测发现"不公告就不采样"，改回即可 |
| B | 把 FIFO 挪到 share window 里没人用的偏移（v49 试过 `0x8F400000`，头不再被覆盖），再 `SET_CONFIG` 报新 base | 中。`0x8F400000 + 1 MB = 0x8F500000` 正好是 share window 末尾，最后一条记录会越界 8 字节；且不确认该偏移有没有别的 mblock |

**我这次没有动 A**，因为它是"关掉 MTK 协议栈的一部分"，属于设计决策，需要你确认固件对 SENS 公告没有依赖。作为过渡，驱动现在会**检测**这个坏头并自己重写回去（见 §2.1），所以至少不会再往用户态送错位数据。

### 顺带确认：`share_mem_init()` 还会 `wp = 0; rp = 0`

`share_memory.c:204-205` 把写指针清零。如果它发生在采样已开始之后，`wp` 会**倒退**，老代码的 `chsensor_fifo_last` 只比较相等、只做 `+44` 不回退，于是永久失步、且会把整圈旧 DRAM 当新样本推上去。新代码按环形语义算前向距离，`delta > size/2` 判为倒退并 resync（`ipi_comm.c:341-347`）。

---

## 2. 已修的问题

编号沿用交接文档 §5 的坑表，新增项标 `新`。

### 2.1 `新` FIFO 头几何没有校验 → 错位数据直接进 IIO

| 项 | 内容 |
| --- | --- |
| 位置 | `ipi_comm.c:311-330`（新） |
| 老逻辑 | `if (!size \|\| wp >= size \|\| wp == last) goto out;` —— 只要 `size` 非 0 就信 |
| 根因 | 见 §1：`size` 可能变成 36 的倍数 |
| 修法 | 校验 `size % 44 == 0` 且 `wp < size`；不满足就打印并 `chsensor_fifo_write_header()` 重写为 `0xFFFC8`（按 500 ms 限流，避免和抢占者反复对写）。另外每次读到 `wp` 都用环形距离算增量，`delta > size/2` 判为倒退并 resync，只消费整条记录 |

### 2.2 `新` `chsensor_enable_work` 读 SCP DRAM 没有 `is_scp_ready()` 保护 → AP oops

| 项 | 内容 |
| --- | --- |
| 位置 | 老代码 `ipi_comm.c:365-370` |
| 老逻辑 | `if (chsensor_fifo_map) { u32 wp = readl(chsensor_fifo_map + 0x04); ... }` —— **无条件读** |
| 根因 | 这是交接文档坑 #6 的**同类问题但漏了一处**：`chsensor_fifo_work` 加了检查，`chsensor_enable_work` 没加。SCP 在 `fullreset` / recovery 期间被复位时读到死窗口 → `Unable to handle kernel paging request` |
| 修法 | 整个 work 先 `if (!is_scp_ready(SCP_A_ID))` 早退并每 2 s 重试（该路径不产生任何 IPI，轮询免费）；核心回来后 `config_done/attempt` 清零，从 SET_CONFIG 重放（固件复位后 FIFO 指针已丢失） |

### 2.3 `新` 对**已入队**的 delayed work 反复 `INIT_DELAYED_WORK` → timer wheel 链表损坏

| 项 | 内容 |
| --- | --- |
| 位置 | 老代码 `ipi_comm.c:412`（poll）、`419`（getdata），每次 `attempt >= 2` 都执行 |
| 根因 | `INIT_DELAYED_WORK` 会重写 `work->data` / `timer.entry`。`chsensor_getdata_dw` 自己每 500 ms 重排 5 次（`ipi_comm.c:189`），一旦 `retry` 节点在它 pending 时触发（或时序稍有偏差），就等于**对活着的 timer 做 `list_del` 后重写链表节点** → `timer wheel` 损坏 / `list_del corruption` / 随机 oops |
| 对应现象 | "重试次数一多就崩溃"——命中的不是 IPI 数量，而是这个 |
| 修法 | 全部 5 个 delayed work 只在 `chsensor_debugfs_init()`（`ipi_comm.c:430-434`）初始化一次；`ipi_retry_transfer()` 里的重复 `INIT_DELAYED_WORK` 删除。`chsensor_poll_dw` 只排一次、`chsensor_getdata_dw` 用 `seq3_sent` 保证整个 session 只跑一次（老代码每次 pass 都跑 5 包 × 4 次 = 20 个无用 IPI） |

### 2.4 `新` `mpool_send_pkt()` 用 `static` 包缓冲 → 并发撕裂

| 项 | 内容 |
| --- | --- |
| 位置 | `ipi_comm.c:113`（改为栈上） |
| 老代码 | `static uint8_t pkt[128 + 8];` |
| 根因 | 该函数有三个并发来源：`chsensor_enable_work` / `chsensor_getdata_work`（都是 system_wq，可跑在不同 CPU），以及 `ipi_comm_sync() → __ipi_xfer() → ipi_prefetch_messages()`——**最后这条在调用者自己的上下文里同步执行**。`memcpy` 完成后到 `mtk_ipi_send()` 真正读走之间，另一个调用者可以把它改掉 → 固件收到半新半旧的包 → CHRE 解析器把它当成坏帧（`reB.md` §`chre_fw_ipi` 的无界校验循环） |
| 修法 | 改成栈上 `uint8_t pkt[136]`。另外 `mpool_blocked` 改 `atomic_xchg`（`ipi_comm.c:158`），避免 "一次性 bring-up" 守卫被两个并发调用者同时穿透 |

### 2.5 `新` 采样一稳定就不该再发 IPI（原逻辑永不停止）

| 项 | 内容 |
| --- | --- |
| 老逻辑 | `schedule_delayed_work(..., 4000)` 无条件重排 → 每 4 s 检查一次，且**任何一次 `wp` 回 0 都会重发整套序列**（`wp && wp != last` 这个条件在 `wp == 0` 时是"继续发"） |
| 修法 | 新增 `chsensor_fifo_flowing`，由 FIFO 读线程在成功推送记录时置位；`enable_work` 看到它就永久返回，不再排 timer。`attempt` 上限 6 保留（对应"3~6 次才开始"），超限打印一次后停 |

### 2.6 `new` `fifo` debugfs 节点在 SCP 复位期间读 DRAM → 同一个 oops

| 项 | 内容 |
| --- | --- |
| 位置 | `scp-chopin.c:657`（新） |
| 修法 | 加 `is_scp_ready(SCP_A_ID)` 早退；同时打印 `size % 44` 告警，以后一眼能看出头有没有被 `share_mem_init` 覆盖。为此加 `#include "scp/include/scp.h"` |

### 2.7 状态变量竞态

`chsensor_reset_req` 由 debugfs 写 handler 写、由 work 读改写，原来是普通 `int`；改 `atomic_t` + `atomic_xchg`（`ipi_comm.c:398, 481`）。`chsensor_fifo_last_valid` 让"第一次读"从固件当前 `wp` 开始，不再回放上次开机残留在 DRAM 里的旧记录。

---

## 3. 关于 `IPI 11 timeout` —— 需要修正交接文档里的一个假设

交接文档 §6.2 写：

> `IPI 11 timeout` 每次都在，说明 SCP 对我们的控制包根本不回 ack，"命中采样"是概率事件

前半句对，**后半句的因果链需要修正**。看 `mtk_ipi_send()` 的实现：

```c
/* drivers/soc/mediatek/mtk_tinysys_ipi.c:319 */
int mtk_ipi_send(ipidev, ipi_id, opt, data, len, timeout)
{
    ...
    timeover = cpu_clock(0) + MS_TO_NS(timeout);   /* timeout = 0 → timeover = now */
    ret = rpmsg_trysend(...);                      /* 只先试一次 */
    while (ret && ts_before(timeover)) { ... }     /* timeout=0 → 这个循环一次都不进 */
    ...
    if (ret == MBOX_PIN_BUSY) {
        ipi_timeout_dump(ipidev, ipi_id);          /* ← "Error: sensor IPI 11 timeout" */
        return IPI_PIN_BUSY;                       /* -6 */
    }
```

而驱动调用时传的是 `timeout = 0`（`ipi_comm.c:133`）。所以：

1. **每次 `mtk_ipi_send` 只做一次 `rpmsg_trysend`**，不等待。真正的重试是 `mpool_send_pkt()` 自己那个 `do/while` + `usleep_range`（`ipi_comm.c:132-143`）。
2. `MBOX_PIN_BUSY` 的含义是 **"上一次发到 pin 11 的消息还没被 SCP 取走"**（`mtk-mbox.c:458-475` 的 `pin_recv->lock` / 接收 IRQ 状态位）。
3. 每次命中 busy 都会打印那两行 dump。所以 **`IPI 11 timeout` 的条数 ≈ busy 次数**，它衡量的是"SCP 消费 mbox 有多慢"，不是"内核崩溃风险"。15 条/轮就是这轮里重试了 15 次。

**推论（重要）**：

* 增加重试次数**不会**让 SCP 收到更多包——`mtk_ipi_send` 在 `pin->mutex_send` 上串行化，同一时刻只有一个包在飞，重试的是**同一个包**。所以"重试 → IPI 风暴"其实是"重试 → 日志更多 + 每个包阻塞更久"。
* 因此"重试 5 次以上就崩"的主因**不是** IPI 数量，而是 §2.2（复位期间读 DRAM）和 §2.3（timer 链表损坏）。这两条修掉之后，重试次数应该可以安全地放大——如果还需要放大的话。
* 真正的"SCP 不回 ack"是**另一件事**：sensor_comm 那条 `wait_for_completion_timeout(1s)` 路径（`ipi_comm.c:787-800`）永远超时，因为固件根本没有 MTK `sensor_comm` 分发。这条已经是死代码，不影响 FIFO 通路。

---

## 4. 还没动、建议下一轮确认的两点

### 4.1 时间戳域

FIFO 路径的 `s.timestamp` 直接取固件记录里的 u64（`ipi_comm.c:360`），**绕过了** `sensorhub_deliver()` 里的 `timesync_filter_get(&dev->filter)`（`sensorhub_main.c:174`）。如果固件的时间戳不是 AP `CLOCK_BOOTTIME` 域，`iio_push_to_buffers_with_timestamp()` 出来的时间会跳。

* 我们 `SET_TIMESTAMP` 传的是 `ktime_get_boottime_ns()`，理论上固件会以此为基准——但要实测。
* 快速验证：`cat /sys/bus/iio/devices/iio:device1/in_accel_z_raw` 连读，同时 `dmesg -w` 看有没有 `timesync` 告警；或直接在 `chsensor_fifo_work` 里临时用 `ktime_get_boottime_ns()` 覆盖 `s.timestamp` 对比 iio-sensor-proxy 的 `monitor-sensor` 输出是否更稳。

### 4.2 `sensorhub_iio_poll()` 用错了时间戳源

`sensorhub_iio_poll()` 里 `iio_push_to_buffers_with_timestamp(..., iio_get_time_ns(indio_dev))`，但 `iio_pollfunc_store_time` 已经把时间戳放在 pollfunc 的 buffer 里了，应该是 `((s64 *)p->buffer)[0]`。目前这条路径只在"有 trigger 但 SCP 不流"时走到，影响小，但写法是错的。

---

## 5. 验证步骤

```sh
# 0. 先确认头对不对（这是 §1 的判据）
echo 1 > /sys/kernel/debug/scp-chopin/fifo
#   期望: fifo rp=0 wp=... size=1048520            ← 44 的倍数
#   如果: size=1048536 ... [size is not a multiple of 44!]   → §1 的覆盖发生了

# 1. bring-up 只剩 6 次，且成功后不再刷 IPI
dmesg | grep -c "chsensor: send ret"        # 应 ≤ 6
dmesg | grep "chsensor: bring-up gave up"   # 只应在真失败时出现
dmesg | grep -c "IPI 11 timeout"            # 前 30 s 之后不该再增长

# 2. 数据稳定性：连续 20 次读，z 应该在 9700~10300 之间小幅抖动，
#    不应该出现"方向整体翻转"或"某一轴固定在 0"
D=/sys/bus/iio/devices/iio:device1
i=0; while [ $i -lt 20 ]; do
  echo "$(cat $D/in_accel_x_raw) $(cat $D/in_accel_y_raw) $(cat $D/in_accel_z_raw)"
  i=$((i+1)); sleep 1
done

# 3. 长跑（> 5 分钟）——这是 §1 的关键判据：错位只在第一次回绕后出现
#    约 4 min 后如果 z 开始乱、或 x/y/z 变成"互相串位"，说明 §1 没修干净

# 4. 崩溃回归
dmesg | grep -c "Oops\|Unable to handle\|list_del corruption\|refcount_t"
#    注意交接文档 §6.4 的 drm_panel_add / lcm_probe refcount UAF 与传感器无关，
#    若仍出现请先禁用 mi-dsi-panel 验证

# 5. 反复压 retry（老版本崩在这里）
i=0; while [ $i -lt 20 ]; do
  echo 1 > /sys/kernel/debug/chsensor/retry; i=$((i+1)); sleep 2
done
```

---

## 6. 本次改动清单

```
drivers/misc/mediatek/sensor/2.0/sensorhub/ipi_comm.c   +258 -86
drivers/misc/mediatek/scp-chopin.c                      +19  -8
```

编译已验证（`make ... drivers/misc/mediatek/scp-chopin.o` 与 `.../sensorhub/` 均通过，无新增告警）。

主要变更点：

| 行 | 内容 |
| --- | --- |
| `ipi_comm.c:113` | `mpool_send_pkt` 包缓冲移到栈上 |
| `ipi_comm.c:158` | `mpool_bringup_done` 原子化 |
| `ipi_comm.c:249` | 新增 `SENS_FIFO_BYTES`（44 的倍数，单一来源） |
| `ipi_comm.c:272` | 新增 `chsensor_fifo_write_header()` |
| `ipi_comm.c:321-338` | FIFO 头几何校验 + 限流自动重写 |
| `ipi_comm.c:331-347` | 环形前向距离 + 倒退 resync |
| `ipi_comm.c:382` | 回写 `rp`（固件不读，但让环形自描述） |
| `ipi_comm.c:398-434` | delayed work 一次性初始化，删除重复 `INIT_*` |
| `ipi_comm.c:481-600` | bring-up 重写：SCP 就绪门控、SET_CONFIG 只发一次、上限 6 次、出数即停 |
| `scp-chopin.c:657` | `fifo` 节点加 `is_scp_ready` 保护 + 44 对齐告警 |

---

## 7. 实机日志复核（2026-10-27 21:39，v89 镜像）

### 7.0 先确认：这份日志是**修复前**的内核

判据：日志里是 `chsensor: ENABLE accel 100Hz (seq 3, retry)`，
修复后的代码打印 `(seq 3, retry %u)`（带次数）。所以这次崩溃正是 §2.2/§2.3
那两个已修的问题，不是新问题。

### 7.1 崩溃点被日志直接坐实：delayed work 被反复 re-init

`GET_DATA` 的实际间隔：

| # | 时刻 | 间隔 |
| --- | --- | --- |
| 0 | 18.414 | — |
| 1 | 18.926 | 0.51 s |
| 2 | 19.438 | 0.51 s |
| 3 | 19.950 | 0.51 s |
| 4 | 20.462 | 0.51 s |
| 5 | 22.510 | **2.05 s** |
| 6 | 26.606 | **4.10 s** |
| — | 之后无输出，系统死亡 | |

`chsensor_getdata_work` 自己每 500 ms 重排 5 次，而 `chsensor_enable_work`
在 `attempt >= 2` 时又对它调用 `INIT_DELAYED_WORK` 并重新 `schedule`。
间隔从 0.5 s 漂到 2.0 s、4.1 s，正是"对 pending 的 delayed work 重写
`timer.entry`"的表现——timer wheel 链表被破坏，几秒后整个定时器子系统挂掉。
这与 `attempt 6` 之后就再没有任何输出完全吻合。

修法见 §2.3（全部 work 只在 `chsensor_debugfs_init()` 初始化一次）。

### 7.2 `wp` 整 boot 为 0 的新根因：`SET_CONFIG` 发早了 4 秒

```
5.950  scp-chopin-start: started: c0 st=00000000 pc=0000034c   ← SCP 刚放出来
6.898  chsensor: SET_CONFIG fifo 0x8f0ca700 size 1048520       ← 只过了 0.95 s
```

固件在自身 init 完成（~1.0 s，"accGyro: init done"）之前收到包会丢弃并打印
`len 34 > 269, gRxBuf 0x0`。0.95 s 正落在这个窗口里，于是：

* 固件**没有**执行 SET_CONFIG → `0xa9e88`（chre_dram）保持旧值/0；
* 我们从没注册 FIFO，所以 `0x8F0CA700` 的 `wp` 永远不会被写；
* 后续所有 CHRE enable 都是在给一个没有 FIFO 的 hub 下命令。

旧代码只在 `ipi_retry_transfer()` 里发配置、把 *enable* 延后 5 s——
延后的是错的那一个。**已修**：配置与 enable 现在一起发生在
`chsensor_enable_work()` 的第一次运行（`chsensor_delay_ms` = 5 s 之后），
`ipi_retry_transfer()` 里那套提前发送已删除。

### 7.3 §1 的 header 覆盖这次**没有**发生

```
15.131  scp-chopin: fifo rp=0 wp=0 size=1048520 records=23830
1.937   @@@@ reserved: <1  ca700  0>          ← SENS_MEM size = 0
6.899   sensorhub share mem config ack -19, continuing
```

`size = 1048520` 是 44 的倍数，说明 `share_mem_init()` 这次没写进去
（reserve mem 查不到，`share_mem_config()` 提前失败）。所以 §1 是
**boot 相关**的，不是每次必现——与它的"间歇性"表现一致。
结论不变：自愈逻辑保留（不会误触发），但根本修法仍是 §1 的方案 A/B。

### 7.4 `SET_CONFIG send ret 0` 是假成功

```
6.898418  chsensor: SET_CONFIG fifo 0x8f0ca700 size 1048520
6.898444  chsensor: SET_CONFIG send ret 0
6.898460  Error: scp_ipidev IPI 11 timeout ...   ← 之后 15 条 busy
```

`ret 0` 只代表 `mtk_ipi_send()` 最终拿到了 mbox pin，**不代表 SCP 处理了这个包**。
真正能证明"固件开始采样"的仍然只有 `wp` 变化。

### 7.5 顺手修掉的一个 taint

```
workqueue: sensor_ready is using neither WQ_PERCPU or WQ_UNBOUND
WARNING: kernel/workqueue.c:5938
```

`ready.c:213` 的 `alloc_workqueue("sensor_ready", WQ_MEM_RECLAIM | WQ_HIGHPRI, 0)`
在新内核上必须显式声明 per-CPU 或 unbound。已加 `WQ_UNBOUND`。

### 7.6 与本问题无关、但日志里出现两次的一处

```
refcount_t: addition on 0; use-after-free.
  drm_panel_get -> drm_panel_add -> lcm_probe
refcount_t: saturated; leaking memory.
  drm_panel_get -> of_drm_find_panel -> mtk_dsi_probe
```

即交接文档 §6.4 的 `mi-dsi-panel` / `lcm_probe` 引用计数 UAF。它会 taint 内核，
但这次不是它导致死机（崩在 26.6 s，面板那条在 13.3 s 且只是 WARNING）。

### 7.7 另外两个可以忽略的观测

* `ipi_comm register ipi 40 failed -3`（`IPI_UNAVAILABLE`）：这块板子的 pin
  table 里根本没有 40（mbox3 只到 15/20-23）。原注释说"必须注册否则 BUG_ON"
  已经过时，非致命，不用管。
* 20 次 `poll` 全部 `stat=00000000 pol=00081000`：EINT12 的锁存位一直是 0。
  在固件根本没被 enable 的前提下这是**预期**的，先不要拿它当硬件问题证据。

### 7.8 修复后的预期差异（下次 boot 对照）

| 项目 | 修复前 | 修复后应为 |
| --- | --- | --- |
| `SET_CONFIG` 时刻 | SCP 启动后 ~0.95 s | SCP 启动后 ~≥5 s |
| `SET_CONFIG` 与 `DISM/ENABLE` 顺序 | 相隔 5 s | 连续（同一次 pass） |
| `GET_DATA` 次数 | 每个 pass 5 次，共 20 次，间隔漂移 | 全程只 1 组 5 次，间隔 500 ms |
| `send ret` 行数 | ≤6 pass × 5 包 | 第 1 次 pass 后按 `attempt` 计 |
| 出数后 | 仍每 4 s 继续发 | 停止，不再有任何 IPI |

---

## 8. v90 镜像（2026-10-27 21:47）

```
out-scp-v90-alpine/boot-stockdt.img   md5 faeb1de603284c1ac0b0dce7719ca155
out-scp-v90-alpine/boot-owndtb.img    md5 b82e12cfaa290bcbc9b590dd28c63954   (备用)
内核版本 7.3.0-g59012e451bb9-dirty
```

已核对 `Image.gz` 里确实带新代码：

```
fifo header bad (size %u wp %u), rewriting to %u
ENABLE accel 100Hz (seq 3, retry %u)          ← 上一版这里是 "retry)"，可据此区分
SET_CONFIG fifo 0x%x size %u ret %d           ← 上一版是两条分开的打印
```

`--cmdline` 不用传：`CONFIG_CMDLINE_FORCE=y`，内核强制用 `.config` 里的
`CONFIG_CMDLINE`（含 `chopinport=v2onehundredfifth`），
boot 镜像的 cmdline 根本不会生效。

刷写：

```sh
fastboot flash boot_a out-scp-v90-alpine/boot-stockdt.img && fastboot reboot
```

---

## 9. v90 实测：崩溃已消失，但 SCP 根本没被启动（2026-10-27 21:50）

### 9.1 好消息

```
7.3.0-g59012e451bb9-dirty            ← 新内核
dmesg | grep -c "GET_DATA"  ->  0    ← 旧版那 20 条刷屏 + 间隔漂移消失
dmesg | grep -E "Oops|list_del|refcount_t: addition on 0"  -> 只有 lcm_probe 那条旧的
50s 仍活着，没有卡死
```

§2.3 的 timer 破坏确认修好：`chsensor_getdata_dw` 只在 `seq3_sent` 时排一次，
不再被 `INIT_DELAYED_WORK` 反复重置。

### 9.2 新问题：这一版镜像没有启动 SCP

```
[1.871931] [SCP] auto start disabled (mediatek,no-auto-start); light start: echo "666 0 1" > ...
[50.176335] scp-chopin: SCP not ready, fifo dump skipped
```

整份日志里**没有** `scp-chopin: patchscp`，也**没有** `scp-start: started`。
所以 `SET_CONFIG` / `DISABLE` / `ENABLE` / `GET_DATA` 一条都没有——
不是驱动不发，是 `chsensor_enable_work` 的 `is_scp_ready()` 门控把它们挡住了
（这是正确行为）。

根因：**SCP 的启动不在内核里，在 userspace**。
`mediatek,no-auto-start` 让厂商驱动不自动放核心，只有
`port/rootfs-alpine/etc/local.d/10-chsensor.start` 里那套
`patchscp → fullreset → sram → start` 会放。

而这个脚本是 **OpenRC 的 local 服务**，本 initramfs 用
`port/initramfs/init.c` 当 PID 1，**没有 OpenRC**，所以 `local.d/*.start`
从来没被执行过。（前一次能出 `patchscp`，是因为那次跑的是 `dist/` 那套
带 OpenRC 的完整 Alpine ramdisk，不是 `out-scp-*` 这套。）

### 9.3 已修：init.c 现在会跑 local.d

`init.c` 增加 `run_local_d()`，在 step 7 之后、交给 console 之前
fork 一个子进程执行 `for f in /etc/local.d/*.start; do /bin/sh "$f"; done`。
脚本内部要睡 1+4+18+12 秒，所以必须放子进程，不能阻塞 init。

### 9.4 v91 镜像

```
out-scp-v91-alpine/boot-stockdt.img   md5 0ea1cf699d4acbcf2f3ceb14155347e8
内核仍是 7.3.0-g59012e451bb9-dirty（内核没改，只改了 initramfs）
```

已核对 cpio 里有 `/init`、`/etc/local.d/10-chsensor.start`，且 `/init`
带 `init: running /etc/local.d/*.start` 与那条 shell 循环字符串。

### 9.5 不重刷也能验证（在当前 v90 里手动放 SCP）

```sh
mount -t debugfs none /sys/kernel/debug 2>/dev/null
echo 2 > /sys/kernel/debug/scp-chopin/patchscp
echo 1 > /sys/kernel/debug/scp-chopin/fullreset
echo 1 > /sys/kernel/debug/scp-chopin/sram
echo 1 > /sys/kernel/debug/scp-chopin/start
sleep 6
dmesg | grep -E "scp-start: started|SET_CONFIG|DISABLE accel|ENABLE accel|attempt"
echo 1 > /sys/kernel/debug/scp-chopin/fifo     # 期望 size=1048520
```

关键对照点：

| 观测 | 含义 |
| --- | --- |
| `SET_CONFIG` 出现在 `started` 之后 ≥5 s | §7.2 的"发太早"已修 |
| `SET_CONFIG fifo ... ret %d` 是**一条**日志 | 确认跑的是新代码 |
| `attempt` 停在 6 以内且之后不再有 IPI | 重试有界 |
| `wp` 仍为 0 | 排除发太早之后，就是 reB.md 的不 ack 问题 → 上方案 A |

---

## 10. 手动放 SCP 后的实测（2026-10-27 12:21）——找到 `wp` 恒 0 的直接原因

### 10.1 四个修复全部确认生效

```
16.366  SET_CONFIG fifo 0x8f0ca700 size 1048520 ret 0     ← 单行日志 = 新代码
16.367  DISABLE accel (seq 1)      ← 16.366 - 10.348(SCP started) = 6.0 s ≥ 5 s
16.686  ENABLE accel (seq 2, re-arm)
21.742  ENABLE accel (seq 3, retry 2)
...
34.030  ENABLE accel (seq 3, retry 5) → send ret 0 (attempt 6) → 之后不再发送
```

| 检查项 | 结果 |
| --- | --- |
| 崩溃 | ✅ 整份日志无 oops / list_del / timer 报错，34 s 后系统仍活着 |
| `IPI 11 timeout` | ✅ **一条都没有**（旧版每个包 15 条）→ SCP 的 mbox 是健康的，包被收下了 |
| SET_CONFIG 时机 | ✅ SCP 启动后 6.0 s |
| 重试上限 | ✅ 停在 attempt 6 |
| `wp` | ❌ 仍是 0 |

`IPI 11 timeout` 消失这一点很关键：它证明**包确实被 SCP 取走了**，
所以问题不在投递、不在时序、不在 IPI，而在固件收到之后没干活。

### 10.2 直接原因：SCP 被重启了，而 SET_CONFIG 每 session 只发一次

```
16.366  SET_CONFIG fifo 0x8f0ca700            ← 第 1 个 SCP 实例
30.882  scp-chopin-start scp-start: started   ← SCP 被重新放起来（第 2 个实例）
34.674  [SCP] cfg[0x014c] ... cfg_core0[0x0114] ...   ← print_clk_registers()
36.933  scp-chopin: SCP not ready, fifo dump skipped
```

`print_clk_registers()` 在整棵树里**只有一个调用者**：
`scp_sys_reset_ws()`（`scp_helper.c:1942`）。所以 34.67 那一大段寄存器 dump
= **厂商驱动对 SCP 做了一次复位**，随后 36.9 已经 `not ready`。

而驱动的 `chsensor_config_done` 在一次 session 里只置位一次：
`30.882` 换了新固件实例后，`SET_CONFIG` 再也没有发过，
`chsensor_attempt` 也已经烧到 6 → 直接放弃。

⇒ **第 2 个（以及第 3 个）SCP 实例从来没有收到过 FIFO 注册，`wp` 当然是 0。**
这也解释了交接文档里"需要 3~6 次才出数"的随机性：出数的那几次，
恰好是 SCP 没有被重置。

### 10.3 已修 v92

1. **监听 SCP 生命周期并重新 arm**：`scp_A_register_notify()`，
   `SCP_EVENT_STOP` / `SCP_EVENT_READY` 都调 `chsensor_reset_state()` 并重排
   `chsensor_enable_dw`。于是每次重启都会重新走 SET_CONFIG + enable，
   而且 `attempt` 预算按"每个 SCP 实例"重新计算。
2. **`chsensor_zero_ipi` 调试开关**（`bool`，运行时可用）：
   打开后 `mpool_send_pkt()` 直接返回 0，完全不跟固件说话，
   用来回答"打过补丁的固件能不能自己采样"这个方案 A 的问题。
   路径：`/sys/module/sensorhub/parameters/chsensor_zero_ipi`
   （模块名是 `sensorhub`，不是 `ipi_comm`）。

### 10.4 顺带更正一处几何（我差点判错）

`@@@@ reserved: <id  size  alignment>`（`scp_helper.c:1556`），
不是 `<id offset size>`。按 id 顺序累加后：

| id | 块 | size | start |
| --- | --- | --- | --- |
| 1 | VOW_MEM | 0xca700 | 0x8F000000 |
| **2** | **SENS_MEM** | **0x100000** | **0x8F0CA700** |
| 3 | SCP logger | 0x180000 | 0x8F1CA700 |

所以 SENS_MEM 确实是 1 MB、起点正好 0x8F0CA700，`0x10 + 0xFFFC8 = 0xFFFD8 < 0x100000`，
**FIFO 没有越界**，§1 的 44 对齐自愈也不需要改尺寸。
固件日志环在 **0x8F1CA700**（0x180000）。

### 10.5 v92 镜像

```
out-scp-v92-alpine/boot-stockdt.img   md5 a30bab0d6fe08fff90052b28ccc4da20
内核 7.3.0-g59012e451bb9-dirty
```

### 10.6 下一个判据（不用等镜像也能做）

读固件自己的日志环，看它到底有没有跑 `ICM4X6XXSample`：

```sh
python3 - <<'PY'
import subprocess
base = 0x8F1CA700
buf = b""
for off in range(base, base + 0x4000, 4):
    v = int(subprocess.check_output(["devmem", "0x%X" % off]).strip(), 16)
    buf += v.to_bytes(4, "little")
t = buf.decode("latin1")
for key in ("ICM4X6XX", "accGyro", "jitter", "eint", "dram", "maxEventNumber"):
    i = t.find(key)
    if i >= 0:
        print("---", key, "---")
        print(t[max(0, i - 200):i + 600])
PY
```

三种结果对应三条路：

| 固件日志里看到 | 结论 | 下一步 |
| --- | --- | --- |
| 完全没有 `ICM4X6XXSample` | 芯片侧没出数（INT1/FIFO 配置或 EINT12） | 方案 A + 查 `INT1_CTRL(0x2D)` / `FIFO_CTRL3` |
| 有 `Sample` 但 `maxEventNumber` 为 0/异常 | SET_CONFIG 没生效 | 检查重启时机（v92 已修） |
| 有 `Sample` 且 `wp` 仍 0 | drain 没跑到 `0x3acb2` | 回到 reK.md 的 drain 路径 |

---

## 11. 固件日志（2026-10-27 12:2x）——固件在 boot 阶段就自己关掉了加速计

### 11.1 关键证据

```
[0.087](0) acc spi_num: 2,eint 12
[0.087](0) acc map[0]:1, map[1]:0, map[2]:2, sign[0]:1, sign[1]:-1, sign[2]:1
[0.088](0) ICM40607 detected                      ← 芯片识别、SPI、WHOAMI 全 OK
[0.641](0) ICM4X6XX watermarkReg 16
[0.641](0) ICM4X6XXConfigFifo: Reset, TH_L:0x10, TH_H:0x0
[0.643](0) acc: ratechg done
[0.895](0) jitter check failed even 4 retries:0   ← data-ready 等不到
[0.899](0) sensorPowerAcc on:0, nowOn:1
[0.901](0) accGyro, delay=1797783552, latency:0, min_wm=0, total_cnt=0
[0.901](0) accGyro, watermark:0 < 1               ← ★ 固件主动放弃了
[0.902](0) ICM4X6XXAccPowerOff
[0.902](0) ICM4X6XX set power mode=2
[0.905](0) acc: disable done
```

固件字符串里这两条的原文（`work/scp_code.bin`）：

```
accGyro, delay=%lu, latency:%lld, min_wm=%d, total_cnt=%lu
accGyro, watermark:%d < 1
accGyro, delay=%lu, latency:%lld, min_wm=%d, final_watermark=%d   ← 还有第二段，我们还没看到
```

所以链条是：**固件算出 `watermark = 0`，因为有个 `watermark < 1` 的检查，于是
`ICM4X6XXAccPowerOff` + `set power mode=2` + `acc: disable done`，自己把加速计关了。**

那个 `delay=1797783552`（≈1.8 s）是个不合理值——它是固件 boot 阶段的默认，
不是我们给的 10 ms。也就是说**还没有任何一个合法速率配置落到它手里**。
`final_watermark` 那一段日志我们还没读到，正常路径应该在那里打印。

### 11.2 但这只是环的"开头"

`devmem` 从 `0x8F1CA700` 顺序读 16 KB，读到的是 `t=0.08 … 5.156`，
而我们的 `SET_CONFIG` 落在固件时间 ≈6 s 之后（host 16.366 − SCP 启动 10.348）。
所以：

* 没搜到 `dram` / `maxEventNumber` → **不能**据此断言 SET_CONFIG 没生效，
  只是那段日志不在我们读的这 16 KB 里；
* 环是 0x180000（1.5 MB）且会回绕，`devmem` 逐字读全量太慢。

### 11.3 已修：内核加了 `scp-chopin/log` 节点（v93）

`scp-chopin.c` 新增 debugfs `log`：
扫完整 1.5 MB 日志环，只保留带固件前缀 `[秒.毫秒](核号) ` 的行，
按时间顺序输出**最后 400 行**。一次 `cat` 就能看到最新状态，不受回绕影响。

```sh
cat /sys/kernel/debug/scp-chopin/log | grep -E "dram|maxEventNumber|watermark|Sample|ratechg"
```

若 SCP 处于复位状态会拒绝读（和 `fifo` 节点同样的保护），提示先放核心——
日志环在 AP DRAM 里，重启 SCP 不会丢。

### 11.4 两种结果的含义

| `log` 里看到 | 结论 |
| --- | --- |
| 有 `get dram phy addr=0x9F0CA700,size=1048520, maxEventNumber:23830` | SET_CONFIG 生效了，问题在 enable 的速率没落地（回到 CHRE 解析） |
| 没有这行 | SET_CONFIG 根本没被处理，§10 的重发逻辑改了也白改，得上 §B 攻 ack |
| 有 `final_watermark=` 且 ≥1 | 速率配置成功了，那时再看 `wp` 为什么不动 |
| 只有 `watermark:0 < 1` 反复出现 | 固件始终没拿到合法速率 |

### 11.5 v93 镜像

```
out-scp-v93-alpine/boot-stockdt.img   md5 d7fa2dc6ec9caef8bc92843203ced90b
内核 7.3.0-g59012e451bb9-dirty
```

---

## 12. 日志环的读取方式修正（2026-10-27 12:36）

### 12.1 v93 的 `log` 节点取错了区段

v93 的实现是"扫全块、取最后 400 行"。但固件是**从偏移 0 往后追加**的
（它自己打印 `logger_update: ... dst_len= 1048575 → 1048406 → 1043665`，
`dst_len` 是剩余空间，从 0xFFFFF 递减），要填满 ~1 MB 才回绕。
所以"块尾"是**上一轮 boot 残留的 DRAM**，不是最新日志——
这就是读出来停在 `[0.902]` 而 `devmem` 从偏移 0 读却能看到 `[5.156]` 的原因。

### 12.2 正确判据：从偏移 0 开始的连续上升段

固件日志行格式固定为 `[秒.毫秒](核号) `，且时间单调上升、相邻间隔很小。
所以当前 boot 的日志 = **从偏移 0 开始、时间单调且无大跳变的最长前缀**，
第一个断点之后全是旧数据。

v94 起 `scp-chopin/log` 按这个规则选段，输出首行自带说明：

```
# 1234 lines in ring, this boot: 1180 (t=0.083 .. 15.402 s)
```

环真的写满过（回绕）时才回退到"时间戳最新的那一段"。
单元测试（合成样本 = 1715 行 live + 300 行残留）验证过选段正确。

### 12.3 顺手解决 paste 被吃掉的问题

多行 heredoc 在 minicom 里会被粘坏（实测 `mmap` 那段整段错位）。
两处改进：

1. **`/dev/mem` 的 `mmap` 必须页对齐**，`0x8F1CA700 % 4096 = 0x700`，
   所以原来的 `mmap(..., offset=BASE)` 必然 `EINVAL`。
   改用 `os.pread()` 一次读完，绕开对齐问题：
   ```sh
   python3 -c 'import os;print(os.pread(os.open("/dev/mem",0),0x20000,0x8F1CA700).decode("latin1","replace"))' | grep -E "dram|maxEventNumber|final_watermark|Sample|ratechg"
   ```
2. **把工具放进 rootfs**：`/usr/bin/chopin-scplog`（python3，无外部依赖），
   以后不用再粘代码：
   ```sh
   chopin-scplog -g 'dram|maxEventNumber|final_watermark|Sample|ratechg'
   chopin-scplog -n 200          # 最后 200 行
   chopin-scplog -a              # 列出块里所有 run（每次 boot 一段）
   ```

### 12.4 v95 镜像

```
out-scp-v95-alpine/boot-stockdt.img   md5 5e5601bd4628ff81af91f122228c9129
```
内核 = v94（日志节点已按 §12.2 修正），rootfs 增加 `chopin-scplog`。

---

## 13. `/dev/mem` 读取方式：必须 mmap，不能 pread（2026-10-27 12:45）

### 13.1 现象

```sh
python3 -c 'import os;os.pread(os.open("/dev/mem",0),0x20000,0x8F1CA700)'
# OSError: [Errno 14] Bad address
```

而 `devmem 0x8F1CA700` 一直能用。原因是两者走的路径不同：

* `read()`/`pread()` 走 `CONFIG_STRICT_DEVMEM` 的检查，
  而这块区域是 **System RAM**（`reserve-memory-scp_share` 是 `nomap` 保留区，
  但内核仍把它算作 RAM），于是被拒 → `EFAULT`；
* busybox `devmem` 用的是 **`mmap()`**，不受这条限制。

`mmap` 的 `offset` 必须页对齐，`0x8F1CA700 % 4096 = 0x700`，
所以最初那版 `mmap(..., offset=BASE)` 报的是 `EINVAL`。
正确写法是映射所在页再切片：

```python
aligned = BASE & ~(mmap.PAGESIZE - 1)     # 0x8F1CA000
delta   = BASE - aligned                  # 0x700
m = mmap.mmap(fd, SIZE + delta, mmap.MAP_SHARED, mmap.PROT_READ,
              offset=aligned)
raw = m[delta:delta + SIZE]
```

### 13.2 现在就能用的一行（当前 v93 也行）

```sh
python3 -c 'import mmap,os;b=0x8F1CA700;p=b&~4095;f=os.open("/dev/mem",os.O_RDONLY|os.O_SYNC);m=mmap.mmap(f,0x180700,mmap.MAP_SHARED,mmap.PROT_READ,offset=p);print(m[b-p:b-p+0x20000].decode("latin1","replace"))' | grep -E "dram|maxEventNumber|final_watermark|Sample|ratechg"
```

### 13.3 v95 的脚本是坏的，v96 修好

v95 打进 rootfs 的 `/usr/bin/chopin-scplog` 还是 `pread` 版本（必然 `EFAULT`）。
v96 已改成 mmap，并补了 `SIGPIPE` 处理（否则 `| head` 会抛 `BrokenPipeError`）。

```
out-scp-v96-alpine/boot-stockdt.img   md5 dff6a6d03c020650e6dde07b32240f29
```

用合成日志环（1275 行 live `t=0..14.001` + 300 行残留 `t=0..0.897`）端到端验证过：

```
$ chopin-scplog                  → # 1275 of 1275 lines this boot (t=0.000 .. 14.001 s)
$ chopin-scplog -g 'ratechg'      → 只出 live 段里那 2 行
$ chopin-scplog -a                → run 0: 1275 lines / run 1: 300 lines（残留被正确分离）
$ chopin-scplog | head -3         → 无 traceback
```

---

## 14. v96 的固件日志：SET_CONFIG 从未被固件执行（2026-10-27 12:43）

### 14.1 数据

```
$ chopin-scplog -g 'dram|maxEventNumber|final_watermark|Sample|ratechg'
# 1 of 167 lines this boot (t=0.026 .. 5.331 s)
[0.643](0) acc: ratechg done

$ chopin-scplog -a
run  0:   167 lines  t=0.026 .. 5.331 s
```

当前 boot 只有一个段（没有被残留污染），167 行，到 t=5.331 s 为止。
在这段里：

* **没有** `get dram phy addr=` / `maxEventNumber` → **SET_CONFIG 从未执行**；
* **没有** `final_watermark=` → 速率命令也没被处理；
* **没有** `ICM4X6XXSample` → 自然没有样本，`wp` 必为 0；
* 只有 `acc: ratechg done`（boot 阶段固件自己那一次）。

`scplog` 与 `devmem` 两种读法结论一致，所以这不是读数问题。

### 14.2 结论：包到了，但被错误的消费者解了

reK.md 给出的判据很硬：`SET_CONFIG` 入口 `0x3b03c` 一定会打印
`get dram phy addr=%p,size=%lu, maxEventNumber:%lu`。既然固件日志里没有这一行，
就说明包**没有进入 SCP_SENSOR_HUB_DATA 的 SET_CONFIG 分支**。

而 reB.md 已经查到原因方向：`pkt_id 3`（`IPI_SENSOR`）在固件的分发表里落到
`chre_fw_ipi`（`0x3b3ee`）——一个 CHRE 报文解析器，而不是 `0x3b00e` 的
`SET_CONFIG`。把我们 48 字节的 SDATA 丢给 CHRE 解析器，
结果就是"静默丢弃"或"ISR 里空转"，两种情况都不会打印 `get dram phy addr`。

### 14.3 v97：把 pkt_id 做成运行时旋钮

`enum ipi_id` = `IPI_MPOOL 0` / `IPI_CHRE 1` / `IPI_CHREX 2` / `IPI_SENSOR 3` /
`IPI_SENSOR_INIT_START 4`，驱动之前把 3 写死。

新增：

```
/sys/module/sensorhub/parameters/chsensor_pkt_id    (uint, 默认 3)
```

SET_CONFIG / SET_CUST / SET_TIMESTAMP / GET_DATA 都改用它；日志也带上它：

```
chsensor: SET_CONFIG fifo 0x8f0ca700 size 1048520 ret 0 (pkt_id 3, mode 0)
```

配套脚本 `port/tools/accel-pktid-sweep.sh`：把 0..4 逐个设进去、
`retry` 重新 arm、然后查固件日志里有没有 `dram` 行，第一个命中就停。

```sh
sh /path/to/accel-pktid-sweep.sh
```

判据只有一个：**固件日志里出现 `get dram phy addr=` 就是命中**。

### 14.4 v97 镜像

```
out-scp-v97-alpine/boot-stockdt.img   md5 42bcc8d3a2b848058695f34a03f36158
```

### 14.5 如果 0..4 全都不中

那就不是 id 的问题，而是 mpool 帧格式（`[u32 id][u32 len]`）本身
不是固件 mpool 分派器认的格式。下一步该做的是：
用 `mpool_mode` 的 3 种帧格式 × 5 个 id 交叉扫（15 组），
仍然只看固件日志里那一行，而不是看 `ret`。

---

## 15. pkt_id 0..4 全不中，但暴露出更关键的规律（2026-10-27 12:52）

### 15.1 数据

```
for i in 0 1 2 3 4; do ... chopin-scplog -g "dram|maxEventNumber" ; done

# 0 of 167 lines this boot (t=0.026 .. 5.199 s)   ^^ pkt_id 0
# 0 of 167 lines this boot (t=0.026 .. 5.199 s)   ^^ pkt_id 1
# 0 of 167 lines this boot (t=0.026 .. 5.199 s)   ^^ pkt_id 2
# 0 of 167 lines this boot (t=0.026 .. 5.199 s)   ^^ pkt_id 3
# 0 of 167 lines this boot (t=0.026 .. 5.199 s)   ^^ pkt_id 4
```

两层信息：

1. **没有任何 id 命中** `SET_CONFIG`（5 次 × 7 s，固件日志一行没长）；
2. **日志在 35 秒里完全没有增长**，一直停在 167 行 / `t=5.199`。

### 15.2 三次独立取样，日志都停在 ≈5.2 s

| 取样方式 | 镜像 | 日志末端 |
| --- | --- | --- |
| `devmem` 读环头 | v90 | `5.156` |
| stale run（v93 的错取段） | v93 | `5.331` |
| `chopin-scplog`（正确取段） | v96/v97 | `5.199` |

而 `SET_CONFIG` 的发送时刻恒定为 **SCP 释放后 `chsensor_delay_ms` = 5 s**。
两者重合得过于整齐，这个相关性必须排除或确认。

### 15.3 决定性实验（v92+ 就有开关，不用重刷）

`chsensor_zero_ipi=1` 后重启 SCP，等 30 s，看日志是否越过 5.2 s：

* 越过 → 固件本来会持续写日志，是**我们的包把它卡死**（reB.md 的
  `chre_fw_ipi` 在 mbox ISR 里空转，日志死在那一刻）；
* 仍停在 5.2 s → 固件 boot 之后就不写日志，我们的包是被**静默丢弃**。

一行版（避免粘贴被吃掉）：

```sh
echo 1 > /sys/module/sensorhub/parameters/chsensor_zero_ipi; echo 2 > /sys/kernel/debug/scp-chopin/patchscp; echo 1 > /sys/kernel/debug/scp-chopin/fullreset; echo 1 > /sys/kernel/debug/scp-chopin/sram; echo 1 > /sys/kernel/debug/scp-chopin/start; sleep 30; chopin-scplog -a; chopin-scplog -n 6
```

### 15.4 v98：把实验做成脚本

```
out-scp-v98-alpine/boot-stockdt.img   md5 f6b940faeed30d7618e1d804d8c29385
```

`/usr/bin/chopin-accel-test`：

```sh
chopin-accel-test        # 零 IPI 重启 SCP，等 30 s，打印结论
chopin-accel-test -i     # 对照组：保留控制包
```

它会自动设 `chsensor_zero_ipi`、`chsensor_pkt_id 0`、重新 patchscp +
fullreset + sram + start，等 30 s，然后打印日志 run 列表、最后 6 行、
以及最近的 `SET_CONFIG` / `IPI 11 timeout` 行。

### 15.5 两种结果各自的下一条路

| 实验结果 | 结论 | 下一步 |
| --- | --- | --- |
| 零 IPI 时日志越过 5.2 s | 我们的包打死了 SCP | 停止一切控制包，只走 patchscp + 轮询；能出数就删掉整套 IPI 控制（方案 A 变体） |
| 零 IPI 时日志也停在 5.2 s | 包被静默丢弃，SCP 没被影响 | 帧格式问题：`mpool_mode`(3) × `chsensor_pkt_id`(5) 交叉扫，仍只看固件日志 |

---

## 16. 突破：固件自己说出了丢弃原因（2026-10-27 12:58）

零 IPI 对照没有跑出结论（见 §16.3），但 `chopin-scplog -n 6` 把上一轮实例的
日志尾打了出来，拿到两条决定性证据。

### 16.1 MTK legacy handler 表里没有我们的 id

```
[5.198](0) legacy_handler id 11,pkt_id:0 handler is null!!! prdata:0x0 data:0xbe1cc len:136
```

拆开看：

| 字段 | 含义 |
| --- | --- |
| `id 11` | mpool pin = `IPI_OUT_SCP_MPOOL_0` |
| `pkt_id:0` | 我们从 `[u32 pkt_id][u32 len]` 里填的那个数，**被正确读出来了** |
| `handler is null` | 固件的 legacy mpool 处理器表里这一项是空的 → 丢弃 |
| `data:0xbe1cc` | 共享槽地址，正是 reB.md §`chre_fw_ipi` 那个 |

**同时证明三件事**：包确实到了固件；`[u32 id][u32 len]` 帧格式正确；
**MTK 的 legacy 分派表是空的**。

⇒ `SET_CONFIG`（action 4）的服务端在 `0x3b00e`，它挂在这张空表后面，
所以**换任何 pkt_id 都不可能被服务**。§15 的"5 个 id 全不中"由此得到解释，
而不是"帧格式又错了"。

### 16.2 CHRE host interface 是活的，而且认出了我们的 100 Hz

```
[5.199](0) hostintf: 5199799320, chreType:1, rate:102400, latency:0, cmd:0!
```

`chreType:1` = accel；`rate:102400`，按 1/1024 s 为单位 = **100 Hz**，
和 `en_seq2` / `en_seq3` 里的速率完全一致。

⇒ 我们的 CHRE enable **被正确解析了**。也就是说：
**类型、速率这条链是通的，断的是"FIFO 注册"那一步**，
而注册只能走 CHRE/nanoHub 的 configure 路径，不能走 MTK SET_CONFIG。

### 16.3 零 IPI 对照无效

```
run  0:   167 lines  t=0.026 .. 5.199 s   ← 含 legacy_handler，是"发过包"的旧实例
run  1:    65 lines  t=0.022 .. 0.082 s   ← 零 IPI 的新实例，30 s 只记到 0.082 s
```

新实例 30 秒只写到 0.082 s，说明它**很可能没真正起来**（而不是"被我们的包打死"）。
判据：`dmesg | grep "scp-start: started"` 有没有新增一条，
以及新 run 在 20 秒里有没有从 0.08 长到几秒。

`dmesg` 里最后三条 `SET_CONFIG`（pkt_id 2/3/4，host 52.4/61.9/67.5 s）
都在重启之**前**，所以这次实验对"包是否打死固件"没有信息量。

### 16.4 判据升级：不再看 `dram`，看 `legacy_handler` / `hostintf`

新的扫描判据（`port/tools/accel-pktid-sweep.sh`，v99 起也在 rootfs 里）：

| 固件日志 | 含义 |
| --- | --- |
| `pkt_id:N handler is null` | 到了 SCP，但 id N 没有 MTK 处理器 |
| `hostintf: ... rate:...` | 被 CHRE host interface 解析（活的那条路） |
| `dram ... maxEventNumber` | SET_CONFIG 真的执行了（目前从未出现） |

`ret 0` 永远不能当判据：它只代表 mbox 收下了写操作。

### 16.5 方向修正

原计划"扫 pkt_id 找到 SET_CONFIG"作废——那张表是空的。

新的主战场是 **CHRE 路径**：
固件已经认出了 `chreType:1 / rate:100Hz`，接下来要查的是
CHRE 侧如何让固件把样本写进 DRAM（nanoHub 的 configure/`chre_dram`），
而不是 MTK 的 `SET_CONFIG` action 4。这正好回到交接文档 §6.3 的方案 B。

### 16.6 v99 镜像

```
out-scp-v99-alpine/boot-stockdt.img   md5 453091e618d44b9e373a926e1a19b21e
```

rootfs 新增 `/usr/bin/chopin-pktid-sweep`（按 `legacy_handler`/`hostintf` 判读）。

---

## 17. 手动重启 SCP 这条路本身是坏的（2026-10-27 13:05）

### 17.1 数据

```
~ # cat /sys/class/misc/scp/scp_A_status
SCP A is not ready

~ # dmesg | grep "scp-start: started" | tail -2
[    4.971627] scp-start: started: c0 st=00000000 pc=0000034c | ...
[  165.525307] scp-start: started: c0 st=00000000 pc=0000033c | ...

~ # chopin-scplog -a     (两次，间隔 20 s，完全没变)
run  0:   167 lines  t=0.026 .. 5.199 s
run  1:    65 lines  t=0.022 .. 0.082 s
```

三个独立证据说明**第 2 次 `patchscp+fullreset+sram+start` 没能把 SCP 拉起来**：

1. `SCP A is not ready` —— 厂商驱动始终没收到固件的 ready IPI；
2. 第 2 次启动的 `pc=0000033c`，而第 1 次（能正常跑到 5 秒）是 `pc=0000034c`；
   `SCP_CORE_MON_PC` 读数不同，说明两次停的地方不一样；
3. 新实例（run 1）的日志停在 **t=0.082 s**，
   连 `ICM4x6xxInit`（0.086）都还没到，20 秒里一行没长。

⇒ **运行中的 SCP 不能靠 `fullreset + sram + start` 重启**。
需要干净状态就重启整机。这推翻了交接文档里"debugfs 重启用 `fullreset → sram → start`
让 loader 重新搬运"的隐含假设——那条只对**首次**启动成立。

### 17.2 因此零 IPI 对照必须靠刷机

`chsensor_zero_ipi` 是运行时参数，但 SCP 在 boot 时就被 local.d 放起来了，
中途改参数已经太晚，而重启又不可行。所以改成在 rootfs 里烘一个标记：

* `mkinitramfs.sh`：`CHOPIN_ZERO_IPI=1` 时写入 `/etc/chopin/zero-ipi`；
* `10-chsensor.start`：检测到该标记就在 patchscp **之前**
  `echo 1 > /sys/module/sensorhub/parameters/chsensor_zero_ipi`。

于是有了两个可对比的镜像，都从干净 boot 开始：

```
out-scp-v100-alpine/boot-stockdt.img   md5 43a92bc8d599ba2f61dbb69b3010f4d5   ← 正常发控制包
out-zeroipi-alpine/boot-stockdt.img    md5 a2be7e9cdae047c8b355f4cb6f3ed146   ← 一个包都不发
```

### 17.3 顺带：boot 日志现在自带固件日志

`10-chsensor.start` 结尾追加了：

```
--- firmware log runs ---   (chopin-scplog -a)
--- firmware log tail ---   (chopin-scplog -n 10)
```

跑完约 t≈25 s 自动写进 `/tmp/chsensor.log`，**下次不需要再手打命令**，
一次 boot 就能拿到全部证据：

```sh
sed -n '/firmware log runs/,$p' /tmp/chsensor.log
```

### 17.4 下一轮唯一要看的对比

| 镜像 | 固件日志末端 |
| --- | --- |
| `out-zeroipi-alpine` | 若**越过** t=5.2 s → 我们的包确实让固件停止工作 |
| `out-zeroipi-alpine` | 若**仍停在** t=5.2 s → 固件 boot 后本来就不写日志，包被静默丢弃 |

固件时间 t=5.198 对应 AP 时间 10.97 s（日志行自带 `sync time scp:..., ap:...`），
而 boot 时那次 `SET_CONFIG` 正好落在 AP ≈11 s —— 所以这个对比是干净的。

### 17.5 附带修正 §16 的一个疑点

§16 里 `legacy_handler ... pkt_id:0` 出现在固件 t=5.198 = AP 10.97 s，
而那一轮的 id 扫描是在 AP 42–67 s 才做的。也就是说：

* 这行 `pkt_id:0` **不是**扫描产生的；
* 那一轮扫描在 AP 42–67 s 做的 5 次发送，固件日志里**一行都没有**——
  说明固件在 AP ≈11 s 之后就没再记录任何东西。

这与"首次发包后固件主循环停止、但 mbox ISR 仍在取包（所以 `ret 0` 且无 timeout）"
一致，也正是 §17.4 那个对比要确认的。

---

## 18. 完整固件日志：阻塞点是 `jitter check`，不是协议（2026-10-27 13:06）

两组各 dump 完整日志（对照组 167 行、零 IPI 138 行），把之前被截断的
0.638–0.907 段补全了。

### 18.1 对照组的完整链条

```
0.602  sensorPowerGyro on:1, nowOn:0
0.602  accGyro, delay=1797783552 ... watermark:0 < 1
0.604  sensorPowerAcc on:1, nowOn:0
0.605  ICM4X6XXgEnable power mode=0, time:605228693     ← 上电写的是 mode 0
0.605  ICM4X6XXAccPowerOff
0.606  accGyro: spi error (-1)                          ← SPI 读失败
0.606  ICM4X6XXAccPowerOff, rx inSize and elemSize error ← FIFO 尺寸不匹配
0.636  acc: enable done
0.637  sensorRateAcc rate:409600 ...                    ← 默认 400 Hz，连续三次
0.639  accGyro, delay=2499584 ...                       ← 这次有合法 delay
0.640  acc require Rate 512000, min rate 12800,acc hw rate 0
0.640  ICM4X6XXgRate: acc config Rate 512000 reg f      ← 真配了 512 Hz
0.640  ICM4X6XXCalcuWm, watermark 1,min_watermark 1,total_cnt 1
0.640  ICM4X6XX watermarkReg 16
0.642  acc: ratechg done                                ← ★ 配置链路完全成功
0.648  mtk_type:1, status: 3, offset: [0,0,0]
0.895  jitter check failed even 4 retries:0             ← ★ 等不到 data-ready
0.898  sensorPowerAcc on:0, nowOn:1                     ← 框架关闭
0.901  accGyro, delay=1797783552 ... watermark:0 < 1
0.901  ICM4X6XXAccPowerOff
0.901  ICM4X6XX set power mode=2                        ← 写 mode 2（关）
0.904  acc: disable done
0.907  logger_update
        ...静默 4.4 秒...
5.293  legacy_handler id 11,pkt_id:0 handler is null    ← 我们的包
5.294  hostintf: ..., chreType:1, rate:102400, cmd:0    ← CHRE 解析 100 Hz 成功
       （之后什么都没有）
```

### 18.2 三条硬结论

1. **速率/水位这条链是通的**：`acc require Rate 512000` → `CalcuWm watermark 1`
   → `watermarkReg 16` → `acc: ratechg done`。所以"固件拿不到合法速率"
   这个猜想**排除**——它自己就能配到 512 Hz、watermark 1。
2. **芯片没有 data-ready**：`jitter check failed even 4 retries:0`
   （0.642 配好 → 0.895 放弃，等了 0.25 s）。
   随后框架主动 `set power mode=2` + `acc: disable done`。
   **这正是交接文档 §0.28 的结论，现在有了固件侧的原文证据。**
3. **我们的 CHRE 包落在"已经放弃之后"**：`hostintf` 在 5.294 s，
   而加速计 0.904 s 就被关了。它解析成功、速率正确，但**没有后续动作**——
   因为传感器框架早已进入 disable 状态。

⇒ **整条 MTK 协议 / mpool / CHRE 的工作都在"芯片能否出 data-ready"的下游。**
    协议侧到此已经基本查清（`legacy_handler` 表空、CHRE 解析正常）。

### 18.3 附带发现：固件会自己卡死

零 IPI 那组（138 行）的最后一行是**被截断的**：

```
[0.637](0) sensorRateAcc rate:409600, latency:2499584
[0.638](0) accRateCalculate pending by:36
[0.638](0) sensorRateÿÿÿÿÿÿÿÿ...        ← 写到一半，后面全是 0xFF
```

对照组同一位置是完整的 `sensorRateAcc rate:409600, latency:2499584`。
也就是说零 IPI 那次**固件在打这条日志的中途停住了**——典型的
"持着 log 锁卡死"。而且这次**我们一个包都没发**。

⇒ 固件的早期初始化本身就**偶发卡死**，这与交接文档里
    "只在部分 boot 采样" 的随机性来源一致，和我们的协议无关。

### 18.4 下一刀：把 `jitter check` 这个判决点废掉

现在的实验设计已经很干净：固件在 0.895 s 因为 jitter 判定失败而主动关芯片。
如果把它改成"失败也继续"，加速计就会保持 512 Hz/watermark 1 的armed 状态，
FIFO drain 就会跑起来。这时只有两种结果：

| `wp` | 结论 |
| --- | --- |
| 仍然 0 | 芯片确实完全不产 data-ready → 去查 INT1_CTRL / PWR_MGMT0 / FIFO_CTRL3 |
| 开始动 | 芯片一直在出数，只是固件放弃太早 → 直接解决 |

这条实验比继续调协议有价值得多，因为它能把"芯片有没有数据"一次性分开。

固件侧已经定位到的地址（供后续打补丁用）：

```
0x26754  "ICM4X6XXAccPowerOff, rx inSize and elemSize error" 的引用点
0x284c4  "ICM4X6XX set power mode=%d" 的引用点，紧跟
         0x284d2 c.beqz / 0x284d4 bne s0,2 / 0x284d8 beq s0,3 的分支判定
0x63473  "jitter check failed even 4 retries:%u" 字符串（引用点未定位）
0x718d3  "ICM4X6XXgEnable power mode=%d, time:%lld"
0x61fc2  "ICM4X6XXAccPowerOff, rx inSize and elemSize error"
```

`0x63473` 的引用点用 `lui+addi` / `auipc+addi` 两种模式扫过全镜像都没找到，
说明它的调用点可能不在 `scp_code.bin` 这张平面里（或用了别的取址方式），
下一轮需要专门的定位。

---

## 19. 现状评估：离"稳定"还有多远（2026-10-27 13:20）

### 19.1 已经稳的（内核侧，不用再动）

| 项 | 证据 |
| --- | --- |
| 不再崩溃 | 多轮 boot 无 oops / list_del / timer 报错；`GET_DATA` 刷屏与间隔漂移消失 |
| FIFO 读取正确 | 44 对齐校验 + 自愈；环形前向距离；倒退 resync；不回放残留 DRAM |
| bring-up 有界 | 停在 attempt 6；出数即永久停发；SCP 重启后自动重发 SET_CONFIG |
| IIO 接口 | `in_accel_{x,y,z}_raw` + `scale 0.001` + DT `mount-matrix`，历史上出过数（z≈9940） |
| 工具链 | `scp-chopin/log`、`chopin-scplog`、`chopin-pktid-sweep`、`chopin-accel-test`、boot 自动启动 SCP |

内核/协议侧**已经不是在等完善，而是在等一个样本**。

### 19.2 唯一的阻塞点（固件侧）

```
0.168  accGyro: init done              ← CHRE host interface 可用
0.604  sensorPowerAcc on:1, nowOn:0    ← 框架要速率，拿到的是垃圾默认 delay=1797783552
0.640  ratechg done (512 Hz, watermark 1)   ← 硬件配置成功
0.895  jitter check failed even 4 retries   ← 等不到 data-ready
0.904  acc: disable done                    ← 自己关掉
5.293  我们的 CHRE enable 才到（hostintf rate:102400）→ 无事可做
```

⇒ 加速计在 **0.9 秒**就被固件自己关掉了，而我们的 enable 在 **5.3 秒**才到。

### 19.3 原厂对照与实际工作量

* AP 侧没有 accel 节点（`mt6893-xiaomi-chopin*.dts*` 里搜不到 `icm4|accel`），
  `acc spi_num: 2` 是固件自己的编号 ⇒ **AP 直接驱动这颗芯片这条路基本不成立**；
* `work/accint/dmesg_scp_stock.txt` 是 WiFi/conninfra 的 dump，没有 accel 内容，
  所以还没有"能工作的固件日志"可对照。

### 19.4 固件补丁的可行性边界（诚实结论）

`jitter check failed`（0x63473）的**调用点找不到**：`lui+addi`（同/异目的寄存器）、
`auipc+addi`、任意 `addi imm=0x473`、地址字面量（对齐与非对齐）全部为 0 命中。
同池邻居 `ICM4X6XXAccPowerOff`(0x61fc2) → 0x26754、
`set power mode`(0x6349a) → 0x284c4 都能找到 ⇒ 不是全局偏移问题。

⇒ 想打 `jitter check` 补丁，需要递归下降反汇编器（`dec2.py` 是线性解码，
在 0x28460 已出现大量 `<unknown>`），这是一个**有边界但实打实的工作量**。

### 19.5 转而先试一个更便宜、更可能奏效的假设

既然 `hostintf` 的解析和速率都正确、只是**到得太晚**，
那就在固件放弃之前把它递进去：

`10-chsensor.start` 在 `start` 之后 0.4 s 触发一次 `retry`
（`retry` 约 100 ms 内重跑 bring-up），使包落在固件时间 **0.3–0.5 s** 的窗口内。

判据（`/tmp/chsensor.log` 自动记录）：

| `run 0` 里出现 | 结论 |
| --- | --- |
| `ratechg done` 之后是 **100 Hz** 而不是 512000，且没有 `jitter check failed` | ✅ 框架接受了我们的速率 → 继续看 `wp` |
| `hostintf` 早出现（t~0.4）且随后有 `AccPowerOn` | ✅ 框架重新上电 |
| 仍然 `jitter check failed` | 速率不是问题，是芯片真的不产 data-ready |

```
out-scp-v102-alpine/boot-stockdt.img   md5 186d2f038c4179663b1bbab76c5b8e22
```

### 19.6 距离评估（按"还需要解决的问题个数"而不是时间）

| 阶段 | 状态 |
| --- | --- |
| 内核驱动 / IIO / 不再崩溃 | ✅ 完成 |
| 协议诊断（mpool 帧格式、legacy handler、CHRE 解析） | ✅ 完成，已排除 |
| **让芯片在窗口内接受速率** | ⏳ 一次实验（v102） |
| **让芯片真的产出 data-ready** | ❓ 取决于上一步结果 |
| 样本落进 0x8F0CA700 且 `wp` 前移 | 内核侧已就绪，等数据 |
| iio-sensor-proxy + 自动旋转 | 内核侧已就绪，等数据 |

**如果 v102 这一刀奏效**：剩下的只是打包 userspace，很近。
**如果仍然 `jitter check failed`**：问题坐实在芯片侧（PWR_MGMT0/INT1/FIFO 配置），
需要打固件补丁，那就是"一个反汇编器 + 一到两个补丁"的距离，不再是协议问题。

---

## 20. 突破：CHRE enable 生效了，固件开始大量活动（2026-10-27 13:15）

### 20.1 v102 实测：包里进了窗口（虽然仍偏晚），而且固件动了

```
[7.469947] SET_CONFIG fifo 0x8f0ca700 size 1048520 ret 0 (pkt_id 3, mode 0)
[7.470831] DISABLE accel (seq 1)
[7.789956] ENABLE accel 100Hz (seq 2, re-arm)
[13.037942] ENABLE accel (seq 3, retry 2)
[17.133908] ENABLE accel (seq 3, retry 3)
[21.229905] ENABLE accel (seq 3, retry 4)
[25.325918] ENABLE accel (seq 3, retry 5)
```

固件日志：

```
run  0:   168 lines  t=0.026 .. 1.691 s     ← 含 legacy_handler + hostintf
run  1:  1904 lines  t=15.465 .. 33.433 s   ← ★ 从来没有过的量
```

`t=1.688` 的 `hostintf: ..., chreType:1, rate:102400` 对应 AP 7.45 s，
说明早发的那次确实递进去了（只是落在 1.69 s 而不是目标 0.3–0.5 s，
因为 `is_scp_ready()` 门控到 AP 7.3 s 才放行）。

**但 `run 1` 是关键**：1904 行、18 秒、**≈106 行/秒 ≈ 100 Hz**，
起始 `t=15.465` 换算到 AP ≈21.2 s——正是 `ENABLE accel (seq 3, retry 4)`。

⇒ **CHRE enable 生效了，加速计开始按 100 Hz 动起来了。**
这是整轮工作里第一次看到固件以数据速率活动。

### 20.2 但 `wp` 仍然是 0

```
[  166.466052] scp-chopin: fifo rp=0 wp=0 size=1048520 records=23830
```

原因很清楚：**FIFO 从来没有被注册**。
`SET_CONFIG`（pkt_id 3）走的是那张空的 legacy 表
（`legacy_handler id 11,pkt_id:0 handler is null`），
所以固件的 `chre_dram` 指针从未被设成我们的 `0x8F0CA700`。
样本产生了，但没有落点。

⇒ 阻塞点从「芯片不出数据」变成了「**数据出了但没地方放**」。
这是**实质性的降级**：协议不再是黑盒，缺的是 FIFO 注册那一步。

### 20.3 立刻要看的东西

`run 1` 的 1904 行到底是什么（`ICM4X6XXSample` 每样本一条？还是错误刷屏？）。
v102 的脚本还没有 `-r`，用这一行读（v103 起有 `chopin-scplog -r 1`）：

```sh
python3 -c 'import mmap,os;b=0x8F1CA700;p=b&~4095;f=os.open("/dev/mem",os.O_RDONLY|os.O_SYNC);m=mmap.mmap(f,0x40000,mmap.MAP_SHARED,mmap.PROT_READ,offset=p);print(m[b-p:b-p+0x3f000].decode("latin1","replace"))' | grep -aoE "\[[0-9]+\.[0-9]+\]\([0-9]\) .*" | sed -n '169,200p'
```

### 20.4 v103

```
out-scp-v103-alpine/boot-stockdt.img   md5 6134299fc2b7d7e6b2a56f824f2b7b8f
```

`chopin-scplog` 新增 `-r N`（负数从末尾数，`-r -1` = 最新一段）；
boot 钩子自动 dump `-r -1 -n 25`，以后一次开机就能看到最新活动。

---

## 21. 根因定死：`dram info not got, exit write dram`（2026-10-27 13:30）

### 21.1 v102 的固件日志把最后一步露出来了

```
[15.465](0) hostintf: ..., chreType:1, rate:102400, latency:0, cmd:1!
[15.465](0) sensorPowerAcc on:1, nowOn:0
[15.499](0) sensorRateAcc rate:102400, latency:9999360
[15.501](0) acc require Rate 102400, min rate 12800, acc hw rate 0
[15.501](0) ICM4X6XXgRate: acc config Rate 102400 reg 8
[15.502](0) ICM4X6XXCalcuWm, watermark 1,min_watermark 0,total_cnt 1
[15.502](0) ICM4X6XX watermarkReg 16
[15.502](0) ICM4X6XXConfigFifo: Reset, TH_L:0x10, TH_H:0x0
[15.504](0) acc: ratechg done
[15.512](0) dram info not got, exit write dram        ← ★
```

**加速计被完整配置成功**：100 Hz（`reg 8`）、watermark 1、FIFO 阈值 16、
`ratechg done`，这次连 `jitter check` 都没失败。
然后 drain 要往 DRAM 写时发现没有目标，**直接退出，一个字节都没写**。

### 21.2 代码级定位

`get dram phy addr` 字符串引用点是 **0x3b03c**，与 reK.md 推断的 SET_CONFIG
打印点完全一致，说明反汇编偏移体系正确。drain 路径：

```
3ab7e: lui  s0, 0xaa
3ab82: lw   a0, -376(s0)      ; a0 = *(0xA9E88) = chre_dram
3ab86: c.beqz a0, 0x3ac06     ; 为空 -> 0x3ac06
...
3ac06: lui  a0, 0x65
3ac0a: addi a0, a0, -294      ; "dram info not got, exit write dram"
3ac0e: jal  0x3fc82           ; printk
3ac12: li   a0, -1
3ac14: c.j  0x3acac           ; return -1  ← 样本就此丢弃
```

而 `chre_dram`（0xA9E88）**只有 SET_CONFIG 会写**（`0x3b028: sw a0, 0xa9e88`），
SET_CONFIG 又因为 `pkt_id 3` 落进空 legacy 表而永远不可达。

⇒ 整条链只差"把 FIFO 地址告诉固件"这一步，而这一步在本固件上不可达。

### 21.3 绕过：把 drain 的目标直接改成我们的页

两个读点都是单条 `lw X, -376(Y)`，只负责取一个 32 位指针，
所以每条都能换成一条 `lui`：

| 偏移 | 原 | 新 | 作用 |
| --- | --- | --- | --- |
| `0x3AB82` | `03 25 84 e8`（`lw a0,-376(s0)`） | `37 d5 0c 9f`（`lui a0,0x9F0CD`） | 让判空通过 |
| `0x3ACCC` | `03 2b 85 e8`（`lw s6,-376(a0)`） | `37 db 0c 9f`（`lui s6,0x9F0CD`） | drain 直接写到 0x9F0CD000 |

`0x9F0CD000` 是 SCP 侧地址，对应 AP `0x8F0CD000`——位于 SENS 块内
（`0x8F0CA700 .. 0x8F1CA700`）的一页，没有别的块占用。

已加进 `patchscp` 作为 **group 5**（`echo 5 > patchscp`）。

### 21.4 配套的内核改动

`SENS_FIFO_BASE` 从 `0x8F0CA700` 改为 **`0x8F0CD000`**，并且全部尺寸改为推导：

```
SENS_MEM_BASE = 0x8F0CA700
SENS_MEM_SIZE = 0x100000
SENS_FIFO_END = 0x8F1CA700            （SCP logger 起点）
FIFOSize      = ((END - BASE - 0x10) / 44) * 44 = 0xFD6E0 = 1038048（23592 条）
MAP_SIZE      = 0xFD6F0 → ioremap 0xFE000
```

`scp-chopin/fifo` 调试节点和 boot 钩子里的 `devmem` 地址同步更新。
boot 钩子改调 `echo 5 > patchscp`。

### 21.5 v104

```
out-scp-v104-alpine/boot-stockdt.img   md5 1a7583aae987e35917158efc3002eb1f
```

### 21.6 判据

刷上后看两处：

```sh
chopin-scplog -r -1 -n 20 | grep -E "dram|Sample|ratechg"
echo 1 > /sys/kernel/debug/scp-chopin/fifo; dmesg | tail -5
```

| 结果 | 结论 |
| --- | --- |
| `dram info not got` 消失，`wp` 开始动 | ✅ 打通 |
| 仍有 `dram info not got` | 补丁没生效（看 patchscp 输出 expected/got） |
| 出现 `ICM4X6XXSample` 且 `wp` 动 | 完全打通，可以进 IIO 验证 |
| SCP 卡死/重启 | `lui` 补丁打破了什么，需要回退 group 5 |

---

## 22. v104 把 SCP 打死了：漏了三个读点（2026-10-27 13:35）

### 22.1 现象

刷 v104 后：

```
[   17.620155] scp-chopin: SCP not ready, fifo dump skipped
[   17.620457] Error: scp_ipidev IPI 11 timeout (last done is IPI 17)
[   45.106184] ipi_comm chsensor: SCP not ready, bring-up deferred   (之后一直重复)
```

SCP 从 boot 起就没 ready，`chopin-scplog` 也读不到东西。

### 22.2 根因：`chre_dram` 有 5 个读点，我只改 2 个

v104 只补了 `0x3AB82` 和 `0x3ACCC`。按 2 字节粒度重扫
（这段代码含 16 位压缩指令，4 字节对齐扫描会漏）：

```
0x39392  lw a0, -376(s0)   ← 漏
0x3AB82  lw a0, -376(s0)   ← 已补（判空）
0x3ABDE  lw a0, -376(s0)   ← 漏，且紧接着解引用
0x3ACCC  lw s6, -376(a0)   ← 已补（drain）
0x3B048  lw a0, -376(s1)   ← SET_CONFIG 的日志，不可达，不动
```

`0x3ABDE` 与 `0x39392` 的模式都是**下一条就解引用**：

```
3abde: lw  a0, -376(s0)     ; a0 = chre_dram = 0
3abe2: c.lw s0, 4(a0)       ; *(0+4) → SCP 里 NULL 解引用
```

所以 v104 把「drain 提前退出」变成了「运行期崩」，固件再也起不来。

**教训**：这类"改一个全局的读取点"的补丁，必须先把**所有**读点枚举干净
（按 2 字节步进 + 只取 `(w & 3) == 3` 的 32 位指令），否则补一半比不补更糟。

### 22.3 v105：补齐四条，并把 group 分层改对

| 偏移 | old | new | 作用 |
| --- | --- | --- | --- |
| `0x39392` | `03 25 84 e8` | `37 d5 0c 9f`（`lui a0,0x9F0CD`） | 另一处读点 |
| `0x3AB82` | 同上 | 同上 | 判空通过 |
| `0x3ABDE` | 同上 | 同上 | 解引用不再为 NULL |
| `0x3ACCC` | `03 2b 85 e8` | `37 db 0c 9f`（`lui s6,0x9F0CD`） | drain 直写 |

四条 `old` 都已与 `work/scp_code.bin` 逐字节核对一致。

**group 重新分层**（这是 v104 的第二个问题）：

| group | 内容 |
| --- | --- |
| 0/1 | 原有 base + power gate（此前一直用 `echo 2`，即只有这两组） |
| **2** | **drain 重定向（本节的四条）** |
| 3 | 芯片侧 INT1 极性（原来在 group 2） |

v104 用 `echo 5` 会把 group 3（极性）也带上，而**极性补丁从未出现在任何一次
成功出数的 boot 里**。所以 v105 的钩子用 `echo 3`：正好等于
「此前稳定的 `echo 2`」+「drain 重定向」，不多不少。

boot 钩子还会把 `dmesg | grep patchscp` 打进日志，方便确认每条的 expected/got。

### 22.4 v105

```
out-scp-v105-alpine/boot-stockdt.img   md5 d5530e9ad71195bc6154c533f8459dd2
```

---

## 23. 为什么"以前能出数、但不稳定"——我改坏的那一处（2026-10-27 13:40）

### 23.1 用户的问题点出了矛盾

历史上（v40–v89）加速度计**确实出过数**（`z ≈ 9940`，`wp` 会走）。
但今天我们已经证明：`chre_dram` 从未被设置 → 固件 drain 打印
`dram info not got, exit write dram` → `wp` 永远是 0。

如果 SET_CONFIG 从来没成功过，**历史上的 `wp` 就不可能走动**。
所以 SET_CONFIG 当年**是成功过的**。

### 23.2 根因：我把 SET_CONFIG 从"每轮重发"改成了"每 session 一次"

`59012e451` 之后的版本里，我把 `SET_CONFIG`/`SET_CUST`/`SET_TIMESTAMP`
一起收进了 `if (!config_done)` 块，只发一次。而**旧版是每轮 pass 都重发整套序列**
（`chsensor_enable_work` 每 4~5 s 一轮，最多 6 轮）。

关键在于：**固件的 mpool legacy handler 是在它自己 boot 后一段时间才注册的**。
证据就是那一行：

```
legacy_handler id 11,pkt_id:0 handler is null!!! prdata:0x0 data:0xbe1cc len:136
```

它出现在固件时间 **1.69 s** —— 也就是我们现在唯一那次 SET_CONFIG 的落点。
之后我们的代码再也不发 SET_CONFIG，于是**永远没有第二次机会**。

旧版每轮都发，所以 3~6 轮里总有一发落在"handler 已注册"之后 →
`chre_dram` / `maxEventNumber` 被设上 → `wp` 开始走 → 出数。
这就是交接文档里"需要 3~6 次才出数"的真实机制：**不是概率，是注册时机**。

### 23.3 "不稳定"是另一批 bug（已修）

当年"能出数但方向乱跳/不稳定"的原因，是这次复核里已经修掉的那些：

| 编号 | 问题 | 状态 |
| --- | --- | --- |
| §1 | FIFO 头被 `share_mem_init` 写成 36 字节几何 → 首次回绕后 44 字节记录错位 | 已加 44 对齐校验 + 自愈 |
| — | 旧版 **每轮都把 wp 清零**（`chsensor_fifo_write_header` 在每轮 config 里调用）→ 正在流的数据被拽回去 | 现在只在首轮写头 |
| §2.3 | delayed work 反复 re-init → timer wheel 损坏 → 崩溃 | 已修 |
| §2.4 | `mpool_send_pkt` 静态缓冲竞态 | 已修 |
| §18.3 | 固件自身偶发卡死（无包也会停在 ~0.64 s） | 固件侧，未解 |

所以两件事要分开：
* **"能出数"** 依赖 SET_CONFIG 在 handler 注册之后落地 → 需要**每轮重发**；
* **"不稳定"** 是驱动侧的错位/回绕/清零/timer 问题 → 已经修完。

### 23.4 v106

* SET_CONFIG 恢复为**每轮 pass 重发**（`SET_CUST`/`SET_TIMESTAMP` 仍只发一次）；
* FIFO 头仍然**只写一次**（不重复清零 wp）；
* `patchscp` 回到 **`echo 2`**（base + power gate）——drain 重定向补丁不再需要，
  因为 SET_CONFIG 成功后固件会自己拿到真正的目标；
* FIFO 基址保持 `0x8F0CD000`，SET_CONFIG 报的也是这个地址，两边一致。

```
out-scp-v106-alpine/boot-stockdt.img   md5 0dbf2b88223a55ba421e29220855d41b
```

### 23.5 判据（这次有直接反馈）

```sh
grep -E "SET_CONFIG fifo" /tmp/chsensor.log          # 应该出现多次（pass 1..N）
chopin-scplog -g 'handler is null|dram|ratechg'      # 看 handler 是否还报 null
echo 1 > /sys/kernel/debug/scp-chopin/fifo; dmesg | tail -3
```

| 现象 | 结论 |
| --- | --- |
| `SET_CONFIG` 出现多次，且 `handler is null` 某次之后不再出现 | ✅ 注册成功，`wp` 应该开始走 |
| 始终每次都 `handler is null` | handler 注册比 6 轮还晚 → 加大 attempt 上限 / 拉长间隔 |
| `SET_CONFIG` 有 `ret -EIO` | mbox 堵了，看 `IPI 11 timeout` |

---

## 24. 通了：v106 实测加速度计稳定出数（2026-10-27 13:50）

### 24.1 固件侧

```
[1.698](0) legacy_handler id 11,pkt_id:0 handler is null!!! ... len:136
[1.699](0) get dram phy addr=0x9f0cd000,size=1038048, maxEventNumber:23592
[1.699](0) get dram phy rp=0,wp=0
[1.701](0) get dram phy addr=0x9f0cd000,size=1038048, maxEventNumber:23592
[1.701](0) get dram phy rp=0,wp=0
```

第一条 `handler is null` 是**第一次** SET_CONFIG 撞在注册之前；紧接着的第二、第三次
就成功执行了（`get dram phy addr=...` 就是 SET_CONFIG 入口 0x3b03c 的打印）。
这直接验证了 §23 的结论：**差别只在"重发"**。

`0x9f0cd000 / 1038048 / 23592` 与内核侧 `SENS_FIFO_BASE=0x8F0CD000`、
`SENS_FIFO_BYTES=0xFD6E0` 完全一致。

### 24.2 内核侧

```
[30.164768] rec-1 off=98472 type=65536 ts=24441841290 x=490  y=-241 z=9969
[30.165797] rec-2 off=98428 type=65536 ts=24431841288 x=478  y=-241 z=9964
[30.166843] rec-3 off=98384 type=65536 ts=24421840672 x=495  y=-239 z=9945
```

| 项目 | 数值 | 判读 |
| --- | --- | --- |
| x/y/z | ≈490 / −241 / 9969 | z ≈ g，与历史 9940 同量级 |
| ts 间隔 | 10,000,002 ns | **10 ms = 100 Hz** |
| off 步进 | 98472 → 98428 → 98384 | 连续记录，环形步进正确 |
| type | 65536 = 0x00010000 | 与该固件的已知行为一致，代码不据此过滤 |

IIO 读数（10 次，间隔 1 s）：

```
497 -167 9959   481 -227 9964
488 -177 9921   483 -237 9967
514 -220 9957   481 -241 9947
483 -237 9957   471 -222 9964
481 -227 9957   486 -227 9952
478 -237 9957
```

`scale = 0.001` ⇒ x≈0.48、y≈−0.22、z≈**9.95 m/s²**；抖动 ±0.03 m/s²，方向稳定。

### 24.3 最终可工作的配置

| 项 | 值 |
| --- | --- |
| 镜像 | `out-scp-v106-alpine`（md5 `0dbf2b88223a55ba421e29220855d41b`） |
| patchscp | `echo 2`（group 0 base + group 1 power gate，**不含** drain 重定向和 INT1 极性） |
| FIFO 基址 | `0x8F0CD000`（SCP `0x9F0CD000`），`FIFOSize=0xFD6E0=1038048`（23592 条） |
| SET_CONFIG | **每轮 pass 重发**（关键） |
| FIFO 头 | **只写一次**，不重复清零 wp |
| 内核 | `7.3.0-g59012e451bb9-dirty` |

### 24.4 剩下的两件事（都与传感器无关）

1. **白屏**：本 boot 仍有 3 条 `refcount_t` 警告，出自
   `drm_panel_get -> drm_panel_add -> lcm_probe` / `of_drm_find_panel -> mtk_dsi_probe`。
   `mediatek-drm` 本身注册成功（`fb0: mediatekdrmfb`），但 panel 的 refcount 被
   saturate（永久泄漏）后，面板很可能没被正确初始化 → 白屏。
2. **固件偶发卡死**：部分 boot 会停在 ~0.64 s（无任何 IPI 也会），
   表现为该次 boot 不出数；重试/重启即可。

### 24.5 下一步：打包 userspace 验证自动旋转

内核侧已全部就绪，剩下是用户态：

```sh
dbus-daemon --system --fork
/usr/libexec/iio-sensor-proxy &
monitor-sensor
```

`monitor-sensor` 不需要屏幕即可看到 orientation 变化（Accelerometer orientation
changed: normal / left-up / right-up / bottom-up）。

---

## 25. 自动旋转闭环：发现一个「双重重映射」隐患（2026-10-27 14:05）

### 25.1 内核侧已经齐了

* `sensorhub_iio` 三个 accel 通道都挂了
  `ext_info = sensorhub_accel_ext_info`，
  里面是 `IIO_MOUNT_MATRIX(IIO_SHARED_BY_TYPE, ...)`；
* 所以 `/sys/bus/iio/devices/iio:device1/in_accel_mount_matrix` 应该直接可读，
  内容来自 DTS 的 `mount-matrix`；
* rootfs 里 `iio-sensor-proxy` / `dbus-daemon` / `monitor-sensor` 都已存在
  （v106 起就有，**不需要重刷就能测**）。

### 25.2 隐患：固件已经做过一次同样的重映射

固件 boot 时自己打印：

```
[0.087](0) acc map[0]:1, map[1]:0, map[2]:2, sign[0]:1, sign[1]:-1, sign[2]:1
```

即 `out0=+in1, out1=-in0, out2=+in2` —— 这**和 DTS 里写的 mount-matrix
`0 1 0 / -1 0 0 / 0 0 1` 是同一个映射**。

也就是说：芯片原始轴已经被固件转过一次，如果内核 IIO 层再套同一个矩阵，
等于把轴再转 90°，方向判定会错。实测数值验证了这个风险：

```
芯片原始 (469, -217, 9938) -> 固件输出 (-217, -469, 9938) -> 再套 DT 矩阵 (-469, 217, 9938)
```

`z` 因为矩阵对角是 1 所以不受影响（这也解释了为什么平放读数看起来完全正常）。

⇒ **两者只能留一个**：
* 要么 DTS 的 `mount-matrix` 改成单位矩阵（固件已经校正过）；
* 要么把固件那次映射当"未知"、由内核校正。

正确做法只能靠实测：用 `chopin-orientation -i` 看固件给出的原始帧，
再把手机分别竖持/横持/倒置，比较两组的轴对应关系。

### 25.3 新增两个用户态工具

| 脚本 | 作用 |
| --- | --- |
| `/usr/bin/chopin-rotation` | 起 system D-Bus + `iio-sensor-proxy`，然后跑 `monitor-sensor` |
| `/usr/bin/chopin-orientation` | 纯 python：读 raw + scale + mount matrix，自己算 display 帧并打印 orientation；`-i` 忽略 DT 矩阵，`-w` 每秒一行 |

### 25.4 v108

```
out-scp-v108-alpine/boot-stockdt.img   md5 3ecaaced7fa76b1de1654596b1f9cc54
```

## 26. 用户态两个坑（已修）：EAGAIN 与 dbus 的 messagebus 用户（2026-10-27 18:00）

### 26.1 `in_accel_*_raw` 在首样本前返回 -EAGAIN

```
BlockingIOError: Read returned None.
```

来源是 `sensorhub_iio.c:283`：

```c
	mutex_lock(&st->lock);
	if (!st->has_data) {
		mutex_unlock(&st->lock);
		return -EAGAIN;
	}
```

`has_data` 由第一次 `sensorhub_iio_push()` 置位，所以 boot 后/SCP 复位后有
一个短暂的"无数据"窗口。Python 把这个 `-EAGAIN` 表现成
`BlockingIOError: Read returned None`。

修法：`chopin-orientation` 的 `rd()` 加重试（40 × 50 ms）。

### 26.2 `dbus-daemon --system` 需要 `messagebus` 用户

```
dbus-daemon[264]: Failed to start message bus: Could not get UID and GID for username "messagebus"
```

这个 rootfs 是从 Alpine 包解出来的，**post-install 脚本没跑过**，
所以 `/etc/passwd` / `/etc/group` 里没有 `messagebus`。
已补上（按 Alpine 约定 uid=100 / gid=101）：

```
/etc/passwd: messagebus:x:100:101:Linux D-Bus Message Broker:/run/dbus:/sbin/nologin
/etc/group : messagebus:x:101:
```

`chopin-rotation` 也改成失败时打印明确提示并回退到 `chopin-orientation`。

### 26.3 关于 uuid / machine-id

`dbus-daemon --system` 还要求 `/etc/machine-id`（或 `/var/lib/dbus/machine-id`）
且 `/run/dbus` 可写；`chopin-rotation` 都会自己建。

### 26.4 已确认的事实

```
$ cat /sys/bus/iio/devices/iio:device1/in_accel_mount_matrix
0, 1, 0; -1, 0, 0; 0, 0, 1
```

**DTS 的 mount matrix 确实被正确暴露成 IIO 属性**，iio-sensor-proxy 会用到它。

### 26.5 还缺一个数据：竖持时的读数

目前所有样本都是平放（`disp z ≈ +9.94`，x/y 接近 0），
而 `z` 因为矩阵对角为 1 不受重映射影响，**不能用它判断 matrix 对不对**。

必须竖持/横持/倒置各取一行，才能确定：
* 若 `-i`（固件原始帧）在竖持时 `disp y` 最大 → 固件已经校正过 →
  DTS matrix 应改为单位矩阵；
* 若 `disp x` 最大 → 固件未校正 → 保留现有 matrix。

### 26.6 v109

```
out-scp-v109-alpine/boot-stockdt.img   md5 64abfceebaa693416336684f510abe89
```

## 27. 定案：mount matrix 必须是单位矩阵（2026-10-27 18:20）

### 27.1 决定性实测（`-i`，即固件原始帧）

```
平放   dev(  +0.97   +0.84   +9.83)
竖持   dev(  +0.73   +9.35   -0.14)      ← y 主导，z 掉到 0
竖持   dev(  +0.63   +9.67   +0.90)
横持   dev(-9552/1000  +0.26  -2.12)     ← 原始计数 x=-9552 y=260 z=-2121
```

关键点：**加速度计平放报 +9.8（不是 -9.8）**，说明它上报「朝上的那个轴为正」。
所以竖持时屏上方 = +y ⇒ **固件输出的原始帧已经就是 display 帧**。

### 27.2 再套 DTS matrix 会转错 90°

用实测值验算：

```
平放         套 DTS matrix -> disp( +0.84  -0.97  +9.83) = face-up   （碰巧对，z 不受影响）
竖持         套 DTS matrix -> disp( +9.46  -0.90  +2.36) = right-up  ✗ 应为 normal
横持(左高)   套 DTS matrix -> disp( +0.30  +9.50  -1.80) = normal   ✗ 应为 left-up
```

竖持/横持**全部错位一次**，与 §25.2 的推断完全一致：
固件已经按 `map[0]:1, map[1]:0, map[2]:2, sign[0]:1, sign[1]:-1, sign[2]:1`
做过一次校正，内核不能再做。

### 27.3 修改

`arch/arm64/boot/dts/mediatek/mt6893-xiaomi-chopin.dts` 的
`sensor_hub` 节点：`mount-matrix` 改为单位矩阵，并把理由写进注释
（含实测数值，避免以后有人"顺手改回去"）。

DTB 已确认：

```
sensor-hub {
        compatible = "mediatek,scp-sensorhub";
        mount-matrix = "1", "0", "0", "0", "1", "0", "0", "0", "1";
};
```

### 27.4 v110

```
out-scp-v110-alpine/boot-stockdt.img   md5 6ae21c395566f3e91e09aa5c62f5bfca
```

### 27.5 验证

```sh
cat /sys/bus/iio/devices/iio:device1/in_accel_mount_matrix   # 应为 1,0,0; 0,1,0; 0,0,1
chopin-orientation                                          # 不再需要 -i
```

竖持应打印 `normal`，顺时针横持应打印 `left-up`，平放 `face-up`。
两者一致 ⇒ mount matrix 与固件校正对齐，iio-sensor-proxy 可以直接用。

## 28. mount matrix 验证通过（六种姿态全对）(2026-10-27 18:10)

固件原始帧 + 单位矩阵，实测：

```
竖持      dev( +0.75   +9.11   +3.52)  normal
横持(右)  dev( +9.24   +1.59   +3.39)  right-up
横持(左)  dev( -9.48   +1.38   -0.20)  left-up
倒置      dev( -3.20   -9.21   +0.76)  bottom-up
平放      dev( -0.47   +0.04   +9.95)  face-up
面朝下    dev( -0.09   -2.28   -9.28)  face-down
```

用户确认「旋转状态与标注一致」。
⇒ §27 的结论（固件已自行校正、内核必须用单位矩阵）**得到完整验证**。

`/sys/bus/iio/devices/iio:device1/in_accel_mount_matrix` 现在读回
`1, 0, 0; 0, 1, 0; 0, 0, 1`。

### 28.1 新的断点：iio-sensor-proxy 找不到传感器

```
$ chopin-rotation
== starting dbus-daemon --system
== starting iio-sensor-proxy
iio-sensor-proxy did not stay up; see /tmp/iio-sensor-proxy.log
** Message: 00:08:49.321: No sensors or missing kernel drivers for the sensors. Exiting
```

dbus 已经能起了（`messagebus` 用户已补），问题在 iio-sensor-proxy 没有枚举到
我们的 IIO 设备。

最可能的原因（按可能性排序）：

1. **没有 udevd**：iio-sensor-proxy 走 GUdev，`udev_enumerate` 虽然能扫 /sys，
   但 GUdev 的 uevent monitor 需要 `/run/udev/control`，构造失败会导致
   一个设备都拿不到；
2. 设备名/属性不满足它的探测条件（它要 `in_accel_*_raw` + `in_accel_scale`，
   我们都有）；
3. 探测时读到 `-EAGAIN`（首样本前）被当成"驱动缺失"。

### 28.2 备选方案：不依赖 iio-sensor-proxy

对自动旋转来说，`iio-sensor-proxy` 只是"把 IIO 朝向翻译成 D-Bus 信号"的中间层。
如果它在无 udev 的 initramfs 里起不来，完全可以**自己写一个朝向守护进程**：

* 直接读 `/sys/bus/iio/devices/iio:device1/in_accel_{x,y,z}_raw` + `scale`
  + `mount_matrix`（`chopin-orientation` 已经在做这件事）；
* 状态变化时调用合成器的旋转命令（`wlr-randr --output ... --transform 90`）；
* 零依赖：不需要 dbus、不需要 udev、不需要 iio-sensor-proxy。

代价是只服务于我们自己的合成器，别的桌面程序看不到 `net.hadess.SensorProxy`。

## 29. 方案 A：自动旋转守护进程 + 清理 33 GB（2026-10-27 18:35）

### 29.1 磁盘清理

`out-*` 下 144 个构建目录合计 **34 GB**（每个约 200–360 MB，主要是解包的
`initramfs-root/`）。删除 142 个，只保留：

* `out-scp-v106-alpine` —— 首个稳定出数的配置（内核基线，留作回归对照）
* `out-scp-v110-alpine` —— 当时最新

删除前确认：都不是符号链接、没有被任何脚本引用、未被 git 跟踪。
结果：已用 211 G → **178 G**，可用 139 G → **172 G**（释放 33 GB）。

### 29.2 iio-sensor-proxy 为何不用了

设备侧一切正常：

```
$ ls /sys/bus/iio/devices/iio:device1/
buffer  buffer0  in_accel_mount_matrix  in_accel_sampling_frequency
in_accel_scale  in_accel_x_raw  in_accel_y_raw  in_accel_z_raw
scan_elements  trigger  ...
$ cat .../trigger/current_trigger      # 空
$ cat .../buffer/enable                # 0
```

`scan_elements/in_accel_{x,y,z}_{en,index,type}` 都在，只差一个 trigger。
iio-sensor-proxy 优先走 IIO buffer（需要 trigger），失败才退 poll；它一条
`Found IIO ... accelerometer` 都没打印就退出了。

但即使修好它也没用：**它只往 D-Bus 广播朝向，真正旋转屏幕的是合成器**，
而这套 initramfs 里没有合成器也没有 udev。所以直接自己做，不再依赖它。

### 29.3 `/usr/bin/chopin-autorotate`

纯 python3，零外部依赖：

* 读 `in_accel_*_raw` / `in_accel_scale` / `in_accel_mount_matrix`；
* 分类为 `normal / left-up / right-up / bottom-up`，另外：
  * `|z| > 7 m/s²` → `flat`（平放/翻面），**保持上一次旋转不变**（手机行为）；
  * 主轴不超过次轴 1.6 倍 → `unclear`，不动作（避免乱晃时乱转）；
  * 需要连续 3 次一致才生效（去抖）；
* 状态变化时调合成器：

| backend | 命令 |
| --- | --- |
| `swaymsg` | `swaymsg output <name> transform <t>` |
| `wlr-randr` | `wlr-randr --output <name> --transform <t>` |

`BACKEND=auto` 依次尝试；输出名自动探测（`swaymsg -t get_outputs -r` 取
active，或 `wlr-randr` 第一列）；合成器还没起来就每 0.2 s 重试，**命令行
环境下无害**。

校准：`chopin-autorotate -t normal|left-up|right-up|bottom-up`，
若某个方向转了 90° 反向，改 `/etc/chopin/autorotate.conf` 的
`MAP=normal,270,90,180`，不必改代码。

分类逻辑用实测值自测过：

```
竖持 normal · 横右 right-up · 横左 left-up · 倒置 bottom-up
平放 flat · 面下 flat · 临界(6.5,7.2) unclear
```

### 29.4 新增文件

```
/usr/bin/chopin-autorotate
/etc/chopin/autorotate.conf
/etc/local.d/30-autorotate.start      # boot 时后台启动，无传感器则直接退出
```

### 29.5 v111

```
out-scp-v111-alpine/boot-stockdt.img   md5 32597681223a0a46e67534695cc93799
```

**镜像 52.4 MB / boot 分区 128 MB**，还剩约 75 MB 余量。
下一步往 rootfs 装合成器时要留意：`initramfs-root` 会整体进 cpio，
装完后要重新跑 `build-images.sh` 并确认 < 128 MB。

### 29.6 合成器侧的注意事项

sway 用 DRM 后端时**必须有 seat**：现代 wlroots 只支持 libseat，
所以要装并启动 `seatd`（或 elogind），然后
`LIBSEAT_BACKEND=seatd sway`。这一步在板子上没有 setup 脚本，需要手工验证。
`labwc` + `wlr-randr` 同样需要 seat。

## 30. 决定：合成器不装进 boot 镜像（2026-10-27 18:22）

### 30.1 结论

**`sway`/`cage`/`labwc` 都不装进 initramfs。** boot 分区 128 MB，装不下
（至少没有余量）。

依据（apk 在 rootfs 上的实测统计，`--simulate add`，app 总装机量口径）：

| 方案 | 装完总量 | 相对当前(79.2 MiB/68 包)的增量 |
| --- | --- | --- |
| 当前 rootfs | 79.2 MiB / 68 包 | — |
| `cage seatd wlr-randr` | 305.9 MiB / 108 包 | **+226.7 MiB** |
| `labwc seatd wlr-randr` | 332.3 MiB / 144 包 | +253.1 MiB |
| `sway seatd wlr-randr` | 333.3 MiB / 136 包 | +254.1 MiB |

体积主要来自 wlroots 的 GL 依赖链：
`mesa` → `mesa-dri-gallium` → `llvm22-libs` + `spirv-tools`（软件渲染 llvmpipe），
Alpine 的 mesa 没有按驱动拆分，panfrost 也在这个大包里。

当前：rootfs 96 MB（磁盘）→ `initramfs.cpio.gz` 35.6 MB → 镜像 52.4 MB。
再塞 +227 MB 未压缩，即使 gzip 后也要 +70 MB 上下，会顶到/超过 128 MB。
**没有余量，不装。**

### 30.2 后续怎么装（不需要重打 boot 镜像）

`chopin-autorotate` 已经设计成"一直等合成器出现"，所以：

1. 把合成器装到 **rootfs 分区**（不是 initramfs）：
   ```sh
   apk add sway seatd    # 或者 cage / labwc
   ```
2. 起 seat（sway 的 DRM 后端必须有 libseat）：
   ```sh
   seatd -g video &
   LIBSEAT_BACKEND=seatd sway &
   ```
3. `chopin-autorotate`（v111 起已在 boot 时自动后台运行）会检测到
   `swaymsg` 出现，随后就开始旋转。日志 `/tmp/chopin-autorotate.log`：
   ```
   output DSI-1 on swaymsg
   right-up -> transform 90
   ```
   **不用重新打包镜像。**

### 30.3 如果只想先看到"屏幕真的会转"

那就走方案 C：给内核内置的 KMS 工具（`init` 二进制里的 `setmode`，裸
`DRM_IOCTL_MODE_GETRESOURCES`，不依赖 libdrm）加一个 `rotate` 子命令，
直接设 primary plane 的 `rotation` 属性。零依赖、不占空间。
缺点：转的是 framebuffer console，文字会躺倒，只用于验证链路。

### 30.4 最终镜像

```
out-scp-v111-alpine/boot-stockdt.img   md5 32597681223a0a46e67534695cc93799
52.4 MB / 128 MB
```

## 31. 只测自动旋转：两条零成本路径（2026-10-27 18:45）

### 31.1 硬件事实：90/270 只能靠软件渲染

`mtk_ovl_supported_rotations()`（`mtk_disp_ovl.c:345`）只返回

```c
DRM_MODE_ROTATE_0 | DRM_MODE_ROTATE_180 | DRM_MODE_REFLECT_X | DRM_MODE_REFLECT_Y
```

**没有 90/270**。所以：
* 用 `modetest -w <plane>:rotation:` 走 KMS 只能转 180°，横屏无能为力；
* `CONFIG_FRAMEBUFFER_CONSOLE_ROTATION` 也没开（`# ... is not set`），
  而且 fbcon 旋转只能靠 boot 参数，静态；
* 竖↔横必须由合成器把画面自己画成旋转的 → 必须有软件/GPU 渲染。

### 31.2 任何合成器都要付 mesa 的账

| 组合 | 装完 | 增量 |
| --- | --- | --- |
| 基线 | 79.2 MiB / 68 包 | — |
| `mesa-egl mesa-gbm` | 297.6 MiB / 88 包 | +218.4 MiB |
| `wlroots0.20 cage` | 305.8 MiB / 106 包 | +226.6 MiB |
| `labwc` | 332.3 MiB / 144 包 | +253.1 MiB |
| `sway` | 333.3 MiB / 136 包 | +254.1 MiB |
| Xorg + xrandr | 319.1 MiB / 131 包 | +239.9 MiB |

连单独装 `mesa-egl` 都会拉 `mesa` 元包 → `mesa-dri-gallium` →
`llvm22-libs` + `spirv-tools`。**换合成器绕不过去。**

### 31.3 但「测自动旋转」不需要合成器

需要验证的是**决策链**：传感器 → 朝向 → 旋转指令。执行那一步是合成器的事。

#### 路径 1：`chopin-autorotate --dry-run`（自研守护进程，只决策）

```sh
chopin-autorotate --dry-run -v
```

输出（翻动手机时）：

```
normal     dev( +0.75  +9.11  +3.52)
normal     dev( +0.74  +9.04  +3.73)
right-up   dev( +9.24  +1.59  +3.39)
DRY-RUN would apply transform 90
left-up    dev( -9.48  +1.38  -0.20)
DRY-RUN would apply transform 270
normal     dev( +0.73  +9.35  -0.14)
DRY-RUN would apply transform normal
```

这完整覆盖了「姿态判定 → 去抖 → 旋转角度」，
差的只有最后一句 `swaymsg/wlr-randr` 调用。

#### 路径 2：标准工具 `iio-sensor-proxy` + `monitor-sensor`

它之前失败是因为**没有 udevd**（GUdev 枚举不到设备）。已装 `eudev`（+1.9 MiB）：

```
/sbin/udevd, /bin/udevadm
/etc/local.d/05-udev.start     # boot 时先起 udevd 并 udevadm trigger
```

顺手还带进来了 `iio-sensor-proxy-udev`（官方 udev 规则包）。

```sh
chopin-rotation        # dbus + iio-sensor-proxy + monitor-sensor
```

预期看到 `Accelerometer orientation changed: normal / left-up / ...`。

### 31.4 v112

```
out-scp-v112-alpine/boot-stockdt.img   md5 e5fe82b5f03f081b634b80fc499d758e
53.1 MB / 128 MB          （比 v111 只多 0.7 MB）
```

新增：`eudev`（udevd + udevadm）、`/etc/local.d/05-udev.start`、
`chopin-autorotate --dry-run`。

## 32. 自动旋转测试通过（2026-10-27 18:55）

### 32.1 测试 1：`chopin-autorotate --dry-run -v` —— **通过**

```
00:27:01 flat      dev( -0.47  -0.54  +9.90)
00:27:02 flat      dev( -0.72  -0.63 +10.12)
00:27:03 flat      dev( -3.35  -0.01  +7.26)
00:27:03 right-up  dev(+11.19  -0.79  +3.20)
00:27:03 right-up  dev( +9.11  +0.01  +4.37)
00:27:03 right-up  dev( +9.21  -0.07  +3.76)
00:27:03 DRY-RUN would apply transform 90
00:27:03 right-up -> transform 90
00:27:03 right-up  dev( +9.36  -0.20  +4.19)     ← 之后保持 90°，不重复触发
...
```

验证到的点：
* 平放（`|z| > 7`）→ `flat`，不动作；
* 从平放到横持的过渡帧（`-3.35, -0.01, +7.26`）没有误判；
* 连续 3 次一致才决策 → **去抖生效**；
* 决策 `transform 90` 与 `right-up` 对应（`MAP` 表生效）；
* 之后持续 `right-up` 不再重复下发 → 不会每帧都调合成器。

**这就是自动旋转的全部决策逻辑，已通过。**

### 32.2 测试 2：`iio-sensor-proxy` + `monitor-sensor`

udev 装上后，`iio-sensor-proxy` 能起来了：

```
    Waiting for iio-sensor-proxy to appear
+++ iio-sensor-proxy appeared
** (monitor-sensor:322): WARNING: Failed to claim accelerometer:
   GDBus.Error:org.freedesktop.DBus.Error.AccessDenied
```

原因：`ClaimAccelerometer` 走 polkit，而
`/usr/share/polkit-1/actions/net.hadess.SensorProxy.policy` 是

```xml
<allow_any>no</allow_any>
<allow_inactive>no</allow_inactive>
<allow_active>yes</allow_active>
```

这个 initramfs 没有 logind，调用者**没有任何 session**，三项都不匹配 → 拒绝。
而且 `polkitd` 根本没装，检查直接失败。

修法：装 `polkit`（+4.2 MiB），并手工补 `polkitd` 用户
（`--no-scripts` 不跑 post-install，和之前 `messagebus` 一样的问题）：

```
/etc/passwd: polkitd:x:102:102:Polkit Daemon Owner:/var/lib/polkit-1:/sbin/nologin
/etc/group : polkitd:x:102:
```

`chopin-rotation` 里在起 iio-sensor-proxy 之前先起
`/usr/lib/polkit-1/polkitd --no-debug`。polkit 对 **uid 0 有隐式授权**，
所以 root 的 claim 会通过。

### 32.3 v113

```
out-scp-v113-alpine/boot-stockdt.img   md5 7bf6b0605358524217232409cb31574b
53.6 MB / 128 MB
```

累计新增：`eudev`(+1.9 MiB)、`polkit`(+4.2 MiB)、`05-udev.start`、
`chopin-autorotate --dry-run`。**不装任何合成器/mesa。**

## 33. polkit 修好，标准接口 claim 成功（2026-10-27 19:05）

```
== starting dbus-daemon --system
== starting polkitd
== starting iio-sensor-proxy
== monitor-sensor (ctrl-c to stop; tilt the phone)
    Waiting for iio-sensor-proxy to appear
+++ iio-sensor-proxy appeared
```

**`AccessDenied` 消失**，`monitor-sensor` 成功 claim 到加速计。
§32.2 的判断成立：这个 initramfs 没有 session，只能靠 polkit 对 uid 0 的隐式授权。

### 33.1 顺带修掉的两个小问题

1. `chopin-rotation` 开头用裸 `cat` 读 raw，撞上首样本前的 `-EAGAIN`：
   ```
   cat: read error: Resource temporarily unavailable     × 3
   scale=0.001000000  x= y= z=
   ```
   改成调用 `chopin-orientation -i`（它内部有 40×50 ms 重试）。

2. 脚本里直接打印出用 D-Bus 查属性的方法，不依赖事件流。

### 33.2 最直接的验证：查 D-Bus 属性

`monitor-sensor` 只在朝向**变化**时才打印，静置时没有输出。
要立刻确认接口可用，直接查属性：

```sh
gdbus call --system --dest net.hadess.SensorProxy \
  --object-path /net/hadess/SensorProxy \
  --method org.freedesktop.DBus.Properties.Get \
  net.hadess.SensorProxy AccelerometerOrientation
```

竖持应返回 `(<'normal'>,)`；横过来再查一次应变成 `(<'left-up'>,)` /
`(<'right-up'>,)`。这同时验证了
「内核 → IIO → iio-sensor-proxy → D-Bus」整条链。

### 33.3 v114

```
out-scp-v114-alpine/boot-stockdt.img   md5 474baef76ad1bbd7dc19d76a61df186c
53.6 MB / 128 MB
```

## 34. "D-Bus 无响应"其实是 ^Z 停了进程（2026-10-27 19:20）

### 34.1 诊断结果

```
$ timeout 5 gdbus introspect ... net.hadess.SensorProxy
Error: Timeout was reached
$ cat /sys/bus/iio/devices/iio:device1/buffer/enable
0
$ ls .../trigger/
current_trigger
$ cat /proc/$(pidof iio-sensor-proxy)/wchan
do_signal_stop
```

**`wchan = do_signal_stop`** —— 进程不是阻塞在 I/O 或锁上，而是处于
**停止（T）状态**。原因：`iio-sensor-proxy` 是 `chopin-rotation` 用 `&`
起的，和脚本同属一个终端进程组，用户按下的 `^Z` 把整组都 SIGSTOP 了。

同时排除掉的猜测：

* `buffer/enable = 0` → 它**没有**走 IIO buffer 路径，§33 之后我怀疑的
  "buffer 读死锁"不成立；
* `trigger/` 里只有空的 `current_trigger`，内核也确实没有任何 trigger
  （`CONFIG_IIO_HRTIMER_TRIGGER` / `SYSFS_TRIGGER` / `CONFIGFS` 全未开），
  但这与本次现象无关；
* 驱动锁顺序没有问题，**不需要改内核**。

### 34.2 修法

`chopin-rotation` 里三个守护进程全部改用 `setsid` 启动：

```
setsid dbus-daemon --system --fork
setsid polkitd --no-debug &
setsid iio-sensor-proxy &
```

这样它们脱离终端的进程组，`^Z` / `^C` 只影响前台脚本，不会把服务一起停掉。

另外 `chopin-rotation -q` 可以只起服务、不跑 `monitor-sensor`，
彻底避免"用 Ctrl-Z 退出"这个习惯动作。

### 34.3 立即验证（v114 起就能跑，无需刷机）

```sh
pkill iio-sensor-proxy; pkill polkitd; pkill monitor-sensor; sleep 1
chopin-rotation -q
gdbus call --system --dest net.hadess.SensorProxy \
  --object-path /net/hadess/SensorProxy --method \
  org.freedesktop.DBus.Properties.Get \
  net.hadess.SensorProxy AccelerometerOrientation
```

### 34.4 v115

```
out-scp-v115-alpine/boot-stockdt.img   md5 bf7207070b365edf34e3c0d9dcd4e986
53.6 MB / 128 MB
```

## 35. 停止态进程不吃 SIGTERM，pkill 假成功（2026-10-27 19:30）

```
$ pkill iio-sensor-proxy; pkill polkitd; pkill monitor-sensor; sleep 1
$ chopin-rotation -q
== dbus already up
== iio-sensor-proxy already running        ← 明明 pkill 过
$ gdbus call ... AccelerometerOrientation
Error: Timeout was reached
```

原因：**停止（T）状态的进程不处理 SIGTERM**。SIGTERM 只是被挂起，
要等收到 SIGCONT 才送达；而 SIGKILL 对停止态进程立即生效。
所以之前被 `^Z` 停掉的那个实例既没死、也不响应 D-Bus，
`pidof` 又照样能看到它 → 脚本报 "already running"，继续用那个僵尸。

### 35.1 脚本加固

`chopin-rotation` 新增 `daemon_alive()`：

```sh
daemon_alive() {
	_pid=$(pidof "$1" 2>/dev/null) || return 1
	_state=$(sed -n 's/^State:[[:space:]]*\([A-Z]\).*/\1/p' /proc/"$_pid"/status)
	if [ "$_state" = "T" ]; then
		echo "== $1 is stopped (T state), killing it"
		kill -9 "$_pid"; sleep 1; return 1
	fi
	return 0
}
```

`polkitd` 和 `iio-sensor-proxy` 的存活判断都改用它：**检测到 T 状态就
`kill -9` 并重新拉起**。

### 35.2 一条命令完成自检

`chopin-rotation`（含 `-q`）现在会自己查一次 D-Bus 属性并打印结果：

```sh
chopin-rotation -q
...
== self-test: orientation right now
(<'normal'>,)
```

不用再手敲那条长 `gdbus call`。

### 35.3 立即验证（v114/v115 上也能用，不需要刷机）

```sh
killall -9 iio-sensor-proxy polkitd monitor-sensor; sleep 1
chopin-rotation -q
gdbus call --system --dest net.hadess.SensorProxy \
  --object-path /net/hadess/SensorProxy --method \
  org.freedesktop.DBus.Properties.Get \
  net.hadess.SensorProxy AccelerometerOrientation
```

### 35.4 v116

```
out-scp-v116-alpine/boot-stockdt.img   md5 8c2d3d87161bf043ace1e3712b9f5763
53.6 MB / 128 MB
```

## 36. `AccelerometerOrientation = 'undefined'` 是设计使然（2026-10-27 19:40）

```
$ chopin-rotation -q
== starting dbus-daemon --system
== starting polkitd
== starting iio-sensor-proxy
== self-test: orientation right now
(<'undefined'>,)
$ cat /proc/$(pidof iio-sensor-proxy)/wchan
do_sys_poll
```

三件事同时成立：

* D-Bus **接口活了**（不再 `Timeout was reached`）；
* `wchan = do_sys_poll` —— 进程健康地在 poll，不是停止态；
* 属性值是 `'undefined'`。

`'undefined'` 不是故障：iio-sensor-proxy **只在有人
`ClaimAccelerometer` 之后才去读传感器**，没人 claim 时属性保持
`'undefined'`。这是它的设计（省电）。

### 36.1 正确用法：claim → 采样 → 读 → release

```sh
gdbus call --system --dest net.hadess.SensorProxy \
  --object-path /net/hadess/SensorProxy \
  --method net.hadess.SensorProxy.ClaimAccelerometer
sleep 2
gdbus call --system --dest net.hadess.SensorProxy \
  --object-path /net/hadess/SensorProxy \
  --method org.freedesktop.DBus.Properties.Get \
  net.hadess.SensorProxy AccelerometerOrientation
gdbus call --system --dest net.hadess.SensorProxy \
  --object-path /net/hadess/SensorProxy \
  --method net.hadess.SensorProxy.ReleaseAccelerometer
```

`monitor-sensor` 内部做的就是这套（它 claim 之后才会打印
`Accelerometer orientation changed: ...`）。

### 36.2 自检内置

`chopin-rotation`（含 `-q`）的自检改成完整流程：

```
== self-test: claim, sample, read
   AccelerometerOrientation = (<'normal'>,)
```

### 36.3 v117

```
out-scp-v117-alpine/boot-stockdt.img   md5 27ed3f8c3cf4bf630b4c3fed8ae520e7
53.6 MB / 128 MB
```

## 37. iio-sensor-proxy 与我们的判定差 180°（2026-10-27 19:50）

### 37.1 现象

用户竖持手机：

```
$ gdbus call ... ClaimAccelerometer
$ gdbus call ... Properties.Get ... AccelerometerOrientation
(<'bottom-up'>,)
$ gdbus call ... ReleaseAccelerometer
()
```

* D-Bus **claim 成功**（polkit 生效），接口可用；
* 但竖持得到 `bottom-up`，而**物理上应该是 `normal`**；
* 同一时刻我们的 `chopin-orientation` 判 `normal`（与物理姿态一致）；
* 该时刻读数 `dev(+0.02, -0.22, +9.96)` 是平放，之后竖持 `y ≈ +9.4`。

⇒ **两个消费者约定相反，差 180°**。不是传感器错误，也不是驱动错误。

### 37.2 iio-sensor-proxy 没有可用的配置入口

二进制里只有：

```
in_accel_mount_matrix / in_mount_matrix / ACCEL_MOUNT_MATRIX
Failed to auto-detect mount matrix, falling back to identity
```

没有配置文件、没有 negate/flip 选项。**唯一可调的就是内核导出的
`in_accel_mount_matrix`**，而它同时被我们自己的工具消费。

### 37.3 先解耦，再调矩阵

`chopin-autorotate` 新增 `MATRIX` 配置项（`/etc/chopin/autorotate.conf`）：

| 值 | 含义 |
| --- | --- |
| `raw`（默认） | 用固件给的原始帧（即 display 帧），**不读 sysfs 矩阵** |
| `sysfs` | 套用内核导出的 `in_accel_mount_matrix` |

这样以后为了迁就 iio-sensor-proxy 而修改 DT 矩阵时，
**我们已验证正确的守护进程不会跟着翻**。

### 37.4 待定：到底是哪种翻转

只需一个额外数据点就能确定：

| 竖持 | 横持（右） | 结论 | 要设的矩阵 |
| --- | --- | --- | --- |
| `bottom-up` | `left-up` | x、y 都被取反 → 整体取反 | `-1,0,0 / 0,-1,0 / 0,0,-1` |
| `bottom-up` | `right-up` | 只有 y 被取反（反射） | `1,0,0 / 0,-1,0 / 0,0,1` |

判定方法：横持（屏幕朝自己、机身转 90°）再读一次属性。

### 37.5 v118

```
out-scp-v118-alpine/boot-stockdt.img   md5 c24f6b22fa3a6524610ef42d9ab6ccfa
53.6 MB / 128 MB
```

## 38. 定案：iio-sensor-proxy 差 180°，用 mount matrix 补（2026-10-27 20:05）

### 38.1 三个数据点

| 物理姿态 | 我们的判定 | iio-sensor-proxy | 差值 |
| --- | --- | --- | --- |
| 竖持 | `normal` | `bottom-up` | y 反号 |
| 横持右（`x=+10.51`） | `right-up` | `left-up` | x 反号 |
| 横持左（`x=-9.34`） | `left-up` | `right-up` | x 反号 |

决定性的是第一条：**正常竖持被报成"上下颠倒"，任何命名约定都解释不了**。
所以不是"命名差异"，是真的差 180°。

x、y 都反号、z 不参与朝向判定 ⇒ 矩阵应为**绕 z 轴 180°**：

```
-1  0  0
 0 -1  0
 0  0  1        （det = +1，是正规旋转）
```

按实测反推出的模型验算，四种姿态全部对齐：

```
姿态   我们的判定    iio 旧(单位阵)   iio 新(180°)
竖持   normal     bottom-up       normal      OK
横右   right-up   left-up         right-up    OK
横左   left-up    right-up        left-up     OK
倒置   bottom-up   normal          bottom-up   OK
```

### 38.2 改动

1. **DTS**：`mount-matrix` 从单位矩阵改为 `-1,0,0 / 0,-1,0 / 0,0,1`，
   注释里写明这是**专为 iio-sensor-proxy**，并附上三个实测数据点。
2. **`chopin-orientation`**：默认改用原始帧（物理正确），
   `-s` 才套 sysfs 矩阵（旧行为 `-i` 保留为同一含义）。
3. **`chopin-autorotate`**：`MATRIX=raw` 为默认（v118 起）。

⇒ 内核矩阵只为标准消费者服务；我们自己的工具**不跟着翻**。

### 38.3 v119

```
out-scp-v119-alpine/boot-stockdt.img   md5 4d5c7955717c375d662dff47f2237869
53.6 MB / 128 MB
```

### 38.4 验证

```sh
# 应该打印 sysfs 的 180° 矩阵，且方向仍是物理正确的（默认 raw 帧）
chopin-orientation -w
cat /sys/bus/iio/devices/iio:device1/in_accel_mount_matrix   # -1,0,0; 0,-1,0; 0,0,1

# 竖持应得到 normal，横持应得到 right-up / left-up
chopin-rotation -q
```

## 39. 真正的根因：内嵌 DTB 从不重建，DT 改动全部静默失效（2026-10-27 20:20）

### 39.1 启动链回顾（这次不能再忘）

```
preloader → LK → GenieZone → Linux
```

* **没有 U-Boot / UEFI**；
* boot.img 里**必须**放**厂商** device tree：LK 要把 `dtbo` 分区的覆盖层套在它上面，
  而覆盖层需要基树里的 60 个符号，给精简树就 `ufdt_apply_overlay() failed`，
  **LK 根本不启动内核**——这就是 `boot-owndtb.img` 永远开不了机的原因；
* 所以采用 MT6895 的办法：**把我们的 DTB 编进内核，在 `setup_arch()` 里替换掉
  LK 传进来的那棵**。

内嵌方式是：

```asm
arch/arm64/kernel/chopin-dtb.S:
        .incbin "arch/arm64/kernel/chopin-board.dtb"
```

而 `arch/arm64/kernel/chopin-board.dts` 只有一行 `#include` 主 DTS。

### 39.2 坑：Makefile 没有从 .dts 重建 .dtb 的规则

```make
targets += chopin-board.dtb
$(obj)/chopin-dtb.o: $(obj)/chopin-board.dtb
```

`targets +=` 只是让 `make clean` 认它；**没有任何规则把 `.dts` 编成 `.dtb`**。
所以那个 `.dtb` 一旦存在就永远不变，`.incbin` 一直嵌进去的是**陈旧的树**。

后果：**我改的 `mount-matrix`（先是单位矩阵、后是 180°）一次都没到过设备上。**
设备上一直读到的 `1,0,0; 0,1,0; 0,0,1` 不是我们的设置，而是
`iio_read_mount_matrix()` 在**属性缺失**时返回的默认单位矩阵。

证据（从 `Image` 里把 FDT 抽出来反编译）：

```
v118 内嵌: 281309 bytes  mount-matrix = "1","0","0","0","1","0","0","0","1"    ← 旧
v119 内嵌: 281309 bytes  mount-matrix = "1","0","0","0","1","0","0","0","1"    ← 也是旧！
新编 dtb:  281990 bytes  mount-matrix = "-1","0","0","0","-1","0","0","0","1"
```

### 39.3 修法：`port/tools/build-kernel.sh`

顺序做了四件事，缺一不可：

1. `make ... mediatek/mt6893-xiaomi-chopin.dtb`（从源码重建）
2. `cp` 到 `$out/arch/arm64/kernel/chopin-board.dtb`（`.incbin` 找的位置）
3. `rm -f $out/arch/arm64/kernel/chopin-dtb.o`（否则不会被重新汇编）
4. `make Image`，然后**把 FDT 从 Image 里抽出来反编译**，打印
   `mount-matrix` 做校验；缺失就告警

### 39.4 v120

```
out-scp-v120-alpine/boot-stockdt.img   md5 58c7c98117201500fa7f80656d74832b
53.6 MB / 128 MB
```

已确认 `boot-stockdt.img` 里 `Image.gz` 的内嵌 FDT 为
`-1,0,0 / 0,-1,0 / 0,0,1`。

**以后改任何 DT 属性都必须用 `port/tools/build-kernel.sh`，不要直接 `make Image`。**

### 39.5 之前结论的修正

* §27「mount-matrix 改成单位矩阵生效了」**不成立**——那只是默认值恰好等于单位矩阵；
* §38 的 180° 矩阵**从未上过设备**，所以 §38.1 那张「iio 旧/新对比」表里的"新"是推算，不是实测；
* 之前观察到 `0, 1, 0; -1, 0, 0; 0, 0, 1`（原始 DTS 值）说明**更早的某次构建**里
  `chopin-board.dtb` 被同步过一次，之后再没同步。

## 40. 全链路实测打通（2026-10-27 20:35）

```
$ cat /sys/bus/iio/devices/iio:device1/in_accel_mount_matrix
-1, 0, 0; 0, -1, 0; 0, 0, 1

$ chopin-rotation -q
== mount matrix
-1, 0, 0; 0, -1, 0; 0, 0, 1
== raw / scale
dev(  +2.24   +6.62   -1.71)  disp(  +2.24   +6.62   -1.71)  normal
== starting dbus-daemon --system
== starting polkitd
== starting iio-sensor-proxy
== self-test: claim, sample, read
   AccelerometerOrientation = (<'normal'>,)
```

三件事同时成立，**都是实测**：

1. DT 的 `mount-matrix` 第一次真正进入设备（§39 的构建 bug 修好后）；
2. 我们自己的工具用 raw 帧，判定与物理姿态一致（`normal`）；
3. **iio-sensor-proxy 的标准接口也给出 `normal`**——180° 偏差消除。

用户确认：「这次对了」。

### 40.1 完整验证清单（全部实测）

| 环节 | 证据 |
| --- | --- |
| SCP 固件出数 | `get dram phy addr=0x9f0cd000,size=1038048`；`rec-1/2/3` 逐条 44 字节，ts 间隔 10 ms |
| 内核 FIFO → IIO | `in_accel_{x,y,z}_raw` 稳定，`scale=0.001`，z≈9.95±0.03 |
| 姿态判定（我们的工具） | 六种姿态：normal / left-up / right-up / bottom-up / face-up / face-down |
| 旋转角度决策 | `--dry-run`：normal→normal、right-up→90、left-up→270，去抖 3 次、过渡帧判 unclear |
| 标准 D-Bus 接口 | claim 成功、`AccelerometerOrientation = (<'normal'>,)` |
| DT 交付链 | 内嵌 FDT 实测 = `-1,0,0 / 0,-1,0 / 0,0,1` |

### 40.2 关键教训汇总（本次会话踩到的）

| 教训 | 后果 | 现在的防线 |
| --- | --- | --- |
| `.incbin` 的 DTB 没有重建规则 | DT 改动静默失效两个构建周期 | `port/tools/build-kernel.sh` + 内嵌 FDT 自检 |
| 停止态进程不吃 SIGTERM | "pkill 了但还在"、D-Bus 假死 | `daemon_alive()` 检测 T 状态并 `kill -9` |
| `^Z` 会停掉同进程组的后台服务 | `wchan=do_signal_stop` | 守护进程全部 `setsid` |
| 先猜再查 | 连猜三轮（buffer 死锁、锁顺序），差点改对的内核 | 先看 `wchan` / `State:` / `/proc/device-tree` |
| `in_accel_*_raw` 首样本前返回 -EAGAIN | 看起来像驱动坏了 | 读取端统一重试 |
| iio-sensor-proxy 只在 claim 后采样 | 属性读成 `undefined` | 自检按 claim→sample→read→release |
| 厂商 mesa 元包 | 任何合成器都要 +218 MiB | 合成器不装进 initramfs，放 rootfs/U 盘 |

### 40.3 仍未解决（都与传感器无关）

1. **白屏**：`mi-dsi-panel` 的 `drm_panel_add` / `lcm_probe` refcount UAF，
   每 boot 3 条 `refcount_t` 警告，偶发白屏；
2. **SCP 固件偶发卡死**：某些 boot 停在 ~0.64 s，该次 boot 不出数，重启即可；
3. **屏幕实际旋转**：需要合成器（`chopin-autorotate` 已就绪，装上自动接管）。

## 41. 验收完成：两个独立消费者在四种姿态下完全一致（2026-10-27 20:45）

```
$ chopin-rotation -q
== raw / scale
dev(  +9.06   +2.10   +3.80)  disp(  +9.06   +2.10   +3.80)  right-up
   AccelerometerOrientation = (<'right-up'>,)

$ chopin-rotation -q
== raw / scale
dev(  -9.99   +1.27   +0.24)  disp(  -9.99   +1.27   +0.24)  left-up
   AccelerometerOrientation = (<'left-up'>,)
```

汇总（全部实测）：

| 物理姿态 | `chopin-orientation`（raw 帧） | `iio-sensor-proxy`（套 180° 矩阵） | 一致 |
| --- | --- | --- | --- |
| 竖持 | `normal` | `normal` | ✅ |
| 横持右（x=+9.06） | `right-up` | `right-up` | ✅ |
| 横持左（x=-9.99） | `left-up` | `left-up` | ✅ |
| 倒置 | `bottom-up` | （按模型推算，同一符号规则） | — |
| 平放 | `face-up` | 不参与朝向（`|z|` 大） | — |
| 面朝下 | `face-down` | 同上 | — |

**两条完全独立的判定路径给出相同结果**，这是最强的验证：
一条是我们自己按 raw 帧算的，另一条是 iio-sensor-proxy 按内核导出的
180° 矩阵算的。它们一致，说明：

1. 传感器的轴映射与符号是确定的、可复现的；
2. DT 的 `mount-matrix` 补偿量正确（恰好 180°）；
3. 标准 D-Bus 接口可供任何桌面环境消费。

### 41.1 加速度计移植：完成

从「能读数但不稳定、还会崩溃」到：

* SCP 固件稳定出数（100 Hz，FIFO 环形，10 ms 间隔）；
* 内核驱动零崩溃，重试有界，FIFO 几何校验；
* IIO 接口：raw / scale / samp_freq / mount_matrix 全部可用；
* 姿态判定与旋转决策经过去抖与防误触发；
* 标准 `net.hadess.SensorProxy` 接口与物理姿态一致；
* 自动旋转执行器（`chopin-autorotate`）就绪，装上合成器即可接管。

### 41.2 剩余事项（均与传感器无关）

| 项 | 状态 | 说明 |
| --- | --- | --- |
| 白屏 | 潜在，未复现 | `mi-dsi-panel` refcount UAF；复现时按交接文档抓 dmesg |
| SCP 偶发卡死 | 偶发 | 某些 boot 停在 ~0.64 s，重启即可 |
| 屏幕实际旋转 | 待部署 | 需合成器（mesa +218 MiB），放 rootfs 分区 / U 盘；`chopin-autorotate` 自动接管 |
