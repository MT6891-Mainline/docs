# 指纹移植交接（2026-11-08 晚）

新会话从这里开始。本文件只讲**指纹这条线**的现状、下一步和不要重踩的坑；
其余子系统见 `HANDOFF-2026-11-08.md`。

---

## 一句话状态

**硬件、TA 用户态、命令 ABI、token 信任链、enroll 会话全都通了**：
DETECT / INIT / **ALGO_INIT(1005)** / PRE_ENROLL / ENROLL / POST_ENROLL 在真机上
都返回 0，token + HMAC + challenge 校验通过，手指 IRQ 6/6 次正常触发。
现在**唯一卡点**是 `CAMERA_CAPTURE(1021)`：它只做一次"把传感器切到 FTD down"
的 arm（+20 次 SPI），之后再调用就一个 SPI 都不发，抓不到图。**下一轮从这里开始。**

---

## 1. 已经做成的

| 层 | 状态 | 位置 |
| --- | --- | --- |
| 内核 SPI 驱动（复位/中断/稳压器/任意 tx-rx/32 位 compat） | ✅ 验证通过 | `git/chopin-linux-7.3` 提交 `b22a8b6`、`2e40244`、`5b1be39` |
| **SPI CS 时序修正** | ✅ 真机 DTB 里三项都是 10000 ns（`/proc/device-tree/spi1@11010000/fingerprint@0/`） | 提交 `94b57df` |
| TA 用户态宿主机 | ✅ 加载/重定位/init/命令派发 | `port/fingerprint/ta32/ta_host32.c` |
| **命令 ABI（含 payload）** | ✅ `{cmd,in_len,out_cap,status}` + payload | 见 `TA-IN-USERSPACE.md` M1.30 |
| **DETECT / INIT** | ✅ 578 / 1190 次 SPI，OTP CRC 通过 | `./ta_host32 goodix.ta plan plan.txt` |
| **token + chain（HMAC-SHA256）** | ✅ 真机通过 | 本文件 §2 |
| **PRE_ENROLL / ENROLL** | ⚠️ 命令能进、校验通过，卡在 fingerlist | 本文件 §2 |
| 离线 qemu 回放 + 逐指令 trace | ✅ 与真机逐字节一致 | `port/fingerprint/ta32/replay.sh`、`ta_xrun.py` |
| 跨进程不重入 | ✅ 已确认（DETECT+INIT 必须同一进程） | `plan` 模式 |

### CS 时序这个坑（务必记住）

原厂 `gf_spi_tee.c` 里的 `mt_chip_conf`：

```c
.setuptime = 10, .holdtime = 10, .cs_idletime = 10,   /* 10 个 SPI 时钟 */
.high_time = 50, .low_time = 50,                      /* 1 MHz */
```

主线 DTS 默认这三个都是 0。补上 `spi-cs-setup/hold/inactive-delay-ns = <10000>`
之前，寄存器读回的是**规律垃圾**（`40 10 20 10 40…`），于是 TA 的所有判断都建立在
错数据上——包括"手指一直存在"这个假象。

---

## 2. 卡住的地方（唯一未解）

```
enroll: round N CAPTURE -> 0 status=0x0 spi=1210 (+0)
```

`1021` 在 `desc{0,0,1}` + `&cmd` + `&origin` 形态下，**只在算法还没初始化时**
做一次 arm（1190 → 1210，配置传感器 + 切 FTD down），之后：

* 再调 1021：+0 SPI，一个导入 stub 都不碰（`last=-`）
* 先跑过 1005（算法初始化）再调 1021：连 arm 都不做，直接返回
* `req` 形态、`desc[0]=1/2`、`len` 换值：全是 +0 SPI
* `desc[2]=2` → BAD_PARAMS，`desc[2]=3` → UNKNOWN_CMD

逐指令 trace 的尾部显示它走的是 `0x175d0` 里 `(CONST+cmd) > 0x5b` 的
"不支持"分支（`0x176b6 → 0x176d2 → 0x17620`），而真正查表跳转的
`0x176d4`（`ldr r0,[r7,r1]; mov pc,r0`，表在 `0x1793c`）一次都没执行到。

### 上一轮的两个卡点都已经解掉

