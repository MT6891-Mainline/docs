# 加速度计（ICM40607）交接文档 — 2026-10-27

> 上一份：`port/HANDOFF-ACCEL-2026-10-26.md`（协议/固件逆向部分仍然有效）
> 完整证据链：`port/AUTO-ROTATION-STATUS.md` §0.19–§0.31
> 固件逆向报告：`work/accint/reA.md` … `reK.md`

---

## 1. 一句话现状

**加速度计已经稳定工作了。** 2026-10-27 用 `out-scp-v106-alpine` 实测：

```
$ i=0; while [ $i -lt 10 ]; do ... in_accel_{x,y,z}_raw ...; sleep 1; done
497 -167 9959     481 -227 9964
488 -177 9921     483 -237 9967
514 -220 9957     481 -241 9947
483 -237 9957     471 -222 9964
481 -227 9957     486 -227 9952
478 -237 9957
```

`scale = 0.001` ⇒ x≈0.48、y≈−0.22、z≈**9.95 m/s²**，重力在 z、量级正确、
抖动 ±0.03 m/s²、方向不翻转。固件侧证据：

```
[1.699](0) get dram phy addr=0x9f0cd000,size=1038048, maxEventNumber:23592
[1.699](0) get dram phy rp=0,wp=0
[30.164]  rec-1 off=98472 ts=24441841290 x=490 y=-241 z=9969
[30.165]  rec-2 off=98428 ts=24431841288 x=478 y=-241 z=9964
```

`ts` 间隔 10,000,002 ns = **10 ms = 100 Hz**；`off` 每次递减 44 = 记录连续。

### 决定性的一处修复：SET_CONFIG 必须每轮重发

固件的 mpool legacy handler 是在它自己 boot 之后才注册的。
在注册前发过去的包会被丢弃并打印
`legacy_handler id 11,pkt_id:N handler is null!!!`，
于是 `chre_dram` / `maxEventNumber` 不会设置，drain 直接打印
`dram info not got, exit write dram` 并且一个字节都不写 —— `wp` 永远 0。

所以 **SET_CONFIG 必须每一轮 pass 都重发**（历史上"3~6 次才出数"就是这个机制，
不是概率）。一旦 handler 就绪，随便哪一轮都能落地。
反过来，FIFO 头**只能写一次**：在数据流动时清零 `wp` 正是当年"能读数但方向乱跳"
的原因之一。

### 标准接口（iio-sensor-proxy）也已打通

```
$ cat /sys/bus/iio/devices/iio:device1/in_accel_mount_matrix
-1, 0, 0; 0, -1, 0; 0, 0, 1
$ chopin-rotation -q
   AccelerometerOrientation = (<'normal'>,)
```

四种姿态下两个**独立**判定路径结果完全一致（2026-10-27 实测）：

| 物理姿态 | `chopin-orientation`（raw 帧） | `iio-sensor-proxy`（套 180° 矩阵） |
| --- | --- | --- |
| 竖持 | `normal` | `normal` |
| 横持右 | `right-up` | `right-up` |
| 横持左 | `left-up` | `left-up` |
| 倒置 | `bottom-up` | （同一符号规则） |

DT 的 `mount-matrix` 是**绕 z 轴 180°**，只为了迁就 iio-sensor-proxy 的
符号约定（它和我们差 180°，实测：竖持它报 `bottom-up`、横持 x 反号）。
我们自己的 `chopin-autorotate` / `chopin-orientation` 用 raw 帧
（`MATRIX=raw`），不受这个矩阵影响。

**改任何 DT 属性必须用 `port/tools/build-kernel.sh`。**
内核不是用 boot.img 里的 DTB 启动的：LK 只接受厂商树（它要套 dtbo 覆盖层），
我们的树是 `.incbin` 进内核、在 `setup_arch()` 里换掉的。
而 `chopin-board.dtb` 在 Makefile 里**没有从 .dts 重建的规则**，
直接 `make Image` 会让 DT 改动静默失效——这个坑已经踩过两次。

### 仍未解决（与传感器无关）

