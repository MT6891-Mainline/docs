# 指纹移植：GF3626ZS2 在主线上的第一步

> 2026-11-08。本文记录指纹这条线的**起点**：硬件已经完全打通，能读能写能复位，
> 但"识别"这一步的路线还没选。看完第 6 节再动手。

---

## 1. 一句话状态

**传感器活了。** 主线上电后内核自己读出 `chip id: 10 a0 00 25`（厂商驱动认可的
`0x10 ?? ?? 0x25`），8 MHz / 1 MHz 都稳，寄存器窗口能读 256 字节。识别算法
（模板、匹配）**还没接**——这是下一步要决定的事。

---

## 2. 硬件事实（已核实）

| 项 | 值 | 出处 |
| --- | --- | --- |
| 传感器 | **Goodix GF3626ZS2**，侧边电容式 | `vendor/thh/ta/8888c03f…ta` 里的字符串；`ro.hardware.fp.sideCap=true` |
| 总线 | SPI1（`11010000.spi1`），CS=GPIO157，CLK=156，MI=158，MO=159 | 厂 DTS + pinctrl-mt6893.h |
| 复位 | GPIO14，低有效 | 厂 DTS `fingerprint_reset_low/high` |
| 中断 | GPIO15，上升沿，空闲上拉（`MTK_PUPD_SET_R1R0_10`）| 厂驱动 `IRQF_TRIGGER_RISING` |
| 供电 | **MT6360 LDO1 = "VFP" 3.3 V**，没有独立使能脚 | 厂 DTS `vfp-supply` |
| 中断号 | 主线枚举后是 **30** | 实测 |

### 踩到的坑：LDO1 默认只有 1.8 V

DTS 里 `mt_pmic_vmc_ldo_reg`/`mt_pmic_vfp_ldo_reg` 两个 label 名字是反的（反编译
产物），但 phandle 指向的 `ldo1` 的 `regulator-name` 确实是 `"VFP"`，所以原来是
对的。真正的问题是 **mt6360 稳压器节点根本没绑定驱动**，LDO1 停在 PMIC 默认的
1.8 V —— 传感器能"应答"但芯片 ID 读出来全 0。补上
`mediatek,mt6360-regulator` 节点、把 LDO1 设成 3.3 V always-on 之后立刻正常：

```
mt6360,ldo1: Bringing 1800000uV into 3300000-3300000uV
goodix_fp spi1.0: chip id: 10 a0 00 25
```

### 踩到的坑：CS 引脚没有 mux

LK 把 GPIO156/158/159 留在了 SPI1 功能（mode 1），**唯独 GPIO157（CSB）还是
GPIO（mode 0）**。也就是说 bootloader 没有把片选交给 SPI 控制器。必须由内核
pinctrl 显式设成 `SPI1_A_CSB`，否则总线是哑的。实测（`/dev/mem` 直接读
`0x10005300` 那一片 mode 寄存器）：

```
pin 14 mode 0   pin 15 mode 0
pin 156 mode 1  pin 157 mode 0   <- 只有这个不对
pin 158 mode 1  pin 159 mode 1
```

---

## 3. 这次做了什么

内核改动（提交 `b22a8b6`，`git/chopin-linux-7.3`）：

| 文件 | 内容 |
| --- | --- |
| `drivers/input/misc/goodix_fp.c` | 新增，~640 行 SPI 传输驱动 |
| `drivers/input/misc/Kconfig` / `Makefile` | `CONFIG_INPUT_GOODIX_FP` |
| `arch/arm64/boot/dts/mediatek/mt6893-xiaomi-chopin.dts` | SPI1 引脚组、指纹引脚组、`vfp-supply`、`reset-gpios`、`interrupts`、MT6360 `regulator` 节点（LDO1） |
| `arch/arm64/configs/chopin_defconfig` | `CONFIG_INPUT_GOODIX_FP=y` |

驱动只负责 userspace 干不了的三件事，**协议全留给用户态**（和厂商 HAL 的分工一致）：

1. VFP 稳压器 + 复位时序（拉低 15 ms，拉高 15 ms）
2. 手指中断（`spi->irq`，来自 DTS `interrupts`）
3. 原子的寄存器窗口（互斥锁，不会和复位/掉电打架）

### 寄存器协议

```
写：  [0xf0] [addr_hi] [addr_lo] [data ...]          单次 CS
读：  [0xf0] [addr_hi] [addr_lo]   (地址相)
      [0xf1] [dummy ...]           (数据相，全双工)
```

`0xf1` 那个字节读到的东西要丢掉，**数据从命令行后的第一个字节开始**。这个格式
和厂商驱动 `gf_spi_read_bytes_ree()` 完全一致。单次传输按 1021 字节切片，切片之间
重发地址相。

### 用户态接口

`/dev/goodix_fp`：