* **SIGILL / "Illegal instruction"**：不是 TA 崩溃，是我们给的请求缓冲区太小，
  TA 越界写把 musl 堆踩坏，musl 用 `udf` 主动陷阱。默认 4 KiB out 之后消失。
* **`max_count = 0`**：`cmd 1005`（算法初始化）本来就会设
  `max_fingers_per_user=5`、`max_fingers=32`，不需要再 poke。

## 3. 下一步（按优先级，全部不需要刷机）

1. **把 `0x175d0` 的派发读透**（最直接）：
   * `0x1764a: ldr r3,[pc,#0x2c0]` 取出那个常量，看 `CONST+cmd` 为什么 > 0x5b；
   * 跳转表在 `0x1793c`，`table[(CONST+cmd)]`，反推这个入口到底接受哪些 cmd；
   * 目标：找到 capture 真正的入口（很可能不是走 sub=1 这条）。
2. **反 `hal_milan_irq` / `irq_image_enroll_or_authenticate`**：
   看手指沿到来之后 HAL 实际发的是哪条命令（已知 0x401/0x402 是
   `gf_hal_notify_*` 的通知号，不是 TA 命令）。
3. **验证"arm 之后"的真实顺序**：可能 capture 需要先 `1005` 再 `1007` 再
   `1021`，也可能是"先 capture（arm）→ 等 IRQ → 再 capture（取图）"，
   两种都用 `plan` + `GF_CAPS` 快速对比。
4. 抓图成功后：`POST_ENROLL(1008)` 落盘 → 看 `/var/lib/goodixta/finger_0_0.so`
   是否出现 → 再做 authenticate。

### 现在可以直接复跑的命令

```sh
cd /root/ta
# 完整 enroll：DETECT → INIT → ALGO_INIT → PRE_ENROLL → ENROLL，
# 之后每按一次手指就 capture 一次，最后 POST_ENROLL
GF_PATCH_GATE=2 ./ta_host32 goodix.ta enroll 6 20000

# 想试别的采集命令：
GF_PATCH_GATE=2 GF_CAPS=1021,1005 ./ta_host32 goodix.ta enroll 1 20000

# 逐命令调试（5 列：cmd 文件 out_cap sub len_hint）
GF_PATCH_GATE=2 ./ta_host32 goodix.ta plan plan.txt
```

## 4. 不要重踩的坑（都实测过）

| 假设 | 结论 |
| --- | --- |
| 寄存器 `0x58` 是手指状态位，spoof 它 | ❌ 试过 `0/0x40/0xff/0xffff`，门不读它 |
| `desc[1]` 传入重试计数 `i` | ❌ TA 会覆盖 `desc[1]`，`i` 仍为 0 |
| capture 之后插一次 detect 再 capture | ❌ 流程不可重入，第二次 detect 卡死 |
| 等 IRQ 再采集（真机、真实数据） | ❌ 中断正常触发，但 TA 把结果缓存，SPI 计数不变 |
| DMA 传输损坏了那 273 字节配置 | ❌ 同窗口 DMA 读 96 B 与 FIFO 分 4×24 B **逐字节相同** |
| `desc[0]`/`desc[2]` 是"模块/子命令"选择器 | ⚠️ 有状态：同一组值在不同前置状态下会走不同入口 |
| 手工从十六进制推 `r5=12` | ❌ 静态读分支不可靠，qemu 实测是 `r5=0` |
| `GF_PATCH_GATE=1`（NOP `0x25cd8`）能过门 | ❌ 那不是失败点；要 NOP 12 个区域分支（`=2`） |
| `milan_check_finger_up` 读状态位 | ❌ 是 12 个区域的 fdt 平均值 vs `fdt_down.data` 比较 |
| `i` 是重试计数、由调用方推进 | ❌ `i` 是区域编号（0..11，全过时 `0xc`） |
| token 用固定 32 字节假 HMAC 就行 | ❌ TA 用 keymaster key 做真 HMAC-SHA256 校验 |
| `ut_pf_ts_cp_open/size` 成功返回 0 | ❌ 返回句柄/大小，0 是错误（导致"对象存在但大小 0"） |
| `ut_pf_km_get_hmac_key(buf, len)` | ❌ 第二参是长度**指针** |
| 回放按内容匹配 | ⚠️ 按**序号**取应答，流程一变值就错位（曾误报 dac error） |
| 一个进程调一条命令 | ❌ TA 状态不可跨进程重建；`plan` 一次跑一串 |
| "Illegal instruction" 是 TA 崩了 | ❌ 是 musl `a_crash()`：请求缓冲区太小把堆写坏 → 默认 4 KiB out |
| `max_count=0` 要用 poke | ❌ 先调 `cmd 1005`，它会设 `max_fingers_per_user=5` |
| 每轮重跑 `pre_enroll` | ❌ 会话开着时会被拒（`g_enrolling_finger is not NULL`） |
| 1025/1026 是采集命令 | ❌ 此固件返回 `UNKNOWN_CMD` |
| `0x401/0x402` 是 TA 命令 | ❌ 是 HAL 的 `gf_hal_notify_*` 通知号 |