* **白屏（潜在，未复现）**：`mi-dsi-panel` 的 `drm_panel_add` / `lcm_probe`
  refcount UAF——每 boot 固定 3 条 `refcount_t` 警告，refcount 被 saturate
  （永久泄漏）。2026-10-27 晚连续多次重启**均未再复现**白屏，所以降级为潜在问题，
  不阻塞任何工作。
  复现时抓这几条即可定位：
  ```sh
  dmesg | grep -E "refcount_t|drm_panel_add|lcm_probe|mtk_dsi_probe" | head -20
  dmesg | grep -E "mediatek-drm|fb0|mi_disp" | tail -20
  cat /sys/class/graphics/fb0/modes
  ```
  根因方向：`lcm_probe` 里 `drm_panel_get()` 作用在尚未 `drm_panel_add()`
  的 panel 上（refcount 从 0 加），说明某条 probe 路径上 `drm_panel_init()`
  没走完，是个竞态；修法是保证 `drm_panel_get()` 之前 panel 必已 add，
  或者干脆去掉这次多余的 get。与传感器链路完全无关。
* 固件自身偶发卡死（无任何 IPI 也会停在 ~0.64 s，导致个别 boot 不出数）。

---

## 2. 可用的镜像

| 镜像 | md5 | 状态 |
| --- | --- | --- |
| `out-scp-v120-alpine/boot-stockdt.img` | `58c7c98117201500fa7f80656d74832b` | **最新（重要）**：修复内嵌 DTB 从不重建的构建 bug，`-1,0,0/0,-1,0/0,0,1` 首次真正进入内核。**改 DT 必须用 `port/tools/build-kernel.sh`** |
| `out-scp-v119-alpine/boot-stockdt.img` | `4d5c7955717c375d662dff47f2237869` | ：DT mount-matrix 改为绕 z 180°（`-1,0,0/0,-1,0/0,0,1`），修 iio-sensor-proxy 的 180° 偏差；我们的工具改用 raw 帧。53.6 MB |
| `out-scp-v118-alpine/boot-stockdt.img` | `c24f6b22fa3a6524610ef42d9ab6ccfa` | ：`chopin-autorotate` 的 `MATRIX=raw|sysfs` 解耦（iio-sensor-proxy 与我们的判定差 180°）。53.6 MB |
| `out-scp-v117-alpine/boot-stockdt.img` | `27ed3f8c3cf4bf630b4c3fed8ae520e7` | ：自检改为 claim→sample→read→release（`undefined` 是没人 claim 时的正常值）。53.6 MB |
| `out-scp-v116-alpine/boot-stockdt.img` | `8c2d3d87161bf043ace1e3712b9f5763` | ：`chopin-rotation` 自动清理停止态守护进程（T 状态不吃 SIGTERM）+ 一条命令自检。53.6 MB |
| `out-scp-v115-alpine/boot-stockdt.img` | `bf7207070b365edf34e3c0d9dcd4e986` | ：守护进程改用 `setsid`，`^Z` 不再把它们停掉（`wchan=do_signal_stop` 那个坑）。53.6 MB |
| `out-scp-v114-alpine/boot-stockdt.img` | `474baef76ad1bbd7dc19d76a61df186c` | ：修 `chopin-rotation` 的 EAGAIN 显示；`monitor-sensor` 已能 claim。53.6 MB |
| `out-scp-v113-alpine/boot-stockdt.img` | `7bf6b0605358524217232409cb31574b` | +`polkit`（修 `monitor-sensor` 的 AccessDenied）。53.6 MB |
| `out-scp-v112-alpine/boot-stockdt.img` | `e5fe82b5f03f081b634b80fc499d758e` | ：+`eudev`（让 `iio-sensor-proxy`/`monitor-sensor` 可用）+ `chopin-autorotate --dry-run`。53.1 MB |
| `out-scp-v111-alpine/boot-stockdt.img` | `32597681223a0a46e67534695cc93799` | ：传感器 + `chopin-autorotate` 自动旋转守护进程。52.4 MB，**不装合成器**（+227 MB 装不进 128 MB）。合成器装到 rootfs 分区即可，无需重打镜像 |
：+`chopin-autorotate` 自动旋转守护进程（swaymsg / wlr-randr 后端）；52.4 MB |
| `out-scp-v110-alpine/boot-stockdt.img` | `6ae21c395566f3e91e09aa5c62f5bfca` | `mount-matrix` 改为单位矩阵（固件已自行校正，内核再套会错 90°）；含 `chopin-rotation`/`chopin-orientation` |
| `out-scp-v109-alpine/boot-stockdt.img` | `64abfceebaa693416336684f510abe89` | ：修 `in_accel_raw` 的 -EAGAIN 重试 + rootfs 补 `messagebus` 用户（dbus 可起）|
| `out-scp-v108-alpine/boot-stockdt.img` | `3ecaaced7fa76b1de1654596b1f9cc54` | +`chopin-rotation` / `chopin-orientation`（后者 `-i` 可看固件原始帧）|
| `out-scp-v106-alpine/boot-stockdt.img` | `0dbf2b88223a55ba421e29220855d41b` | SET_CONFIG 恢复「每轮重发」（这就是 3~6 次才出数的机制）；FIFO 头只写一次；`echo 2` |
| `out-scp-v105-alpine/boot-stockdt.img` | `d5530e9ad71195bc6154c533f8459dd2` | drain 重定向补丁补齐 4 个读点（v104 漏了 3 个，NULL 解引用把 SCP 打死）；group 重新分层，钩子用 `echo 3` |
| `out-scp-v104-alpine/boot-stockdt.img` | `1a7583aae987e35917158efc3002eb1f` | **坏的**：只补了 2 个读点，`0x3ABDE` NULL 解引用 → SCP 起不来 |
| `out-scp-v103-alpine/boot-stockdt.img` | `6134299fc2b7d7e6b2a56f824f2b7b8f` | `chopin-scplog -r N`（`-r -1` = 最新一段）+ boot 钩子自动 dump 最新 run |
| `out-scp-v102-alpine/boot-stockdt.img` | `186d2f038c4179663b1bbab76c5b8e22` | **关键实验版**：`start` 后 0.4 s 早发 enable；实测固件在 `retry 4` 后开始 106 行/秒 活动 |
：boot 时在 `start` 后 0.4 s 触发一次 `retry`，把 100 Hz 递进固件还没放弃的窗口；boot 日志自动记录完整 accel 链条 |
| `out-scp-v101-alpine/boot-stockdt.img` | `dd77916ee0757cc564f173cc3ff4a930` | boot 日志自动附固件日志；hook 用绝对路径调 scplog |
| `out-zeroipi2-alpine/boot-stockdt.img` | `5ac91c56e6715b5265bc5bc2330c51ae` | **最新对照**：一个控制包都不发（`/etc/chopin/zero-ipi` 标记）|
| `out-scp-v100-alpine/boot-stockdt.img` | `43a92bc8d599ba2f61dbb69b3010f4d5` | 同 v101，但 hook 用裸 `chopin-scplog`，可能因 PATH 静默失败 |
| `out-zeroipi-alpine/boot-stockdt.img` | `a2be7e9cdae047c8b355f4cb6f3ed146` | 同上（配对 v100）|
| `out-scp-v99-alpine/boot-stockdt.img` | `453091e618d44b9e373a926e1a19b21e` | `/usr/bin/chopin-pktid-sweep`，按固件日志 `legacy_handler`/`hostintf` 判读。**关键结论：MTK legacy 表为空，SET_CONFIG 不可达；CHRE 路径已认出 100 Hz** |
| `out-scp-v98-alpine/boot-stockdt.img` | `f6b940faeed30d7618e1d804d8c29385` | rootfs 加 `/usr/bin/chopin-accel-test`（零 IPI 重启 SCP，判定「包是否打死固件」）|
| `out-scp-v97-alpine/boot-stockdt.img` | `42bcc8d3a2b848058695f34a03f36158` | `chsensor_pkt_id` 运行时旋钮（mpool 包 id 0..4 可扫）；配套 `port/tools/accel-pktid-sweep.sh` |
| `out-scp-v96-alpine/boot-stockdt.img` | `dff6a6d03c020650e6dde07b32240f29` | `chopin-scplog` 改用 mmap（v95 的 pread 版必然 EFAULT）+ SIGPIPE 处理 |
| `out-scp-v95-alpine/boot-stockdt.img` | `5e5601bd4628ff81af91f122228c9129` | **坏的**：scplog 用 pread，`/dev/mem` 对 System RAM 返回 EFAULT |
| `out-scp-v94-alpine/boot-stockdt.img` | `27ea8dbf8efe50a1cf14197f6c3845f2` | `log` 节点改为取偏移 0 起的连续段（v93 取块尾，拿到的是上一轮残留）|
| `out-scp-v93-alpine/boot-stockdt.img` | `d7fa2dc6ec9caef8bc92843203ced90b` | 新增 `cat /sys/kernel/debug/scp-chopin/log`（但取的是块尾 = 上一轮残留，见 v95）|
| `out-scp-v92-alpine/boot-stockdt.img` | `a30bab0d6fe08fff90052b28ccc4da20` | SCP 重启后自动重发 SET_CONFIG（`scp_A_register_notify`）；新增 `chsensor_zero_ipi` 开关（`/sys/module/sensorhub/parameters/`）|
| `out-scp-v91-alpine/boot-stockdt.img` | `0ea1cf699d4acbcf2f3ceb14155347e8` | v90 + initramfs 自启 SCP（`init.c` 跑 `/etc/local.d/*.start`）；但内核还没有「SCP 重启后重发 SET_CONFIG」的逻辑 |
| `out-scp-v90-alpine/boot-stockdt.img` | `faeb1de603284c1ac0b0dce7719ca155` | 崩溃已修（`GET_DATA` 不再刷屏），SET_CONFIG 挪到 +5 s，FIFO 头 44 对齐自愈；但 **SCP 没启动**（ramdisk 不跑 local.d），故无任何 chsensor 输出 |
| `out-scp-v89-alpine/boot-stockdt.img` | `478688835cec77a08d5270c2546cc0a6` | 自动重试 6 次（**会崩**：`GET_DATA` 的 work 被反复 re-init） |
| `out-scp-v88-alpine/boot-stockdt.img` | `47fe0f0e5b34c78fbe9bdecde9d49e61` | 含手动 `retry` 节点 |
| `out-scp-v87-alpine/boot-stockdt.img` | `8a223fd207b3a5ba7c7e4274b8974caa` | 一次性序列（最保守、最稳定） |
| `out-scp-v86-alpine/boot-stockdt.img` | `ca819fb22be12ea0f5b0578ddc9f030c` | 移除 MTK connectivity 整栈 |
| `out-scp-v80-alpine/boot-stockdt.img` | `e0abefd2c6fd090fe5bea077b62df67a` | 首次确认数据流动 |
| `out-scp-v60/boot-stockdt.img` | — | 日常系统内核（不含传感器补丁） |