| ioctl | 用途 |
| --- | --- |
| `GOODIX_FP_IOC_READ` (`_IOWR('g',0x20, struct{__u16 addr;__u32 len;__u8* buf;})`) | 读窗口，单次最大 64 KB |
| `GOODIX_FP_IOC_WRITE` (`_IOW('g',0x21,…)`) | 写窗口 |
| `GOODIX_FP_IOC_RESET` (`_IO('g',0x22)`) | 复位脉冲 |
| `GOODIX_FP_IOC_IRQ_ENABLE/DISABLE` (`0x23`/`0x24`) | 中断开关 |
| `GOODIX_FP_IOC_IRQ_WAIT` (`_IOW('g',0x25,__u32)`) | 等一次手指中断，参数是超时 ms |

`read()`/`poll()` 也能拿中断事件（每个沿 1 字节）。

debugfs `/sys/kernel/debug/goodix_fp/`：`cmd`（`echo "r 0 8"` / `echo "w 0x10 01 02"`）、
`reset`、`speed`（`echo 8000000 > speed`）。

设备上没有编译器，用 python 直接打：

```python
import fcntl, ctypes, struct, os
fd = os.open("/dev/goodix_fp", os.O_RDWR)
def ioc(d, nr, size): return (d << 30) | (size << 16) | (ord("g") << 8) | nr
READ = ioc(3, 0x20, 16)
def rd(addr, n):
    b = ctypes.create_string_buffer(n)
    fcntl.ioctl(fd, READ, struct.pack("<H2xIQ", addr, n, ctypes.addressof(b)))
    return b.raw[:n]
print(rd(0, 8).hex(" "))
```

### 读到的寄存器地图（未配置状态，一次性快照）

```
0000: 10 a0 00 25 00 00 00 00 10 10 00 00 00 00 00 10   <- 0x00 芯片 ID
0010: 00 11 13 03 00 1f 00 00 00 00 00 00 00 00 00 00
0020: 00 10 00 01 ff ff 12 00 00 00 00 00 00 00 00 00
0030: 20 a0 05 0c 05 00 00 00 00 01 00 00 00 00 00 00
0058: 00 01 00 10 01 00 14 04 00 02 af a1 00 80 00 20
0080: 49 00 28 80 28 28 28 00 28 00 28 ...  (一串 0x28，像阈值表)
00a6: 80 3a
```

`0x0100` 只有开头几个字节（`01 00 00 00 00 00 00 f4`），再往上
（`0x1000/0x2000/0x4000/0x8000/0xf000`）读出来全是 0——芯片在未配置状态下
只有低端窗口有内容。

---

## 4. 厂商侧长什么样（为什么不能直接拿来用）

从 `super` 分区里捞出来的 `vendor_a`（本地副本 `work/fp/vendor/`）：

```
mfp-daemon → fingerprint.goodix.so → libgoodixhwfingerprint.so
           → libgf_hal.so ─┬─ /dev/goodix_fp ioctl（上电/复位/中断/时钟）
                           └─ libgf_ca.so → libteecli.so → TA 8888c03f…ta
```