---

## 5. 工具与命令

### 设备

```sh
cd /home/kily/project/mt6893
export CHOPIN_HOST=192.168.0.108
bash work/scripts/dev.sh 'uname -r'          # 当前应为 v204
```

`ta_host32` 和 `goodix.ta` 在设备上放在 `/root/ta/`。

### 常用运行方式

```sh
cd /root/ta
./ta_host32 goodix.ta cap 1                  # detect → init → capture
./ta_host32 goodix.ta irqcap 8               # 等 IRQ → capture（循环）
./ta_host32 goodix.ta probe 1021 0 1         # 单条命令
./ta_host32 goodix.ta req 1007 enroll.bin 0 1   # 单条带 payload
./ta_host32 goodix.ta plan plan.txt          # 一串命令（同一进程！）
GF_PATCH_GATE=2 ./ta_host32 goodix.ta plan plan.txt
GF_POKE=0x2b9dcc=5 ./ta_host32 ...           # 每条命令前写数据段
./ta_host32 goodix.ta hmactest               # HMAC-SHA256 自检（RFC 4231）
GF_DUMP3=0x2b9d80:0x80 ./ta_host32 ... plan plan.txt
GF_RECORD=/root/ta/x.rec ...                 # 录 SPI
```

`plan.txt` 每行：`<cmd> <payload 文件|-> [out_cap] [sub]`。

### 离线（主机）

```sh
cd /home/kily/project/mt6893/port/fingerprint/ta32
./replay.sh cap 1                                    # 回放，不需要手机
DFILTER=0x25b18+0x2b0 ./replay.sh cap 1              # 指令+寄存器轨迹 → qemu.log

# 逐指令 trace（-d exec,cpu 是每次执行都 dump 寄存器）
GF_REPLAY=spi2.rec qemu-arm -B 0x10000000 -one-insn-per-tb \
    -d exec,cpu -dfilter 0x25b18+0x2b0 -D work/t.log ./ta_host32 goodix.ta cap 1
PYTHONPATH=/home/kily/project/mt6893/work/env/pylibs python3 ta_xrun.py work/t.log
```

依赖：`qemu-arm`（主机已装）、capstone 在 `work/env/pylibs`。

> `-D` 不要写 `/tmp`：每次 bash 是独立 mount namespace，下一条命令读不到。

### 关键地址

| 东西 | 地址 |
| --- | --- |
| TA 里的日志封装 | `0xa29c` |
| `milan_check_finger_up` 入口 | `0x25b18`（Thumb） |
| 12 个区域的失败分支（`GF_PATCH_GATE=2` NOP 的） | `0x25ba0`…`0x25cd0` |
| 汇总日志（fdt/down/area） | `0x25d5e` |
| 共享门 `r5 < [r6+0xfc]` | `0x25cd2`（`[r6+0xfc]`=10，`r5`=区域号） |
| 传感器配置结构 `config` | `[0x2ab324+0x3c]` |
| `delta_up` | `config+0x634+(0x82-ldrh[config+0x630])` = 0x13 |
| `fdt_down.data[]` | `config+0x424+area`，`area = 0x84-ldrh[config+0x420]+2i` |
| fingerlist `max_count` | `0x2b9cc8+0x104`（正常由 cmd 1005 设成 5） |
| `gf_ta_enroll` token 块 | payload+`0x6e`（`msg[0x25] || HMAC[0x20]`） |
| 通用命令入口 `0x175d0` | `(CONST+cmd) > 0x5b` 走"不支持"分支；跳转表在 `0x1793c` |
| 芯片 ID 寄存器 | `0x0000`（`10 a0 00 25`） |
| 手指/DAC 相关读 | `0x0058`（2 B）、`0x00a2`（24 B）、`0x0106`（2 B） |
| DAC 校验读 | `0x0220`/`0x0236`/`0x0238`/`0x023a`，期望值 `[0x2ab324+0xe4]`=0xc6 |