刷写：`fastboot flash boot_a out-scp-vNN-alpine/boot-stockdt.img && fastboot reboot`
串口：`ttyACM0` @115200（alpine 会把 shell 放到 `/dev/ttyGS0`）

---

## 3. 已经验证可用的链路（不要重做）

```
SCP 固件（7 个 DRAM 补丁，见 §4）
  → SET_CONFIG 注册 DRAM FIFO @ 0x8F0CA700（FIFOSize = ((0x100000-16)/44)*44 = 0xFFFC8）
  → SET_CUST(12) + SET_TIMESTAMP + batch/enable（nanoHub，pkt_id 1，reason 0x1091）
  → 固件每写一批样本发一条 action=6 通知（**已被内核丢弃**，见 §5）
  → 内核每 100ms 轮询 FIFO：header{rp,wp,FIFOSize,resv} + 44 字节 data_unit_t
  → sensorhub_push_sample() → IIO `chopin-accel`
  → /sys/bus/iio/devices/iio:device1/in_accel_{x,y,z}_raw   （scale 0.001）
```

### 记录布局（实测）

| 偏移 | 内容 |
| --- | --- |
| +0 | 类型字节 —— **本固件读出来是 0**（u32 为 0x00010000），**不要用它过滤** |
| +4 | u64 时间戳 |
| +12 | s32 x（mm/s²） |
| +16 | s32 y |
| +20 | s32 z（重力 ≈ 9940） |

