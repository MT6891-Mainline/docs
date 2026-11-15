# 加速度计（ICM40607 / SCP）交接文档 — 2026-10-26

> 目标：chopin（Redmi Note 10 Pro，MT6893）mainline Linux 下让加速度计出数据 →
> 自动旋转。本文件是当前进度的唯一权威交接，配合
> `port/AUTO-ROTATION-STATUS.md` §0.19–§0.30 与 `work/accint/reA..reJ.md` 阅读。

---

## 1. 一句话状态

**协议、传输、固件补丁注入全部打通，并且见过一次真实的样本写入通知
（`action=6`，指针每 10ms 递增 44 字节）；现在差两件事：让 enable 在每次
SCP 重启后可重复触发，以及定位样本真正落在哪个物理地址。**

进度：协议/传输 **~99%**，固件补丁工程 **~95%**，功能可用度 **~88%**，
自动旋转端到端 **~88%**。

---

## 2. 已经确定的事实（不要重复验证）

### 2.1 固件架构

- SCP 固件：`tinysys-scp-RV33_A`，位于 **`scp_a` 分区 = `/dev/block/259:8`**（6MB，
  偏移 0 起就是 `scp.img`），**LK 对它做 MediaTek RSA 签名校验**
  （cert2 里存 SHA-256(头)+SHA-256(载荷)，RSASSA-PSS；`reG.md`）。
  **任何字节改动都会被 LK 拒绝 → 卡第一屏重启**（已实测）。
- 固件只暴露 **CHRE/nanoHub**，**没有 MTK sensor_comm 通道**
  （legacy 表只有 pkt_id 1/2/3/4：chre_scp_rx / chre_x / chre_fw_ipi / elliptic）。
- mpool：mbox2、IPI 11（AP→SCP）；帧 `[u32 pkt_id][u32 len][payload]`。
- pkt_id 3 = `SCP_SENSOR_HUB_DATA`，payload = `{u8 sensorType; u8 action; u8 rsvd[2]; u32 data[11]}`，
  只接受 action **2/4/5/8/10**；`action=4` = SET_CONFIG（注册 DRAM FIFO），
  `action=6` = **NOTIFY（回包/样本通知）**。
- pkt_id 1 = nanoHub comms（`reason 0x1091`），payload =
  `{0x31, u32 seq, u32 reason, u8 len, ConfigCmd 20B, u32 crc32}`，
  ConfigCmd = `{u32 evtType=0x300; u64 latency; u32 rate(1024*Hz); u8 sensorType; u8 cmd; u16 flags}`；
  `cmd 0` = batch，`cmd 1` = enable；CRC 用固件自己的变体
  （非反射、半字节表、init 0xFFFFFFFF、无终值异或）——见 `work/accint/nanohub_pkt.py`。
- 固件日志在 boot 后 ~1.0s 就停止写入，**无法用来调试**（stock 也拿不到）。

### 2.2 运行时固件补丁（可行且已验证）

LK 签名不可绕，但 LK 会把镜像搬到 **DRAM `0x9fa00000`**（代码基址，
`DRAM = 0x9fa00000 + 代码地址`），我们的内核驱动本来就有这块的 ioremap。
于是：

```
/sys/kernel/debug/scp-chopin/patchscp   (写 1/2/3 选择补丁组)
   ↓ 改 DRAM 里的 1~6 条指令（先校验原值，幂等）
/sys/kernel/debug/scp-chopin/fullreset; sram; start
   ↓ loader 把 DRAM 重新拷进 TCM，不再验签
SCP 运行补丁后的固件
```

已实现的 6 个补丁（DRAM 偏移 = 代码地址）：

| 偏移 | 原 → 新 | 作用 |
| --- | --- | --- |
| `0x26EB2` | `93e56500` → `93e5e500` | `INT_SOURCE0 |= UI_DRDY_INT1` |
| `0x286DE` | `11c5` → `0100` | 强制 FIFO = STREAM |
| `0x23944` | `639e2505` → `13000000` | FSM 闸门 1 → nop |
| `0x23AE0` | `63905507` → `13000000` | FSM 闸门 2 → nop |
| `0x25FAA` | `29ce` → `f9a0` | `c.j 0x26078`：强制走 accel 上电块 |
| `0x0A968` | `0589` → `0545` | 传感器注册表恒 READY |