### 命令号速查（本固件实测）

| cmd | 名称 | 备注 |
| --- | --- | --- |
| 1000 | DETECT_SENSOR | 578 SPI |
| 1001 | INIT | 1190 SPI，OTP CRC 通过；payload 348 B |
| 1005 | **ALGO_INIT** | 设 max_fingers=32 / per_user=5，31 SPI |
| 1006 | PRE_ENROLL | 112 B，challenge 在 out+0x68（`cp_rd_random`） |
| 1007 | ENROLL | 180 B，token 在 +0x6e；会话存在 `g_enrolling_finger` |
| 1008 | POST_ENROLL | 模板落盘 |
| 1010 | ESD check | +4 SPI |
| 1020 | SET_SAFE_CLASS | 104 B，class 在 +0x64 |
| 1021 | CAMERA_CAPTURE | 100 B 请求，**当前只 arm 不取图** |
| 1025/1026 | — | `GF_ERROR_UNKNOWN_CMD` |

## 6. 刷机（必须走 fastboot）

```sh
cd /home/kily/project/mt6893
fastboot flash boot_a out-scp-v204-fp/boot-stockdt.img && fastboot reboot
```

- **Linux 里写不进 `boot_a`**（UFS 对活动 boot 分区有写保护，`dd` 返回
  `critical target error`）；`boot_b` 可写，但这台机器没有 `misc` 分区，
  槽位不好切。
- `fastboot` 需要 `/dev/bus/usb`，而 DSH 沙箱把 `/dev` 挂成精简 tmpfs，
  所以那条命令要带 `sandbox_permissions: danger-full-access`。
- 回退镜像：`out-scp-v199-733-final/boot-stockdt.img` 或 `out-scp-v203-fp`。

---

## 7. 厂商侧资料在哪

| 想知道什么 | 看这个 |
| --- | --- |
| 原厂内核驱动（SPI 时序、复位、REE 读写格式） | `n10p/vendor/kernel_mt6893-chopin_kvm/drivers/input/fingerprint/goodix_cap/gf_spi_tee.c` |
| 厂商 HAL / TA 清单 | `work/fp/MANIFEST.txt`、`work/fp/NOTES.md` |
| HAL 符号表（267 个） | `work/fp/libgf_hal.symbols.txt` |
| TA 本体（ELF32/ARM，2.4 MB） | `work/fp/vendor/thh/ta/8888c03f….ta` |
| SPI 录像 | `port/fingerprint/ta32/spi.rec`（v203，坏数据）、`spi2.rec`（v204，真实数据） |

厂商栈结构：`mfp-daemon → fingerprint.goodix.so → libgoodixhwfingerprint.so →
libgf_hal.so → libgf_ca.so → libteecli → TA(8888c03f)`，**SPI 由安全世界的 TA
驱动**，内核驱动只管电源/复位/中断/时钟门控。

---

## 8. 环境、构建与刷机（从零到手）

### 连设备

```sh
cd /home/kily/project/mt6893
export CHOPIN_HOST=192.168.0.108          # 手机 IP（会变，先 ping 确认）
bash work/scripts/dev.sh 'uname -r'       # 密码 chopin，脚本内部用 askpass
```

### 主机依赖

| 东西 | 位置 |
| --- | --- |
| ARM32 工具链（静态 musl） | `work/env/arm-linux-musleabihf-cross/bin/arm-linux-musleabihf-` |
| capstone（反汇编脚本用） | `work/env/pylibs`，用时 `PYTHONPATH=...` |
| qemu-arm | 系统自带 `/usr/bin/qemu-arm` |
| `arm-none-eabi-objdump` | 系统自带（读 TA 用，记得 `-M force-thumb` 分区域） |

### 编内核 + 打包