读位置：`0x8F0CA700 + 0x10 + wp`（`wp` 是**字节**偏移，位于 header+4）。
`0x8F0CA708` = FIFOSize，必须是 44 的倍数。

---

## 4. 必须打上的 7 个固件补丁（`patchscp`）

```
0x26EB2  93e56500 → 93e5e500    INT_SOURCE0 |= UI_DRDY_INT1
0x286DE  11c5     → 0100        force FIFO STREAM
0x23944  639e2505 → 13000000    FSM gate 1 (nop)
0x23AE0  63905507 → 13000000    FSM gate 2 (nop)
0x25FAA  29ce     → f9a0        c.j 0x26078
0x0A968  0589     → 0545        registry always READY
0x26020  9375050f → 93053000    boot 时强制 PWR_MGMT0=0x03
```

用法：
```sh
echo 2 > /sys/kernel/debug/scp-chopin/patchscp     # 打补丁（幂等，重复执行会打印 expected/got）
echo 1 > /sys/kernel/debug/scp-chopin/fullreset    # 清 TCM
echo 1 > /sys/kernel/debug/scp-chopin/sram
echo 1 > /sys/kernel/debug/scp-chopin/start
```

**绝对不能**直接改 `scp_a` 分区：LK 会做 MediaTek RSA 验签，任何修改都会在
boot 阶段重启（`mt_scp.c:127`）。只能改 DRAM 里的运行副本（`0x9fa00000` +
代码偏移），然后 `fullreset → sram → start` 让 loader 重新搬运（不再验签）。