### 2.3 已见到的真实数据证据

某个 boot（补丁先打、SCP 重启、内核 enable 之后）dmesg 出现：

```
mpool rx: id 3 len 48 bytes 00 06 04 00 d=00000000 407e0100
mpool rx: id 3 len 48 bytes 00 06 04 00 d=00000000 6c7e0100
mpool rx: id 3 len 48 bytes 00 06 04 00 d=00000000 987e0100
```

`00 06` = action 6（NOTIFY），第二字 `0x00017e40 → 0x00017e6c → 0x00017e98`
**每 10ms 递增 0x2C = 44 字节** ⇒ 固件在按 44 字节单元写样本（≈100Hz）。
但 FIFO 头 `wp` 恒 0、`0x8F0CA700` 起 64KB 全 0 ⇒ **样本落点不在我们给的 base**
（疑似被固件 mapper `0x19c38` / 表 `0x749a8` 重映射）。

### 2.4 原厂（Android 15）实测参数

- 加速度计 HAL 名 `icm40607_acc`，12.5–500Hz，FIFO 4500/3000；
- 实际使能 **80ms = 12.5Hz**，batching 0；
- 开机 48s 才激活；enable 前有 `mtk_nanohub_config [1][4][2]…` 与每类 SET_CUST；
- host 顺序：`send_dram_info` → `SET_CONFIG` → `SET_CUST(GET_SENSOR_INFO)` →
  `SET_TIMESTAMP` → `batch(cmd0)` → `enable(cmd1)`（**只有一次 enable**）。

---

## 3. 下一步（按顺序，最多两处改动）

### 3.1 让 enable 可重复触发

文件 `drivers/misc/mediatek/sensor/2.0/sensorhub/ipi_comm.c`，
函数 `chsensor_enable_work()` 末尾加：

```c
	/* 每 30 秒重新使能一次：SCP 被 reset 后不会一直处于关闭状态 */
	schedule_delayed_work(&chsensor_enable_dw, msecs_to_jiffies(30000));
```

然后 `make Image` → `port/tools/build-images.sh`（用 alpine ramdisk，见 §4）。

### 3.2 定位样本落点

1. 用 `reC.md` 里 mapper（`0x19c38`，表 `0x749a8`，高 4 位窗口 + 低 28 位保留）
   反推 `bufferBase = 0x8F0CA700` 的真实映射地址；
2. 或者直接把 `bufferBase` 传成“映射后能落回 SENS 块”的值（例如 `0x0CA700` + 窗口 0x8）；
3. 一旦能在固定地址读到 44 字节单元，就写 **FIFO→IIO 派发**
   （解析 44 字节 → `chopin-accel` 的 `in_accel_*_raw`），再做 `iio-sensor-proxy`。

---

## 4. 调试手段（新会话直接用）

### 4.1 alpine 测试镜像（带 devmem，自动跑测试）

```
out-scp-v64-alpine/boot-stockdt.img   md5 0c156ecd052437626f2b0c22fcebbe24
```

- initramfs = `port/rootfs-alpine`（已加入 `/usr/bin/devmem` 静态二进制、
  `/init`（挂 debugfs/configfs + USB gadget ACM）、
  `/etc/local.d/10-chsensor.start`（开机自动：patchscp → fullreset/sram/start → 采样））。
- 构建：

```sh
cd port/rootfs-alpine && find . | cpio -o -H newc --owner=0:0 | gzip -9 > ../../work/alpine-initramfs.cpio.gz
cd ../.. && KERNEL=$PWD/build/chopin-7.3/arch/arm64/boot/Image \
  DTB=$PWD/build/chopin-7.3/arch/arm64/boot/dts/mediatek/mt6893-xiaomi-chopin.dtb \
  STOCK_DTB=$PWD/port/out/stock-base.dtb RAMDISK=$PWD/work/alpine-initramfs.cpio.gz \
  port/tools/build-images.sh out-scp-vNN-alpine
```