```sh
cd /home/kily/project/mt6893
./port/tools/build-kernel.sh                # 会把 DTB 重新编好并嵌进 Image
rm -rf out-scp-vXXX-fp
INITRAMFS_MODE=rootfs   KERNEL=$PWD/build/chopin-7.3/arch/arm64/boot/Image   DTB=$PWD/build/chopin-7.3/arch/arm64/boot/dts/mediatek/mt6893-xiaomi-chopin.dtb   STOCK_DTB=$PWD/git/chopin-support/dtb/stock-base.dtb   port/tools/build-images.sh out-scp-vXXX-fp
```

注意：**改 DTS 必须重编内核**（DTB 嵌在 Image 里），只换 boot 里的 DTB 没用。

### 编 ta_host32

```sh
cd port/fingerprint/ta32 && make          # 产出 ta_host32（静态 ARM32）
```

链接地址是 `-Ttext=0x800000`（必须离 TA 的 `0x8000–0x33c918` 近，日志 hook 的
ARM `b` 只有 ±32 MB 范围）。改这个地址会让 hook 失效。

---

## 9. 为什么没走另外两条路（决策记录，别再重新论证一遍）

| 方案 | 结论 | 理由 |
| --- | --- | --- |
| **A. 移植 MITEe TEE 驱动，照 Android 架构跑** | ❌ 放弃 | `teei/400/tz_driver` 共 4350 行厂商代码，依赖 `mt_secure_call`/共享内存/`utr` 中断队列等 MTK 私有基础设施；而且跑起来后仍要 Android HAL（bionic ELF，glibc 跑不了）。xaga 那边试过，ioctl 直接硬重启 |
| **B. 自己写 SPI 驱动 + libfprint 自己匹配** | ❌ 放弃 | xaga 实证：能出图但**画质差、libfprint 匹配抽风**。而且协议（fw_cfg、校准、算法）全在 TA 里，绕不开 |
| **C. 用户态跑厂商 TA（本方案）** | ✅ 采用，已验证可行 | TA 只依赖 46 个导入符号 + 1 个 SPI 入口，宿主 32 位静态二进制即可 |

**结论**：C 已经是三者里最省力的；中途不要再回 A/B，除非 C 彻底走不通。

---

## 10. 知识薄弱区（继续推进时会撞到的）

1. **`gf_memmgr_config.so` 的真实格式** —— TA 每次启动都报缺失并 unlink 非法内容。
   已知试过全零 4 KiB、HAL 的 8 KiB rodata，**都被拒**。
   线索：`libgf_hal.so` 有 `hal_test_set_memmgr_config@0x2cca4` /
   `hal_test_get_memmgr_config@0x30558`，可以从它们的参数结构反推格式。
2. **录入/识别的 module/sub 表** —— 每个 `hal_milan_*` 函数都是
   `mov w0,#id; bl invoke`，可以从 `libgf_hal.so` 的 debugdata 地址逐个读出来
   （已有脚本 `ta_1065.py` 演示了怎么找 `movw`）。
3. **keymaster / 模板加密一致性** —— 目前用确定性 key（照 xaga 的做法），
   模板只能在本 host 之间可移植。如果要做持久化，需要确认 TA 是否校验 HMAC。
4. **图像几何** —— TA 自己报 `image extension row = 44, col = 160`，但真图还没
   取到过，所以 160×44 只是"它认为的尺寸"。

---

## 11. 投入与节奏（给后面的人一个预期）

- 本次会话消耗了 **~23 轮工具交互**，从"能读芯片 ID"推进到"TA 跑起来 + 数据正确"。
- 其中**单点最耗时的是 CS 时序那个 bug**（约 6-8 轮才定位），因为它的表现是
  "数据看起来像数据但其实是垃圾"。
- 一旦有了 qemu 回放环境，后续每次实验从"刷机 + 按手指 + 20 秒"变成
  "主机上 1-2 秒"，效率差一个数量级。**这个环境是本次最有价值的工具遗产**，
  再往下推进时优先用它，尽量不要每次上真机。

---

## 12. 完整过程记录

细节（每一步的假设、实验、结论、日志原文）都在：

**[`TA-IN-USERSPACE.md`](TA-IN-USERSPACE.md)** —— M1.1 ~ M1.26

读它比读这份交接更全面；本文件是"接手就能干活"的浓缩版。