---

## 5. 已经踩过并修掉的坑

| # | 现象 | 根因 | 修法 | 提交 |
| --- | --- | --- | --- | --- |
| 1 | `in_accel_*_raw` 恒 `-EAGAIN` | 记录第 0 字节不是 sensor_type，`if (type == 1)` 把样本全丢 | 无条件按 accel 推送 | `49028425d` |
| 2 | `buffer/enable` 报 `EINVAL` | `postenable` 去启动已失效的 MTK 控制通道，失败即拒绝 | 控制失败只告警、继续 | `d9cce1d52` |
| 3 | 扫描全 0、找不到样本 | 记录在 `base+0x10+wp`（`wp` 是字节偏移），不是固定窗口 | 按 `wp` 读；`fifo` debugfs 节点自动解码 | — |
| 4 | 系统卡死/串口无响应 | 每 20ms `ioremap(1MB)`+`iounmap` + 每包 `pr_info` 刷屏 | 缓存映射、降频、日志降级 | `3942973fa` `ba2d7361b` |
| 5 | `scale` 是 1.0 | 固件给的是 mm/s² | accel 固定 `gain=1000` → `scale=0.001` | `a0aa71937` |
| 6 | 内核 oops（`PC is at:` + `die`） | SCP 复位期间仍读它的 DRAM 窗口 | `is_scp_ready(SCP_A_ID)` 检查 | `ca7cbe8a8` |
| 7 | 通知洪水（100 条/秒）拖死系统 | 每条 `action=6` 都走完整控制路径 | mpool 入口直接丢弃 `b[1]==6` | `bafe9d24c` |
| 8 | WiFi/conninfra 刷屏 481 次重试 | 只关了 `MTK_COMBO_WIFI`，WiFi 实际挂在 `MTK_COMBO` 下 | **关掉 `CONFIG_MTK_COMBO` 整栈** | `72a4b7fbc` |
| 9 | `IPI 11 timeout` 永远在涨 | 周期重发整套序列 | 一次性/有界重试，`wp` 一动就停 | `5cf93d0d4` `59012e451` |

---

## 6. 尚未解决的问题（下一轮的主战场）

### 6.1 固件只在部分 boot 采样

实测规律（**重要**）：

```
try1 EAGAIN → try2 EAGAIN → try3 9964 → try4 9979 → try5 9959
```

* 需要 **3~6 次**启动序列才开始采样，一旦开始就持续出数；
* 判定“已开始”的唯一可靠信号是 **`wp` 从 0 变为非 0**；
* 每轮序列约产生 **15 条 `IPI 11 timeout`**，重试 5 次以上 → 崩溃。

手动重试入口（v88+）：
```sh
echo 1 > /sys/kernel/debug/chsensor/retry
```

### 6.2 核心矛盾

* **少重试** → 固件不启动（大多数 boot）；
* **多重试** → IPI 风暴 → 内核崩溃。

`IPI 11 timeout` 每次都在，说明 **SCP 对我们的控制包根本不回 ack**，
“命中采样”是概率事件，不是确定性行为。

### 6.3 两个候选方向

| 方案 | 做法 | 评价 |
| --- | --- | --- |
| **A. 零 IPI** | 去掉所有 `mpool_send_pkt`，只 patchscp + start + 轮询 FIFO | 成本小。若固件能自己采样就完美（历史上确有未重试也出数的 boot）；否则彻底没数 |
| **B. 攻 ack** | 查固件为什么对 pkt_id 3 / CHRE 帧不回 ack（`work/accint/reB.md` §CHRE 解析、`0x3b3ee`、`0xbe1cc`） | 成本大，但这是根治方向 |

### 6.4 另一个潜在崩溃源（与传感器无关）

```
refcount_t: addition on 0; use-after-free.
  drm_panel_add+0x1c/0x60
  lcm_probe+0x240/0x404        ← mi-dsi-panel / lcm_probe
```