**注意**：`build-images.sh` 的默认 CMDLINE 含 `bootopt=64S3,32N2,64N2`，
**不要覆盖 CMDLINE**，否则 LK 阶段直接重启（已踩过）。

### 4.2 串口

- USB gadget 控制台出现在 **`/dev/ttyACM0`（115200）**（alpine 的 `/init` 里建了 ACM gadget，
  UDC = `11201000.usb`）；
- 屏幕（`console=tty0`）也有全部内核日志。

### 4.3 关键观测点

```sh
# 数据通知（action 6）
dmesg | grep "id 3 len 48" | grep "00 06" | tail -5
# FIFO 头
devmem 0x8F0CA700; devmem 0x8F0CA704; devmem 0x8F0CA708
# 扫内存找样本（alpine 的 ash 不支持 0x，要用 16#）
a=$((16#8F0CA000)); end=$((16#8F0DA000))
while [ $a -lt $end ]; do v=$(devmem $a); [ "$v" != "0x00000000" ] && echo "$a $v"; a=$((a+4)); done | head -20
```

### 4.4 主机 ⇄ 设备 SSH（已打通）

`work/accint/sshcfg`（主机 `192.168.0.115` ⇄ 设备 `192.168.0.109`，公钥认证）。
设备跑 Linux 时可直接：

```sh
ssh -F work/accint/sshcfg root@192.168.0.109
adb connect 192.168.0.109:5555     # Android 侧（adb 在主机 adb server 上可用）
```

---

## 5. 资产清单

| 用途 | 路径 |
| --- | --- |
| alpine 测试镜像（devmem + 自动测试 + ttyACM0 shell） | `out-scp-v64-alpine/boot-stockdt.img` |
| Debian rootfs 内核（日常可用） | `out-scp-v60/boot-stockdt.img` |
| 最全固件补丁内核 | `out-scp-v58/boot-stockdt.img`（6 补丁，patchscp 2） |
| rootfs 备份（6.64GiB） | `work/backup/rootfs_sdc75.img`（sha256 见同目录） |
| 固件逆向报告 | `work/accint/reA..reJ.md` |
| 固件工具 | `work/accint/rv.py` `walk.py` `xr.py` `xrf2.py` `dmp.py` `drv.py` `nanohub_pkt.py` `scp4.asm` |
| 原厂抓取 | `work/accint/stock_sensorservice.txt` `dmesg_nanohub2.txt` `stock_props.txt` |
| 文档 | `port/AUTO-ROTATION-STATUS.md` §0.19–§0.30 |

---

## 6. 坑（务必看）

1. **不要覆盖 `build-images.sh` 的默认 CMDLINE**（`bootopt=64S3,32N2,64N2` 必需），
   否则 LK 阶段重启、无内核日志。
2. **不要改 `scp_a` 分区**（RSA 签名，改了必卡启动）；补丁只能走 `patchscp` 的 DRAM 注入。
3. **SCP 重启后 enable 不会自动重跑**（当前代码一次开机只跑一次）——见 §3.1。
4. `patchscp` 是幂等的：第二次执行会打印 `expected … got …` 并跳过，不是错误。
5. `mtk_ipi_send(timeout=0)` 的 `IPI 11 timeout` **不代表 SCP 卡死**（只做一次 trysend）。
6. FIFO 头里的 `wp` 恒 0 **不代表没数据**：固件用内部指针（见 §2.3），
   真实落点需按 §3.2 查 mapper。
7. alpine 的 `sh`（ash）不支持 `0x` 字面量，算术里要用 `16#`。
8. 设备当前是 **Android**（stock，boot_a 已刷回）；要回 mainline 就刷
   `out-scp-v60/boot-stockdt.img`；要测加速度就刷 `out-scp-v64-alpine/boot-stockdt.img`。