* HAL 是 **Android bionic** 的 ELF（`for Android 31`），glibc 上跑不起来。
* **SPI 是安全世界在做**。内核驱动只管电源/复位/中断/**SPI 时钟门控**
  （`mt_spi_enable_master_clk`），真正的传输由 MITEe（MTK 自研 TEE，
  `g_tee_type=1`）里的 Goodix TA 完成。驱动里那句
  `"Not support poll opertion in TEE version"` 就是证据。
* 所以：**HAL 里没有 SPI 协议**，协议在 2.4 MB 的 `8888c03f….ta` 里（ARM32、strip）。
* 固件也不在 `vendor_a`：`firmware/168_fingerprintRecord_RTP.bin` 是"指纹录入完成"
  的**提示音**，不是固件。GF3626 跑内部 FW/OTP，HAL/TA 只下发 fw_cfg/cfg。
* `work/fp/libgf_hal.symbols.txt` 有 267 个函数名（从 `.gnu_debugdata` 里恢复的），
  包括 `hal_milan_an_series_*` 全家桶——milan 就是 GF36xx 这条产品线。

---

## 5. 当前能干什么 / 不能干什么

| 能力 | 状态 |
| --- | --- |
| 上电、复位、读 ID、读写任意寄存器窗口 | ✅ |
| 1–8 MHz 时钟切换、64 KB 连续读 | ✅ |
| 手指中断（IRQ 30） | ✅ 注册好了，但没配置过的芯片不会自己发中断 |
| 采集图像 / 录入 / 匹配 | ❌ 协议在 TA 里，还没逆 |

---

## 6. 两条路，先选一条

### A. 在用户态重写协议 + libfprint 自己匹配（"干净"路线）

要做什么：从 TA（或某个泄出的 Goodix REE 驱动）里逆出 milan 的
`DOWNLOAD_CFG / DOWNLOAD_FW / 采集 / 中断语义`，在用户态实现，再把图像喂给
libfprint 做匹配（libfprint 对图像型传感器就是这么干的，比如 `elanspi`）。

* 优点：不依赖闭源 TEE，全部主线 + 用户态；和 libfprint 的架构天然契合。
* 缺点：**核心工作在逆向 2.4 MB 的 TA**。milan 的 fw_cfg 是一大坨不透明数据，
  这部分大概只能从 TA 的代码里"抄"。
* 风险：高。

### B. 把 MITEe 搬到主线，复用厂商 TA（"能用"路线）

要做什么：

1. 移植 `drivers/misc/mediatek/teei/400/tz_driver`（**4350 行**）到主线：
   `mt_secure_call` → `arm_smccc_smc`、共享内存、`utr` 中断队列、`teei_fp`。
2. 写一个 glibc 的 TEEC 客户端，复刻 `libgf_ca.so`（**只有 15 KB**，很可逆）的
   会话/命令封装。
3. 复刻 `libgf_hal.so` 的 milan 层调用序列（`init/enroll/authenticate`）。
4. 用 libfprint 的自定义驱动或一个 CLI 收口。

* 优点：芯片时序、校准、算法、模板格式、OTP **全部白送**——这是唯一能真正
  "能用"的路线；顺带还能把 TA 的 SPI 流量打出来，反哺 A。
* 缺点：TEE 驱动移植是个大工程，而且依赖厂商闭源 ABI；主力工作量在内核而不是
  用户态。
* 风险：中（范围明确，但坑可能在 SMC/共享内存的细节里）。

### 建议

**先做 B 的第 1 步（TEE 驱动）的可行性验证**，因为它同时是 A 的加速器：
TEE 一旦通了，就能在 `teei_fp`/SPI 层把 TA 的真实命令流抓下来，A 的逆向难度
直接掉一个数量级。反过来先做 A，等于在没有任何参照的情况下猜协议。

如果只想尽快有一个"能用"的指纹，B 是唯一解；如果目标是"上游可接受的干净驱动"，
那 A 才是终点，但仍建议从 B 的抓包能力起步。

---

## 7. 刷机约束（重要，别再踩）

* **从 Linux 里写不进 `boot_a`**：`dd of=/dev/sdc32` 全部返回
  `critical target error, dev sdc`（UFS 对当前活动 boot 分区做了写保护）。
  分区本身 `ro=0`，但 UFS 层拒绝。
* **`boot_b` 可写**（`/dev/sdc56`，实测 1 MB 写回无错）。但这台机器**没有
  `misc` 分区**，A/B 槽位不在常规位置，暂时不知道怎么切槽，别指望它。
* 所以刷机只能走 fastboot：手机关机后**按住音量下 + 插 USB / 开机**。
* `fastboot` 需要 `/dev/bus/usb`，而 DSH 沙箱默认把 `/dev` 挂成精简 tmpfs
  （`lsusb` 能看到设备、`fastboot devices` 却是空的）。跑 fastboot 的那条命令
  要带 `sandbox_permissions: danger-full-access`。
* 刷机命令（已验证）：

```sh
cd /home/kily/project/mt6893
fastboot flash boot_a out-scp-v201-fp/boot-stockdt.img && fastboot reboot
```

* 回退：`out-scp-v199-733-final/boot-stockdt.img`（7.3.3 验证版）。

---

## 8. 下一步（按顺序）

1. **决定 A 还是 B**（见第 6 节）。在决定之前不要再往协议上砸时间。
2. 若选 B：先读 `teei/400/tz_driver` 的 `teei_smc_call.c` + `utr_tui_cmd.c`，
   确认 SMC 号、共享内存布局、中断队列在 MT6893 上是否和主线能对上。
3. 若选 A：先把 `8888c03f….ta` 用 `arm-linux-gnueabi-objdump` 反汇编，找
   `0xf0/0xf1` 附近的状态机和 OTP 读取流程。
4. 顺带（两路都要）：把 `speed` 调到 8 MHz 之后测一下大块读的稳定性（图像
   一次要几 KB 到几十 KB）。
5. 顺手可做：`mt6360-regulator` 节点目前只加了 LDO1，LDO5（VMCH，SD 卡
   `vmmc-supply` 还指向没绑定的老 label）可以一起补上。

---

## 9. 相关文件

| 想知道什么 | 看这个 |
| --- | --- |
| 内核改动 | 提交 `b22a8b6`（`drivers/input/misc/goodix_fp.c`）|
| 厂商驱动（REE SPI 格式、复位、GPIO 解析）| `n10p/vendor/kernel_mt6893-chopin_kvm/drivers/input/fingerprint/goodix_cap/gf_spi_tee.c` |
| 厂商 HAL / TA 清单 | `work/fp/MANIFEST.txt`、`work/fp/NOTES.md` |
| HAL 函数名 | `work/fp/libgf_hal.symbols.txt` |
| 镜像 | `out-scp-v200-fp/`（首版，ID 全 0）、`out-scp-v201-fp/`（当前，正常）|