显示面板驱动的引用计数 UAF，会在不定时刻崩内核。**如果去掉传感器重试后仍崩，
请优先怀疑这里**（临时禁用该面板驱动即可验证）。

---

## 7. 调试入口速查

```sh
# 固件补丁 / 重启 / 采样重试
echo 1 > /sys/kernel/debug/scp-chopin/fifo        # 打印 FIFO 头 + 最新 3 条记录（已解码 x/y/z）
echo 2 > /sys/kernel/debug/scp-chopin/patchscp
echo 1 > /sys/kernel/debug/scp-chopin/fullreset
echo 1 > /sys/kernel/debug/scp-chopin/sram
echo 1 > /sys/kernel/debug/scp-chopin/start
echo 1 > /sys/kernel/debug/chsensor/retry         # v88+

# SCP 状态 / 数据
cat /sys/class/misc/scp/scp_A_status
cat /sys/bus/iio/devices/iio:device1/in_accel_{x,y,z}_raw
cat /sys/bus/iio/devices/iio:device1/in_accel_scale      # 0.001
D=/sys/bus/iio/devices/iio:device1
for i in 1 2 3; do
  echo "$(cat $D/in_accel_x_raw) $(cat $D/in_accel_y_raw) $(cat $D/in_accel_z_raw)"
  sleep 1
done

# 健康检查
dmesg | grep -c "IPI 11 timeout"                  # 不该持续增长
dmesg | grep -c "WIFI failed\|conninfra"          # 应为 0
dmesg | grep -c "Oops\|Unable to handle"          # 应为 0
```

**注意**：alpine 的 `ash` **不支持** `{x,y,z}` 花括号展开，也不认 `0x` 字面量
（用 `$((16#...))`）。

---

## 8. 代码位置

| 文件 | 作用 |
| --- | --- |
| `drivers/misc/mediatek/scp-chopin.c` | 固件补丁、`fullreset/sram/start`、`fifo` 解码节点 |
| `drivers/misc/mediatek/sensor/2.0/sensorhub/ipi_comm.c` | IPI/mpool、埋点序列、FIFO 轮询、`push_sample`、`retry` 节点 |
| `drivers/misc/mediatek/sensor/2.0/sensorhub/sensorhub_main.c` | `sensorhub_push_sample()` |
| `drivers/misc/mediatek/sensor/2.0/sensorhub/sensorhub_iio.c` | `chopin-accel` IIO 设备、scale、buffer |
| `port/rootfs-alpine/` | 测试 rootfs（`init`、`devmem`、`etc/local.d/10-chsensor.start`） |
| `port/tools/build-images.sh` | 打包（**绝不要覆盖 CMDLINE**：默认含 `bootopt=64S3,32N2,64N2`） |

构建：
```sh
make -C git/chopin-linux-7.3 O=$PWD/build/chopin-7.3 ARCH=arm64 \
     CROSS_COMPILE=aarch64-linux-gnu- -j16 Image
cd port/rootfs-alpine && find . | cpio -o -H newc --owner=0:0 | gzip -9 \
     > ../../work/alpine-initramfs.cpio.gz
KERNEL=$PWD/build/chopin-7.3/arch/arm64/boot/Image \
DTB=$PWD/build/chopin-7.3/arch/arm64/boot/dts/mediatek/mt6893-xiaomi-chopin.dtb \
STOCK_DTB=$PWD/port/out/stock-base.dtb \
RAMDISK=$PWD/work/alpine-initramfs.cpio.gz \
port/tools/build-images.sh out-scp-vNN-alpine
```

---

## 9. 下一步建议（按优先级）

1. **先验证方案 A（零 IPI）**：把 `chsensor_enable_work` 里的 `mpool_send_pkt`
   全部去掉，只保留 patchscp + `start` + FIFO 轮询，跑 3 个 boot 看是否出数。
   如果出数 → 直接删掉整套 IPI 控制逻辑，问题彻底消失。
2. 若 A 不行 → 走 B，查 ack（`work/accint/reB.md` 的 CHRE 解析路径）。
3. 稳定性确认后 → 打包 `iio-sensor-proxy`（alpine rootfs 里已装 3.9-r2）：
   ```sh
   dbus-daemon --system --fork
   /usr/libexec/iio-sensor-proxy &
   monitor-sensor
   ```
4. 自动亮度（BU27030，AP 侧 I2C，与 SCP 无关）尚未开始，可独立并行。
