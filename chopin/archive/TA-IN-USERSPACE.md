# 路线 C：把 Goodix TA 跑在用户态

> 2026-11-08。参考 xaga（Redmi Note 11T Pro）上已经跑通的
> [fpc1540-ta](https://github.com/MT6895-Mainline/fpc1540-ta)：
> 把厂商 trustlet 当成普通 ELF 加载，把 TEE 的运行库替换成我们自己的 stub，
> 让 TA 里的**真实驱动 + 算法**在用户态跑起来。
>
> 这条路取代了本文末尾原来"逆 TA 协议"和"移植 MITEe 驱动"两个方案。
> 硬件侧已经就绪，见 [`FINGERPRINT-PORT.md`](FINGERPRINT-PORT.md)。

---

## 1. 为什么走这条路

xaga 上的结论很直接（原文）：

> 参考 `SanniZ/fpc1020-driver` 写了一个 fpc1540 的内核 SPI 驱动，**能出图、
> 能看见指纹纹路**，但**画质很差**，libfprint 手工测试**匹配很抽风**。
> 所以改成复用 TA 里的厂商匹配算法。

Goodix GF3626ZS2 是同一类东西：**match-on-host**，真实驱动和算法都在
`8888c03f….ta` 里，内核驱动和 HAL 都只是 TEE 的 client。自己写 SPI 驱动 +
libfprint 匹配，大概率会复刻同一个结局（画质/校准过不去，因为
fw_cfg、OTP 校准数据、图像增强全在 TA 里）。

所以：**不逆协议，直接把 TA 跑起来。**

---

## 2. 我们这个 TA 长什么样（实测）

```
file:  ELF 32-bit LSB executable, ARM, EABI5, dynamically linked,
       interpreter rom/libld-l4.so, stripped
入口:  0x8f9c
段:    LOAD 0x000000 vaddr 0x8000    filesz 0x1166e0  R E
       LOAD 0x1166e0 vaddr 0x1266e0  filesz 0x145a0c  RW (bss 到 0x33c918)
```

**关键：它只导出 3 个有意义的符号**

| 符号 | 地址 | 作用 |
| --- | --- | --- |
| `ut_pf_fp_init` | `0x91e0` | TA 初始化 |
| `ut_pf_fp_invoke_command` | `0x9220` | **命令入口**：一次调用 = 一个 HAL 命令 |
| `_start` / `_init` / `_fini` | `0x8f9c` / `0x8ca8` / `0xf1ae8` | uClibc 启动 |

这比 xaga 那边**简单一档**：fpc1540.ta 需要逆 `libfp_server` 的主循环才能把命令送进去，
我们的 TA 有两个干净的导出函数，直接 `dlsym` 式地调用就行。

**导入只有 46 个符号**（这正是 stub 的完整清单）：

### 2.1 MicroTrust 运行库服务（18 个）

| 组 | 符号 | 我们的实现 |
| --- | --- | --- |
| **SPI** | `ut_pf_spi_send_and_receive` | **桥到 `/dev/goodix_fp`**（已经能读能写） |
| 模板存储 | `ut_pf_ts_cp_open/close/read/write/size/unlink` | 用文件实现（`/var/lib/goodixta/`） |
| 加密 | `ut_pf_cp_open/close`、`ut_pf_cp_mc_start/update/finish`（memcrypto）、`ut_pf_cp_sc_start/finish`（secure channel）、`ut_pf_cp_rd_random` | AES + RNG，一次性实现 |
| Keymaster | `ut_pf_km_get_hmac_key` | 确定性 key 文件（照 xaga 的 `km_key` 做法） |
| 其它 | `ut_pf_log_msg`、`ut_pf_time_get_system_time`、`ut_pf_time_get_microseconds` | 打印 / `clock_gettime` |

### 2.2 libc（28 个）

`malloc free realloc memcpy memmove memset memcmp strlen sprintf vsprintf
abort raise sin cos tan exp floor` + C++ 运行时
（`__cxa_begin_cleanup __cxa_call_unexpected __cxa_type_match
__gnu_Unwind_Find_exidx _ITM_deregisterTMCloneTable
_ITM_registerTMCloneTable _Jv_RegisterClasses`）+ `main`、`__uClibc_main`。

后两个说明：**TA 的 `main` 是外部提供的**，`_start` 的代码是

```
ldr r0, =0x8cd8      @ main（PLT，指向导入的 main）
ldr r3, =0x8ca8      @ _init
b   __uClibc_main@plt
```

也就是说加载器要给 `main` 和 `__uClibc_main` 各一个实现，然后 TA 自己的
`ut_pf_fp_init` / `ut_pf_fp_invoke_command` 才是我们真正要调的。

### 2.3 命令通道（下一步要解决的唯一未知）

`ut_pf_fp_invoke_command` 的开头是两级派发，实测指令：

```
cmp r3, #1000      ; 外层命令号，1000 是特殊路径
cmp [r4],   #0/#1/#2   ; 一层：类型
cmp [r4+8], #1/#2/#3/#4 ; 二层：子命令
```

和 xaga 文档里的 `route <mod> <sub>` 是同一个味道。**buffer 布局不用猜**：
`libgf_ca.so` 只有 15 KB 且带 `.gnu_debugdata`，从它的
`ut_pf_gp_transfer_user_data` 调用点把结构体抄出来即可。

---

## 3. 和 xaga 的差异（也是新的工作量）

| 项 | xaga (fpc1540) | 我们 (GF3626) |
| --- | --- | --- |
| TA 位数 | ELF64 aarch64 | **ELF32 ARM**（AArch32）|
| host 工具链 | 本机 aarch64 gcc | **arm-linux-musleabihf 交叉、静态链接**（已下到 `work/env/arm-linux-musleabihf-cross/`）|
| 内核要求 | — | `CONFIG_COMPAT=y` **已经开着**，MT6893 的 A78/A55 支持 AArch32 |
| TA 基址 | `0x400000` | `0x8000`（**低于 mmap_min_addr=65536**，需要 root + `CAP_SYS_RAWIO` 才能 `MAP_FIXED`）|
| host 链接地址 | `0x10000000` | 同样要避开 `0x8000–0x340000`，用 `0x10000000` |
| 入口 | 逆出来的 server 循环 | 直接调 `ut_pf_fp_invoke_command` |
| SPI | `FPC1540_IOC_RAW_XFER` | `/dev/goodix_fp` 的 `GOODIX_FP_IOC_READ/WRITE`（已实现）|

---

## 4. 落地步骤

### ~~M1 ta_host32 骨架~~ ✅ 已完成（2026-11-08）

实测结果（`port/fingerprint/ta32/ta_host32.c`，`work/fp/ta32/ta_host32`）：

```
$ /root/ta/ta_host32 goodix.ta
ta_host32: loading goodix.ta
relocations done, 0 missing
/dev/goodix_fp = 3
calling ut_pf_fp_init()
[ts] open("gf_memmgr_config.so", flags=0)
[ta] [GF_TA][E][gf_tee_storage][] exit. err=GF_ERROR_OPEN_SECURE_OBJECT_FAILED, errno=1033
[ta] [GF_TA][E][gf_memmgr_manager][gf_memmgr_load_config] not found config file, so=gf_memmgr_config.so
ut_pf_fp_init() -> 0, 0 spi transactions
```

也就是说：**TA 的代码在用户态跑起来了**，我们提供的 stub 被它调用，
它自己的 `[GF_TA]` 日志能打出来，`ut_pf_fp_init()` 返回 0。

已经踩通/确认的点：

* `CONFIG_COMPAT=y` + A78/A55 的 AArch32：`/root/hello32`（静态 musl ARM32）
  在设备上正常跑，`sizeof(void*)=4`；
* `vm.mmap_min_addr = 4096`，所以 TA 的 `0x8000` 基址**不需要**降 sysctl；
* host 必须 `-static -no-pie -Wl,-Ttext=0x10000000 -Wl,-z,notext`
  （不加 `-z,notext` 会报 "read-only segment has dynamic relocations"）；
* `main` 和 `__uClibc_main` 是 TA 的**导入**符号，我们直接不给实现（填 0），
  因为入口是 `ut_pf_fp_init` / `ut_pf_fp_invoke_command`；
* 剩下两个待办：`gf_memmgr_config.so`（TA 的存储里没有就走"首次启动"路径，
  现在它会 unlink 掉空文件）和 `ut_pf_fp_invoke_command` 的 buffer 布局。

### M1 原始计划（留档）

1. `arm-linux-musleabihf-gcc -static -Ttext=0x10000000` 编译 host。
2. 手写 ELF loader：解析 `PT_LOAD`，`mmap(MAP_FIXED)` 到 `0x8000/0x1266e0`，
   处理 `R_ARM_JUMP_SLOT`（42 个）+ `R_ARM_GLOB_DAT`（2 个）→ 指向 stub 表。
3. 提供 `__uClibc_main`（转调传进来的 `main` 指针）和 `main`（空跑/等待命令）。
4. 调 `ut_pf_fp_init()`，看它要什么（大概率要 SPI 和 random）。

**验证标准**：`ut_pf_fp_init` 返回 0，并且 `ut_pf_log_msg` 里出现
`[gf_ta_chip_series][…] CHIP_TYPE=…`（TA 自己打印的芯片型号）。

### M1.5 已经打通 `ut_pf_fp_invoke_command`（2026-11-08）

调用约定（从 `ut_pf_fp_invoke_command` 的反汇编推出来，并用实测验证）：

```c
long ut_pf_fp_invoke_command(uint32_t *desc, uint32_t *cmd, uint32_t *origin);
/*                                 r0           r1          r2      */
/*  r0 -> { u32 module; u32 ?; u32 subcmd; u32 status; u32 size; ... }
 *  r1 -> 传输层命令号（1000 = 特殊路径，读码时先命中它）
 *  r2 -> origin（TA 不写它）
 */
```

实测：

```
[ta] init -> 0
mod=0 sub=0 -> 0        (desc 没被改，没触发 SPI)
mod=0 sub=1 -> [cp] open(0xffddfdec, 0x7, 0x7004)   <-- 然后进程死在这
```

这个 `module/subcmd` 的 16 字节描述符和 xaga 那边
`docs/hal-protocol.md` 记的 `{u32 module; u32 subcmd; u32 status; u32 size}`
是**同一个东西**——Goodix 的 HAL 只是把同样的框架换了个名字
（`libgf_hal.so` 里的 `hal_milan_an_series_*`）。所以那一整套 module/subcmd
表可以照抄结构，只需要从 `libgf_hal.so` 里把 Goodix 的具体编号读出来。

**当前卡点**：`ut_pf_cp_open(algo=0x7, …)` —— 加密 provider 的 stub 语义。
现在还只是"打日志返回 0"，TA 拿到假的结果之后就挂了。和 `ut_pf_ts_cp_*`
一样，需要按调用点把参数含义弄清楚（大概率是 `TEE_ALG_*` 算法号 + key +
keylen），再把 `ut_pf_cp_mc_*`（memcrypto）实现成真正的 AES。


### M1.6 SPI 桥通了（2026-11-08）

把 `ut_pf_cp_*` 换成 xaga 那边验证过的语义（`open` 返回上下文指针、
`mc_finish` 吐确定性 32 字节 key、`sc_*` 直通）之后，TA 不再挂，整轮 sweep
跑完。**TA 已经开始真的跟传感器说话了**：

```
mod=0 sub=0 -> 0 spi=0
[spi#1] txlen=3 rxlen=0 tx=f0 00 00   ← 就是我们的地址相，读 0x0000
mod=0 sub=1 -> 0 spi=1
mod=0 sub=2 -> [gf_ta_test_chip_series] invalid parameters. cmd_id=GF_CMD_DETECT_SENSOR, cmd_id_code=1000
mod=0 sub=3 -> [gf_ta_dump] invalid parameters cmd_id=GF_CMD_DETECT_SENSOR
```

### 命令描述符的真实语义

| 字段 | 含义 |
| --- | --- |
| `desc[0]` | module（0..2 都进同一族，暂时没看出区别）|
| `desc[2]` | **入口族**：0/1 = 正常路径，2 = `gf_ta_test_*`，3 = `gf_ta_dump_*` |
| `*cmd` | **GF_CMD_* 命令号**，`1000 = GF_CMD_DETECT_SENSOR` |

`desc[2]` 这条路是**有状态的**：先调一次全零描述符（正常路径，会做一次 SPI
读芯片）之后，后续调用才会被路由到同一族；单独直接打 `sub=2` 会落到
`gf_ta_user_cmd_entry_point` 报 `GF_ERROR_BAD_PARAMS`。所以"盲扫"不可靠，
下一步要按 HAL 的真实调用序列走。

### 下一步（明确）

1. 从 `libgf_hal.so` 的 `hal_milan_an_series_*`（debugdata 里带**地址**：
   `hal_milan_an_series_init=0x416a8`、`enroll=0x41ce0` …）反出每个命令的
   `{module, sub, GF_CMD}` 三元组和调用顺序。反汇编已经验证可用。
2. 用 TA 自己的错误日志当"枚举器"（它会把 `cmd_id` 名字打出来），但必须在
   正确的前置状态下调。
3. 抓 `CAMERA_CAPTURE` 的 SPI 流量，确认帧格式和我们内核侧一致，再看画质。


### M1.7 GF_CMD 表开始成形（2026-11-08）

从 `libgf_hal.so` 的公共层反出来（debugdata 带地址，`gf_hal_invoke_command` =
`invoke(type, cmd_id, in, inlen)`，`type=1` 走正常路径）：

| cmd_id | 名字 | 依据 |
| --- | --- | --- |
| 1000 | `GF_CMD_DETECT_SENSOR` | TA 自己的错误日志 |
| 1001 | `GF_CMD_INIT` | `hal_milan_an_series_init` 里 `mov w0,#0x3e9` |
| 1016 | ? | 同函数里 `mov w0,#0x3f8` |
| 1021 | `GF_CMD_CAMERA_CAPTURE` | `gf_hal_common_camera_capture` 里 `mov w0,#0x3fd` |

其余命令可以照这个办法一个个挖：`gf_hal_common_<动作>` 的头部就是 `w0 = cmd`。

### 当前卡点：`milan_init_configuration` 拿不到 chip id

```
$ ta_host32 goodix.ta probe 1001 1 1
[gf_milan][milan_init_configuration] unsupport sensor, chip id=0x00000000
→ GF_ERROR_SENSOR_NOT_AVAILABLE (1062)
```

两件事要修：

1. **顺序**：`DETECT_SENSOR`(1000) 才是读芯片 ID 的命令（sweep 里它做过一次
   `f0 00 00`），`INIT`(1001) 必须排它后面，且同一个进程内；
2. **SPI stub 的返回语义**：读芯片 ID 只出现了地址相 `tx=f0 00 00`，
   数据相 `[0xF1]` 根本没发出去 → 说明 TA 在第一帧之后就判定失败。
   已照 xaga 的做法加了"TEE 服务返回 1 字节状态 + payload 右移一位"，
   但还没验证，下一轮从这里继续。

参考实现（xaga）里那句注释就是关键：
> Large reads: the TEE platform clocks a dummy byte after the command.
> Small reads instead get a leading status byte in the response.


### M1.8 卡点的真因：32 位进程打不到 ioctl（2026-11-08）

`gf_hw_detect_sensor` 读寄存器 0x0000 报 `GF_ERROR_SPI_TRANSFER_ERROR (1023)`，
再往下追是 `[gf_spi_write] drSpiSend failed ret=0xFFFFFFFF` —— 我们的 stub 返回了
`-1`。`errno = 25 (ENOTTY)`。

原因是 **compat ioctl**：`goodix_fp` 有 `.unlocked_ioctl` 但没有 `.compat_ioctl`，
32 位进程（我们的 ta_host32）根本进不到驱动里。64 位的 python 调同一个 ioctl
是成功的：

```
64-bit ioctl ok  rx=00 00 00 00 ...     # python3 直接打 0xc0206726
32-bit ioctl    -> ENOTTY              # ta_host32
```

修法一行：`.compat_ioctl = compat_ptr_ioctl`（提交 `5b1be39`，镜像 v203）。
顺手把 `gf_raw_xfer` 的 tx/rx 不等长改成补零对齐，这样"发 3 字节、不要回读"
也能工作。

**这一条特别值得记**：它是"用户态跑 32 位 TA"这条路线独有的一类坑——内核侧
一切正常、协议也正常，只是 ABI 边界上少了一个回调。


### M1.9 TA 已经能读到芯片 ID（2026-11-08）

v203（`5b1be39`，`.compat_ioctl`）刷进去之后，SPI 立刻通了，而且暴露出
TA 的**真实帧格式**：

```
[spi#1] tl=3 rl=0 n=3 tx=f0 00 00        -> 0    地址相
[spi#2] tl=5 rl=0 n=5 tx=f1 ff ff ff ff  -> 0    数据相，rxlen=0
```

注意两点，都是反直觉的：

1. **数据相 rxlen=0** —— TEE 的 SPI 服务是**原地返回**：收到的字节写回
   `tx` 缓冲区，而不是写另一个 `rx` 指针。
2. 传回来的 5 个字节是 `00 10 a0 00 25` —— 一个状态字节 + `10 a0 00 25`，
   正是我们在内核侧读到的那个芯片 ID。去掉状态字节后原地回填，TA 拿到的
   就是 `10 a0 00 25`。
3. 返回值必须是 **0**（成功）。返回字节数会被当成错误
   （`drSpiSend failed ret=0x3`）。

### 现在的卡点：TA 认不出这颗芯片

```
[gf_milan][milan_init_configuration] unsupport sensor, chip id=0x00000000
→ GF_ERROR_SENSOR_NOT_AVAILABLE (1062)
```

`DETECT_SENSOR` 那一步（`spi=2`）确实读到了 `10 a0 00 25`，但 detect 判定
失败，所以后面的 `INIT` 拿到的是缓存里的 0。

头号嫌疑是那条一直在刷的：
`[gf_memmgr_manager][gf_memmgr_load_config] not found config file, so=gf_memmgr_config.so`

这份 memmgr 配置（每个传感器的参数表）平时是 HAL 首次启动写进 TEE 存储的。
没有它，TA 可能把所有传感器都判成 unsupported。下一步要么从
`libgf_hal.so` 的 8 KiB 不透明表里把它挖出来，要么改 `ut_pf_ts_cp_open`
让 TA 走"首次自建"路径。


### M1.10 ✅ TA 把传感器配置起来了（2026-11-08）

关键发现：**TEE 的 SPI 服务是全双工、按 `txlen` 传输，收到的字节写回 `rx` 指针**
（`rxlen` 一直是 0，`desc` 里是时钟配置 `{high,low,...}`，不是长度）。

改成把收到的字节填进 `rx` 之后，`DETECT_SENSOR` 一路跑完：

```
@@ t=0 s=1 cmd=1000 -> 0   spi=578
```

578 次 SPI 事务，中间是 `f0 01 04 00 01 ff ff`（写 0x0104）、
`f0 01 02 00 01 xx xx`（反复写 0x0102 + 读 0x0106 轮询）——**原厂 TA 正在
给传感器下发配置并轮询它进入就绪状态**，而且没有再报错。

同时修掉两个 stub 语义错误：
* `ut_pf_ts_cp_size()` 返回的是**尺寸本身**，不是状态码。之前把 `0xFFFF0001`
  当成错误码返回，TA 直接把它当成了文件大小（日志里 `size=4294901761`）。
* 缺文件时返回 0 而不是负数。

### 下一个卡点：`INIT`(1001) 段错误

`DETECT` 成功后调 `INIT` 会 SEGV（exit 139，还没有 SPI 调用）。此时
`gf_memmgr_load_config` 报 `config file object corrupt. size=0` ——
memmgr 配置仍然缺失，TA 在拿到大小为 0 的对象后大概率解引用了空指针。
下一步：给 `gf_memmgr_config.so` 造一份最小合法内容，或者把
`ut_pf_ts_cp_*` 的 create 路径补完（TA 用 `flags` 区分创建/打开）。


### M1.11 ✅ DETECT + INIT + CAPTURE 全部返回 0（2026-11-08）

```
@@ t=0 s=1 cmd=1000 -> 0  spi=578     DETECT_SENSOR
@@ t=1 s=1 cmd=1001 -> 0  spi=1190    INIT（含 OTP 校验）
@@ t=0 s=1 cmd=1021 -> 0  spi=1214    CAMERA_CAPTURE
```

INIT 期间的 TA 自检日志：

```
[gf_secure_object] so(gf_otp_info.so) not exist        ← 只是缺备份，可忽略
[milan_otp_check] chicago_hu_check_cp_info] OTP cp crc check success
[milan_otp_check] chicago_hu_check_ft_info] OTP ft crc check success
```

**出厂 OTP 校准数据的 CRC 校验通过** —— 说明 TA 真的把传感器完整驱动起来了，
读到的是有效校准数据，不是空转。

修掉的两个 host 侧问题：

1. **argv 被 TA 踩坏**：TA 跑在我们的栈上，深调用链会覆盖 `argv`，循环里第二
   次 `strtoul(argv[6])` 就拿到 `0x1` → 崩在 musl `strtox`。改成**先把参数
   全部解析进堆数组**再调用 TA。
2. **不能走 musl 的 exit 路径**：TA 会注册自己的 exit/atexit 状态，正常
   `return` 会跳到 `0x2`（SIGBUS）。改成 `fflush(); _exit(0);`。

### 下一个卡点：图像数据还没取出来

`CAMERA_CAPTURE` 返回 0，但图像输出到 TA 期望的 in/out 缓冲区里，而我们传的
`desc` 是 8 个 0。需要从 `libgf_hal.so` 的 `gf_hal_common_camera_capture`
（0x20800）往上追它怎么准备 buffer，把布局照抄过来。


### M1.12 图像尺寸与采集前的"手指状态"门（2026-11-08）

TA 自己报出了图像几何：

```
[gf_algo][gf_algo_get_ee_image_info] image extension row = 44, col = 160
```

即 **160 列 × 44 行**（另有 44 行扩展），和 GF3626 侧边条状传感器的形状吻合。

采集链路现在卡在 `milan_check_finger_up`：

```
[gf_milan][milan_check_finger_up] finger is not leave, i=0
→ GF_ERROR_FINGER_NOT_LEAVE (1065)
```

**已排除的假设**（都是实测）：

| 假设 | 实验 | 结果 |
| --- | --- | --- |
| 是物理状态（手指压着） | 按住 / 拿开各跑一次 | 两种都报同样错误 → 不是物理状态 |
| 状态位在寄存器 0x0058 | 逐字节 / 逐位 spoof | 全零会让 DAC 校验先失败；其余组合仍是 finger not leave |
| 状态位在寄存器 0x00a2（24 字节 delta） | 全零 spoof | 无效 |
| 是 SPI 时钟不对（TA 按 desc 在 1 MHz / 6 MHz 间切换，我们一直用 1 MHz） | 按 `desc[0]` 反算并下发 speed_hz | 已实现，仍报同样错误 |

顺带做完的基础设施（都在 `ta_host32` 里）：

* `--spoof <addr> <val> [idx] [spoof16]`：按寄存器地址篡改返回值，用来做位级实验；
* 分配器埋点：记录 TA 的 `malloc/free/realloc`，列出 >=1 KiB 的缓冲区并落盘；
* 大读钩子：任何 `txlen > 64` 的读都会追加写入 `/root/ta/read_<addr>_<len>.bin`
  （= 传感器的原始像素流，等采集走通就能直接拿到图）。

### 下一步：把 IRQ 接进采集循环

HAL 的做法是：`camera_capture` → 若 `FINGER_NOT_LEAVE` 则**等中断**再重试。
我们目前完全没碰中断（内核驱动的 `IRQ_ENABLE` / `IRQ_WAIT` ioctl 已经可用，
compat 也修好了）。TA 连续调用时会缓存状态、不再发 SPI，所以光重试没用——
需要：

1. `ioctl(IRQ_ENABLE)` → `ioctl(IRQ_WAIT)` 等一次手指沿；
2. 再调 `camera_capture`，看状态是否翻转；
3. 若翻转，说明这条路对，把"等中断 → 采集 → 取图"做成循环。


### M1.13 中断通路验证，finger-up 门仍未过（2026-11-08）

中断这条路是通的：

```
@@ irq enable -> 0
@@ waiting for a finger edge (3/12)...
@@ irq wait -> 0            ← 用户敲击电源键时立即返回
```

空闲时 `IRQ_WAIT` 正常超时，敲击时立刻返回 —— **传感器自身的手指检测是好的**。

但每次 `camera_capture` 之后 TA 都把状态缓存住（`spi` 计数停在 1214，desc 完全不变），
再调也不重试。又排除了四个新假设：

| 假设 | 实验 | 结果 |
| --- | --- | --- |
| 之前手上一直按着，所以门是对的 | 你明确移开手、10 秒不碰 | 仍报 `finger is not leave` |
| 传感器状态被之前实验搞乱 | 先 `GOODIX_FP_IOC_RESET` 再走全流程 | 仍报同样错误 |
| 门用的值在 0x0058 / 0x00a2 | 逐位 spoof（含 0x4010 拆成 bit） | 无效 |
| 是 SPI 时钟 | 按 `desc[0]` 反算 speed_hz | 无效 |

采集返回的 desc 是 `00000000 00000000 00000001 00000000 00000001 0000000b ...`，
`desc[4]=1` 很可能就是 TA 回填的"手指状态 = 有手指"。

### 下一步优先级

1. **反 `milan_check_finger_up`**：字符串在 vaddr `0x101994`，字面量池在
   `0x25dec`，函数体在它之前（Thumb 代码）。TA 是 ARM/Thumb 混编，用
   `-M force-thumb` 分段读，找 `tst/ands` + 分支即可定位它比较哪一位。
2. **看 HAL 的采集前序**：`libgoodixhwfingerprint.so`（20 KB）和
   `fingerprint.goodix.so`（11 KB）都很小，采集前可能还有
   "取手指状态 / load_instance_data" 之类的调用，是我们漏掉的。
3. **试 test 族的 raw data 命令**：`hal_test_driver_cmd@0x294ac` 是通用入口，
   从它拿到 `CMD_TEST_BMP_DATA` / `GF_CMD_TEST_RAW_DATA` 的 id 后直接调，
   这类命令通常不等 finger-up。


### M1.14 test 族与 desc 缓冲区实验（2026-11-08）

按计划试了第 3 条（test 族绕过 finger-up 门），结果与结论：

| 实验 | 结果 |
| --- | --- |
| test 族 sweep（id 1055–1070，`desc[2]=2`） | 全部落到 `gf_ta_invoke_user_cmd_entry_point` → `GF_ERROR_BAD_PARAMS`。**入口族的选择是有状态的**，先 detect+init 之后再打同样的 desc 会走 user 而不是 test |
| `probe <id> 0 2` 逐 id（1040–1070，不 prime） | 全部返回同一个 init 期错误，命令本体没有可区分的反馈 |
| 给 `desc[1]` 传 64 KB 缓冲区 + `desc[3]=长度` | **INIT 直接卡死**（命令 1001 不再返回）→ `desc[1]` 不是简单的 in/out 缓冲指针 |
| 只传 `desc[1]`、长度置 0 | 同样卡死 |

静态分析得到的有用坐标（留作下一步）：

* `hal_test_driver_cmd@0x294ac` → 内部用 `w0=0x423(1059)` 调 `gf_hal_test_invoke_command`
  —— test 族的 id 落在 1059 附近（和正常族 1000/1001/1016/1021 是两套编号）。
* `dump_cali_and_bmp@0x3ab08`（856 B）是 HAL 里真正的"出图"路径：
  `gf_get_sensor_width/height` → `gf_dump_raw_data` → `gf_dump_encoder_fopen/fwrite/fclose`。
  **但它是 aarch64 代码，我们的 host 是 32 位，调不了**。这条只说明了 HAL
  出图的组成，不能直接用。

### 结论：下一步只能反 `milan_check_finger_up`

所有"绕过"的尝试（物理状态、寄存器 spoof、时钟、重置、test 族、desc 缓冲）
都排除了。门在 TA 内部，必须读懂它：

* 错误字符串 `"finger is not leave"` @ vaddr `0x101994`；
* 字面量池引用 @ `0x25dec`，函数体在其之前（**Thumb 代码**，需 `-M force-thumb`）；
* 目标：找到它读的寄存器地址 + 它比较的那一位。

拿到那一位之后，在 `ta_host32` 里用现成的 `--spoof` 精确篡改即可，工具已经就绪。


### M1.15 反汇编进展：定位到日志封装，剩下一个对齐问题（2026-11-08）

用 capstone 之前先做了件更省事的事：在 `ut_pf_log_msg` stub 里读
`__builtin_return_address(0)`，直接拿到"谁在打这条日志"：

```
[ta ra=0xa2f8] [GF_TA][E][gf_milan][milan_check_finger_up] finger is not leave, i=0
```

`0xa2f8` 在 TA 的日志封装里（`0xa29c` 起，负责 push 变参 + `vsnprintf`）。
所以判断代码是**调用这个封装的那一层**。用字节模式扫描（`ta_refs3.py` 里那套）
找到该封装有 **171 个 ARM 调用点**。

同时扫"哪段代码里出现立即数 `0x58`"（`ta_reg58.py`），得到 17 个 Thumb 候选，
其中 `0xb05c`、`0xbdb0`、`0xc4f8`、`0xd28c`、`0xddd8`、`0x11c14` 落在代码段里，
且集中在日志封装附近——**`milan_check_finger_up` 大概率就在 0xb000-0xe000 这一带**。

### 卡点：objdump 在 Thumb-2 上会解同步

`arm-none-eabi-objdump -M force-thumb --start-address=0xbdb0` 出来的结果明显错位
（`movs r0,#0x58` 后面跟着半个 32 位指令）。Thumb-2 是变长编码，**必须从真正的
指令边界开始线性扫描**，否则一路错到底。这就是为什么前面几次"盲读"都读到垃圾。

**下一步（工具已就绪）**：用 capstone 写递归下降（从已知边界 `0xa29c` 开始，
或从每个 `push {..., lr}` 前导开始），重建 0xa000-0xe000 这一段正确的指令流，
再找 `cmp/tst` + 分支。这一步纯离线，不需要设备。

### capstone 安装方式（家目录只读）

```sh
pip install --target=/home/kily/project/mt6893/work/env/pylibs capstone
PYTHONPATH=/home/kily/project/mt6893/work/env/pylibs python3 ...
```


### M1.16 ✅ 找到 `milan_check_finger_up` 的确切代码（2026-11-08）

静态分析在混合指令集上一直打滑，于是换成**动态 hook**：TA 里所有日志都走
`0xa29c` 那个封装，我们把自己的 shim 写进它的第一条指令（ARM `b`，±32 MB），
shim 里读 `__builtin_return_address(0)` —— 得到的就是"谁在打这条日志"。

要打这个补丁，host 必须链接在 TA 的 ±32 MB 内，所以 `-Ttext` 从 `0x10000000`
改成了 `0x800000`（TA 占 `0x8000–0x33c918`，不冲突）。

然后两次运行按序号对齐日志和调用者：

```
log#55 caller=0x25d11 | [gf_milan][milan_check_finger_up] finger is not leave, i=0
```

`0x25d11` 是 Thumb 返回地址（`0x25d10 | 1`），落在 **Thumb 代码**里。手读十六进制：

```
25cd4: 33 59     ldr  r3, [r6, #0xfc]     ; 取一个运行时值
25cd6: 9d 42     cmp  r5, r3
25cd8: 10 d3     bcc  0x25cfc              ; r5 < r3 → 判为"手指没离开"
25cda: 04 20     movs r0, #4               ; 否则只打一条 level-4 调试日志
25cdc: 42 49     ldr  r1, [pc, #0x108]
25cde: 42 46     mov  r2, r8
25ce0: d0 47     blx  r10                  ; r10 = 日志函数
25ce2: 08 b0     add  sp, #8
...
25cee: f0 bd     pop  {r4, r5, r6, r7, pc}
```

**这是数值比较，不是标志位** —— `r5` 和 `*(r6+0xfc)` 都是运行时算出来的量
（大概率是图像/基线的某种统计值）。这解释了为什么按寄存器地址 spoof 全部无效：
门读的不是某个状态位，而是它自己算出来的一个数。

把 `0x25cd8` 的 `bcc` 改成 `nop`（`GF_PATCH_GATE=1`）能跳过这条分支，但返回码
仍然由前面算好的值决定（`r0 = r9`），所以表现不变——说明**判定发生在更早的
图像/统计计算里**。

### 下一步

1. 在 `0x25cd6` 处把 `r5` 和 `r3` 的值打出来（把 `cmp` 位置换成一次
   `bl` 到 shim 需要 4 字节，可以先把 `0x25cd4` 的两条 2 字节指令合并腾挪，
   或者改 hook `gf_milan_reg_access` 并解析它返回的结构）。
2. 更省事的办法：hook **图像读取函数**，把它读到的缓冲区和基线都 dump 出来，
   直接看 TA 算出来的 delta 是多少——如果 delta 极大（比如 0x40/0x20 那种
   规律性数据），说明**传感器返回的"图像"本来就不对**，问题在采集配置，
   不在判断逻辑。


### M1.17 门里的值：是重试计数，不是手指状态（2026-11-08）

让 log shim 顺带读 `r5/r6` 和 `[r6+0xfc]`，失败那次调用拿到：

```
log#55 caller=0x25d11 r5=0x0 r6=0x2ab324 [r6+fc]=0xa
```

把 `0x25c40-0x25cd8` 的十六进制手工解出来：

```
25cca: movs r5, #0xc        ; r5 = 12
25ccc: cmp  r7, r3
25cce: blt  0x25cd2
25cd0: b    0x25dc6
25cd2: movs r4, #0xfc
25cd4: ldr  r3, [r6, r4]    ; r3 = 10  (最大重试次数)
25cd6: cmp  r5, r3
25cd8: bcc  0x25cfc         ; 12 < 10? 不成立
25cda: movs r0, #4          ; 走这里 → 只打 level-4 调试日志
```

也就是说这条 `bcc` 按静态值**不该跳**（12 >= 10），但运行时它跳了 —— 说明
`r5` 在那个时刻并不是 12。**我们的 shim 里用内联汇编读 r5/r6 不可靠**
（编译器在调用前已经把 r5 挪用了，`-O2` 下寄存器分配会覆盖）。

### 结论：静态读到这里已经到极限，下一步该上模拟器

再往下读需要**运行时的精确寄存器值**，而"改代码 + 猜寄存器"这种方式已经不可靠。
正确做法是用 **qemu-arm**（主机上已安装）跑我们的 32 位 host：

1. 在设备上录一段 SPI 事务序列（我们的 stub 已经能全量打印）；
2. 加一个 `GF_REPLAY=<file>` 模式，让 stub 回放这段数据，不需要真硬件；
3. `qemu-arm -d in_asm,cpu` 跑，得到**每条指令 + 全部寄存器**的完整轨迹；
4. 直接看 `0x25cd4-0x25cd8` 那两条指令的输入输出。

这条路能一次看清 `r5`/`r6`/`[r6+0xfc]` 的真实值，也能顺带验证整个 TA 流程
（不依赖手机、可以反复跑、可以下断点）。成本是写一个回放模式（约 50 行），
收益是把"盲猜"变成"看"。


### M1.18 ✅ 离线 qemu 回放环境建成（2026-11-08）

这是本轮最有价值的产出：**整个 TA 流程现在可以在主机上离线跑，不需要手机**。

三件事凑齐：

1. `GF_RECORD=<file>` / `GF_REPLAY=<file>`：SPI stub 能录下/回放每次事务
   （格式 `n <txhex> <rxhex>`，一次 `cap` 全流程 1214 行）。
2. `qemu-arm -B 0x10000000`：主机的 `vm.mmap_min_addr = 65536`，TA 的
   `0x8000` 基址直接 `mmap(MAP_FIXED)` 会 `EPERM`；用 `-B` 给 guest 加一个
   基址偏移就绕过去了。
3. `-d in_asm,cpu -one-insn-per-tb -dfilter <start>+<size> -D log`：
   QEMU 10 的 `-dfilter` 语法是 `0x25c80+0xc0`（`-singlestep` 已改名
   `-one-insn-per-tb`）。

**验证**：回放跑出来的结果和真机逐字节一致（同样 `spi=1214`、同样 desc）。

### 门处的权威寄存器值（qemu 实测）

```
0x25cd4: ldr r3, [r6, r4]   r6=0x2ab324 r4=0xfc → r3 = 0xa
0x25cd6: cmp r5, r3         r5 = 0            → N=1 (0 < 10)
0x25cd8: blo #0x25cfc       跳转成立
```

所以运行时 `r5 = 0`（就是日志里的 `i=0`），阈值 `[r6+0xfc] = 10`。
我此前从十六进制手工推的 `r5 = 12` 是错的——那是同一条指令在另一条路径上的
常量，静态读分支果然不可靠，**qemu 一跑就清楚了**。

### 下一步（现在很快，因为不用手机）

1. 在 qemu 里给 `0x25cd4` 附近的读值下断点式过滤，找到"传感器读回来的
   `0x58` 值"被哪条指令消费；
2. 用 qemu **暴力试值**：把 `0x58` 的返回值在 0..0xffff 里扫，看哪个值让
   `finger is not leave` 消失（每次 1–2 秒，可以跑几百次）；
3. 找到之后就知道真正的"手指离开"编码，再决定是改 stub 还是改采集配置。


### M1.19 门是一个"等手指离开"的轮询循环（2026-11-08）

qemu 的 `-one-insn-per-tb -d in_asm` 是**按翻译顺序**记录，每个指令只记一次，
所以要在整份日志里定位（不是只看开头）。按执行顺序还原出来：

```
25d84: mov  r2, r8
25d86: adds r3, r5, #0
25d88: movs r0, #2
25d8a: blx  sl              ; 打日志（level 2）
25d8c: b    0x25cd2         ; ← 回到门
25cd2: movs r4, #0xfc
25cd4: ldr  r3, [r6, r4]    ; r3 = [全局+0xfc] = 10
25cd6: cmp  r5, r3          ; r5 = i
25cd8: blo  0x25cfc         ; i < 10 → 判"手指没离开"
```

**结论**：这不是"读一次状态位"，而是一个**最多轮询 10 次等手指离开的循环**，
`r5` 就是轮询计数 `i`。失败路径打完日志后 `ldr r5,[pc]` 会覆盖 r5 并返回——
所以每次 `camera_capture` 都只做一次轮询，`i` 永远停在 0，永远失败。

也就是说：**`milan_check_finger_up` 期待的调用者是"被我方反复调用"的**，
而我们每次都在第一轮就放弃（TA 把结果缓存了，见 M1.9）。

### 下一步

1. 找 `i` 的来源：`[r6+0xfc]` 是上限 10，当前值大概率在 `[r6+0xf8]` 附近；
   在 qemu 里直接看 `r6+0xf0..0x108` 的内存，或在调用前把当前值设成 10 试跳。
2. 若 `i` 是全局：**在采集前把它写成 10**，门就一次性通过——这是最小改动，
   而且不改 TA 代码（只写数据段）。
3. 若 `i` 是参数：找调用点，看它从哪取这个值（大概率是 HAL 侧的重试计数）。


### M1.20 计数器全局的实测状态（2026-11-08，注意此节为半成品）

- `0x2ab324` 是 r6 指向的全局结构；**加载后**该处 16 字节全 0，说明
  `[+0xfc]=10` 是**初始化阶段写进去的**，不是静态初值。
- `i`（r5）在门处恒为 0：TA 每次 `camera_capture` 只做一轮轮询就返回，
  而它期待的"重试"由调用方驱动。
- 我加了 `GF_POKE=<addr>=<val>` / `GF_DUMP=<addr>:<len>` 两个调试开关，
  但**还没跑通"在门处 dump 计数器所在偏移"**（env 语法没来得及收敛，
  加入的代码已回滚到提交版本 `97b491a`，避免污染工具）。

### 下一步（明确到可以直接执行）

1. 在 `cap` 模式的每条命令**之后**调用一次 `GF_DUMP=0x2ab324:0x40`
   （注意语法统一成 `<addr>:<len>`），看清 `+0xf0..+0x110` 里哪个偏移是"当前 i"；
2. 用 `GF_POKE` 把那个偏移写成 10（或 11），再看 `finger is not leave` 是否消失；
3. 若消失 → 最小改动完成（只写数据段，不动 TA 代码）；
   若仍是 0 → `i` 是参数，回到调用点找它的来源。


### M1.21 test/dump 族扫不到图像数据（2026-11-08）

按"先弄一张图"的思路，在真机上把 test 族（`desc[2]=2`）和 dump 族（`desc[2]=3`）
的 id 1055–1065 全扫了一遍，用大读钩子（`txlen > 64` 自动落盘）监控：

```
sub=2 id=1055..1065  → 无
sub=3 id=1055..1065  → 无
```

**结论**：这些命令在当前状态下都不会让 TA 去读图像。原因是同一个：入口族的
选择是**有状态**的，而我们没有走到"允许取 raw data"的状态——也就是说，
**"绕过门取图"和"过门"需要的是同一份理解**，前者并不更快。

### 目前把这条路走完所需的两件事

1. **搞清 `i` 的来源**（是全局还是参数）——决定能不能一行 poke 过门；
2. 若 `i` 是 HAL 侧的重试计数，那就在 host 里实现同样的重试驱动
   （每次 `camera_capture` 之间调用一次 detect/状态查询，让计数推进）。

这两件都在现有工具（qemu 回放 + 任意内存读写 + 任意代码打补丁）的能力范围内，
但都需要**新的会话继续**，因为本轮上下文已经耗尽。


### M1.22 计数器的来源：捕获阶段才算出来的（2026-11-08）

用 `GF_DUMP2` 在 INIT 之后、采集之前 dump `r6+0xc0..0x120`（r6=0x2ab324）：

```
dump 0x2ab3e4..0x2ab444: 全 0
```

而门处 `[r6+0xfc]`（= 0x2ab420）读到 **10** —— 说明这个上限是**采集函数内部
现算并写进结构的**，不是初始化留下的。

qemu 轨迹还原出的控制流：

```
25cd2: movs r4, #0xfc
25cd4: ldr  r3, [r6, r4]     ; 上限（采集时算出来 = 10）
25cd6: cmp  r5, r3           ; r5 = i = 0
25cd8: blo  0x25cfc          ; i < 上限 → 进入"未离开"分支
       ...（0x25d84-0x25d8c 是一条循环尾巴，b 回 0x25cd2）
25cfc: 打日志 "finger is not leave, i=0"
25d0e: blx  r4               ; 再调一次（取错误对象）
25d10: ldr  r5, [pc]         ; r5 被覆盖 → 直接返回
```

**`r5`（i）在整个循环里从不自增**，且失败分支会覆盖 r5 后返回。所以：

> TA 期待的"重试计数"是由**调用方**推进的，TA 每次只做一轮判断。

这正是 HAL 的职责——HAL 在 `camera_capture` 之间做别的调用（取状态/延时），
让 i 递增到上限。我们的 host 目前是"连打三次同一个命令"，TA 直接缓存返回。

### 下一步（已收敛成一个明确的实验）

在 host 里实现 HAL 式的重试驱动：

```
for i in 0..10:
    fp_invoke(camera_capture)
    fp_invoke(<某个能推进状态的命令>, 例如 detect 或状态查询)
    看 desc / SPI 计数是否随 i 变化
```

如果 i 仍然是 0，说明计数在 TA 的别处（比如 `[r6+0xf8]`），那就用
`GF_POKE2=<off>=<val>` 直接写；工具已经就绪。


### M1.23 真机试了三条新假设，都否掉了（2026-11-08）

| 假设 | 实验 | 结果 |
| --- | --- | --- |
| 重试计数 `i` 由 `desc[1]` 传入 | `GF_D1=1/5/10` 采集 | TA 直接覆盖 `desc[1]`，`i` 仍为 0 → **`desc[1]` 是输出字段** |
| 采集后用 detect "推一下"状态再采 | `retry` 模式：capture → detect → capture… | 第二次 detect 直接卡住（`[cp] open` 之后无输出），**流程不可重入** |
| 计数器在 TA 全局里，poke 即可 | `GF_DUMP2` dump `r6+0xc0..0x120` | INIT 后全 0，采集时才被写 → 不是可以提前摆好的静态值 |

新增工具（已提交）：`retry <n>` 模式、`GF_D1` / `GF_DUMP2` / `GF_POKE2` 开关。

### 现在必须换思路

三次"从调用方推进 i"的尝试都失败了，说明 **`i` 不是外部输入，而是 TA 内部
在采集函数里自己数出来的**——那它数的对象就是"传感器报告手指状态的次数"。
也就是说：**传感器一直报告"手指在"，TA 数到上限 10 就放弃**。

所以真正的修复点回到采集配置：要么让传感器正确报告"无手指"，要么让
"读状态"这条路径返回符合 TA 期待的值。这需要**从上一次成功的 Android 行为
反推**（比如对比 stock 内核在开机时对 0x0058/0x00a2 的初值处理），而不是继续
在 TA 门口打补丁。

**建议的新会话起点**：
1. 抓 stock 内核 `gf_spi_tee.c` 初始化序列里对这两个寄存器的读写时序（源码里有）；
2. 对照我们的 1214 条录像，看哪一步的写值/顺序不同；
3. 差异点大概率就是"传感器为什么一直说手指在"。


### M1.24 ⭐ 找到真根因：SPI CS 时序（2026-11-08，v204）

从原厂 `gf_spi_tee.c` 里那段 `mt_chip_conf` 对照出来的**唯一实质差异**：

```c
const struct mt_chip_conf spi_ctrdata = {
    .setuptime = 10, .holdtime = 10, .cs_idletime = 10,
    .high_time = 50, .low_time = 50,   /* 1 MHz */
    .cpol = CPOL_0, .cpha = CPHA_0, ...
};
```

`setuptime/holdtime/cs_idletime = 10` 是**10 个 SPI 时钟**（1 MHz 下 10 µs），
而我们从来没在 DTS 里设过，主线的
`spi-cs-setup-delay-ns / spi-cs-hold-delay-ns / spi-cs-inactive-delay-ns`
默认是 0。

补上（v204）之后，寄存器窗口立刻变成**真实数据**：

| 地址 | v203（之前） | v204（现在） |
| --- | --- | --- |
| `0x0000` | `10 a0 00 25` | `10 a0 00 25`（芯片 ID，一直对） |
| `0x0058` | `40 10` | `02 af a1 01 00 00 14` |
| `0x00a2` | `40 10 20 10 40 10 20 …`（规律垃圾） | `01 89 01 66 01 4d 01 3f 01 4c 01 80`（**12 位 ADC 采样，真实像素**） |
| `0x0106` | `00 00` | `00 c2` |

也就是说：**此前所有"传感器一直说手指在"的判断，都是建立在采样错位的数据上**。
`40 10 20` 那个规律序列不是传感器输出，是 CS 建立时间不足导致的首字节采样错位。

顺带澄清一个之前的误判：TA 往 `0x0020` 写的那 270 字节里本来就含
`00 86 40 10 20 …`，所以读回同样的模式一度让我们以为"读到了自己写的配置"。

**当前状态**：数据是真的了，但 `milan_check_finger_up` 仍然报 `i=0`。
下一步应该重新在**正确数据**上做一次运行时观察（此前所有 spoof 实验都是在
坏数据上做的，结论可能需要重测）。


### M1.25 在真实数据上重新定位判断点（2026-11-08）

用 v204 重新录了一份 SPI（`spi2.rec`，1214 条），qemu 回放与真机一致。
然后在判断分支上下断点式过滤，拿到权威寄存器：

```
25b98: ldrh r3, [r5]      ; r5 = 407ff408 (栈上表) → r3 = 0x08
25b9a: subs r0, r1, r7    ; r0 = 0xb3 - 0x13 = 0xa0
25b9c: cmp  r0, r3        ; 0xa0 vs 0x08
25b9e: blt  0x25ba2       ; 不成立 → 落 0x25ba0
25ba0: b    0x25d8e       ; → i=0 → 回到门 → i<10 → 报"手指没离开"
```

所以真正的判断在 `0x25b9c`，输入是 `r1`、`r7`、`r3`（从栈上的表取 u16）。
数值上 `r1=0xb3(179)`、`r7=0x13(19)`、`r3=8` —— 看起来更像**索引/步长范围**而不是
像素阈值（我上一节"M1.24"里说的"像素差值 vs 阈值"可能过于武断，这里更正）。

### 下一轮的第一步

在 qemu 里看 `r1`/`r7`/`r3` 的来源（它们分别由哪条指令写入、来自哪个数组），
再判断这个比较的真实语义。工具全部就绪：`replay.sh` + `-dfilter 0x25b80+0x28`。


### M1.26 继续与原厂内核逐项对照（2026-11-08）

把原厂驱动里所有"可能影响传输"的设定过了一遍：

| 原厂设定 | 我们 | 结论 |
| --- | --- | --- |
| `setuptime/holdtime/cs_idletime = 10` | 0 → **v204 已补成 10 µs** | ✅ 真根因（M1.24） |
| `com_mod = FIFO_TRANSFER`（永远 FIFO） | 主线 `len>32` 且 4 字节对齐时走 **DMA** | ❌ 实测排除：同窗口 DMA 读 96 B 与 FIFO 分 4 次读 24 B **逐字节相同** |
| `spi->max_speed_hz = 1 MHz`，TA 按 desc 切换 | 已按 `desc[0]` 反算下发（925 kHz / 5.5 MHz） | ✅ 一致 |
| `CPOL=0 / CPHA=0 / MSB / LENDIAN` | `SPI_MODE_0`、8 bit | ✅ 一致 |
| `gf_spi_clk_enable()` 包住整段操作（`mt_spi_enable_master_clk`） | 主线按传输粒度 runtime PM | 传输数据正确，判为无关 |
| DT `power-type = <1>` | — | 原厂驱动里**根本没读这个属性**，忽略 |
| `gf_miso_gpio_cfg`（MISO 上下拉切换） | — | 只在 `SUPPORT_REE_OSWEGO` 下编译，与我们无关 |
| `gf_bypass_flash_gpio_cfg` | — | 原厂里是空函数（TODO） |

原厂驱动里唯一自己写的寄存器是 fw 升级路径（`0x5081/0x4180/0x4010/0x40B0/0x404B`），
正常初始化全在 TA 里 —— 所以"从内核补功能"这条路已经走到底了：
**总线层面已经完全对齐**。

### 剩下的只有一个问题

`0x25b9c` 那个比较的输入（`r1=0xb3`、`r7=0x13`、`r3=8`）还没搞清楚来源。
下一步在 qemu 里看它们由谁写入、取自哪个数组——纯离线，`replay.sh` 就绪。

### M2 原始计划（留档）

1. `ut_pf_spi_send_and_receive` → `ioctl(/dev/goodix_fp)`。注意参数语义
   （buffer/长度/读回，需从调用点确认）。
2. 从 `libgf_ca.so` 反出 `ut_pf_fp_invoke_command` 的 buffer 布局，先打通
   **CHIP_INFO / GET_DEV_INFO** 这类无副作用命令。

**验证标准**：TA 报出的 chip id / fw_version 和内核侧读到的 `10 a0 00 25` 自洽。

### M3 图像 + 录入/匹配

1. `GF_CMD_CAMERA_CAPTURE` 抓一帧，降成 BMP/PNG 看画质——**这一步就能验证
   "画质差的坑"是不是被绕过去了**。
2. 模板存储接文件，跑 `enroll` / `authenticate`。
3. 收口：CLI 先跑通，再包成 libfprint 驱动（照 xaga 的 `fpc1540ta` 驱动写）。

### M4（可选）libfprint 驱动

xaga 的 libfprint 驱动协议（`new_set/use_set/save_set/identify_sets`）可以
直接照抄结构，只换 socket 协议和 TA 调用。

---

## 5. 风险

| 风险 | 说明 | 缓解 |
| --- | --- | --- |
| 32 位兼容层 | 设备上从没跑过 AArch32 程序 | `CONFIG_COMPAT=y` 已开；先跑一个 `arm-linux-musleabihf` hello world 验证 |
| `mmap_min_addr` | TA 基址 0x8000 | root + `CAP_SYS_RAWIO`；或临时 `sysctl vm.mmap_min_addr=4096` |
| SPI 语义 | `ut_pf_spi_send_and_receive` 的参数顺序未知 | 反 `gf_spi_read` 的调用点（TA 里有字符串和清晰的调用序列）|
| 加密/Keymaster | 模板用 HMAC key 包装，换 key 就丢模板 | 和 xaga 一样用确定性 key，接受"只在我们的 host 内可移植" |
| TA 依赖 TEE 特权 | 某些路径可能假设自己在安全世界（读 OTP、写 efuse）| 先跑，撞到再 stub |

---

## 6. 现在就能做的第一件事

```sh
# 工具链（已下载）
work/env/arm-linux-musleabihf-cross/bin/arm-linux-musleabihf-gcc -static \
    -Ttext=0x10000000 -o ta_host32 *.c

# 设备上确认 32 位能跑（一条命令就能验证整条前置条件）
scp hello32 root@192.168.0.108:/tmp/ && ssh root@192.168.0.108 /tmp/hello32
```

hello world 一过，M1 就是纯写代码的事。

---

## 2026-11-08 晚（第二轮）：门、ABI、token、存储

新一轮把上一份交接里"M1.25 下一轮第一步"的问题解掉了，并且顺着推进到了
**enroll 命令**（录入），现在只剩一个明确的卡点。全部实验都能离线复现，
不需要手机。

### M1.27 ✅ `milan_check_finger_up` 到底在比什么

上一轮把注意力放在 `0x25b9c` 的 `cmp r0, r3`，但没读出三个输入的语义。
这一轮用 qemu 的**逐指令 + 全寄存器** trace 一次看清了。先记住函数边界：

| 东西 | 地址 |
| --- | --- |
| `milan_check_finger_up` 入口 | `0x25b18`（Thumb） |
| 12 个区域逐个比较 | `0x25b84` – `0x25cce` |
| 某个区域失败 | 各自的尾巴 `0x25d8e`/`0x25d92`/…/`0x25dc8` |
| 汇总日志（fdt / down / area） | `0x25d5e` |
| 共享门 `r5 < [r6+0xfc]` | `0x25cd2` |
| 错误返回 `GF_ERROR_FINGER_NOT_LEAVE` | `0x25d16` 起，`r5=0x429=1065` |

真实语义（寄存器实测）：

```
r6      = 0x2ab324                     ; TA 全局
config  = [r6+0x3c]                    ; = 0x139d48，传感器配置结构
r4      = ldrh[config+0x420]           ; = 0x20
delta_up= ldrb[config+0x634 + (0x82 - ldrh[config+0x630])]   ; = 0x13
area_i  = 0x84 - r4 + 2*i              (i = 0..11)
down_i  = ldrb[config+0x424 + area_i]   ; = 0xb3（i=0）
avg_i   = fdt_average_data[i]           ; 由 0x256d5 "get FDT average data" 填在栈上
```

比较：**失败**（`b` 到 `0x25d8e`+）的条件是
`down_i - delta_up >= avg_i`（`blt` 不成立）。也就是
`0xa0 >= 0x08` → 报 `area[0] is finger down`。

日志原文（设备上抓的）：

```
[gf_milan][milan_check_finger_up] delta_up=0x13
[gf_milan][milan_check_finger_up] fdt_average_data[0]=0x0008
[gf_milan][milan_check_finger_up] g_milan_chip_handle.config->fdt_down.data[100]=0x00B3
[gf_milan][milan_check_finger_up] area[0] is finger down
[gf_milan][milan_check_finger_up] finger is not leave, i=0
```

`i` 是**区域的编号**（0..11，全部通过时 `r5=0xc`），不是重试计数——上一轮
"i 由调用方推进"的推论可以作废。

### M1.28 ✅ 过门的正确补丁：`GF_PATCH_GATE=2`

上一轮 patch 的是共享门 `0x25cd8`，但那不是失败点；真正把控制流带走的是
**12 个区域各自的失败分支**：

```
0x25ba0 0x25bbc 0x25bd8 0x25bf4 0x25c10 0x25c2c
0x25c48 0x25c64 0x25c80 0x25c9c 0x25cb8 0x25cd0
```

把它们 NOP 掉（全是 2 字节 `b`），函数就会一路走到"12 区域全部通过"的
`r5=0xc` 分支。`GF_PATCH_GATE=2` 就是这个（`=1` 仍是老的 `0x25cd8`）。
设备上验证：门的日志消失，流程继续走到 `gf_hw_set_mode`。

### M1.29 ✅ qemu 逐指令 trace（本轮最有用的小工具）

```
GF_REPLAY=spi2.rec qemu-arm -B 0x10000000 -one-insn-per-tb \
    -d exec,cpu -dfilter 0x25b18+0x2b0 -D qemu.log ./ta_host32 goodix.ta cap 1
PYTHONPATH=work/env/pylibs python3 ta_xrun.py qemu.log
```

`-d exec,cpu` 是**每次执行**都 dump 一份寄存器（`in_asm` 是翻译期、每条只记
一次，容易误读）。配合 `-dfilter` 只保留关心的地址段，输出是"指令 + 全寄存器"
的执行序列，`ta_xrun.py` 再把每条指令和寄存器值拼成一行。上一轮靠猜的
`r5/r6/r1/r7/r3`，这一轮 3 条命令就出来了。

> 注意：`-D` 的输出**不要写 `/tmp`**：每次 bash 调用是独立的 mount namespace，
> 下一条命令看不到。写到 `port/fingerprint/ta32/work/` 下。

### M1.30 ⭐ 命令 ABI：带 payload 的调用方式

之前只会用 `desc[8] + *cmd` 调"无 payload"的命令。真正的形态是
`ut_pf_fp_invoke_command(desc, req, len)`：

```
req[0]  u32  cmd_id            (输入)
req[4]  u32  in_len            (输入)
req[8]  u32  out_cap           (输入) → 也被 TA 当作 out_len 回写
req[12] u32  status            (0)
req[16] ...  payload
len     = payload 长度（TA 内部会 len-16 再算指针）
```

`desc` 仍然是 `{0, 0, sub, 0, ...}`（`sub=1` 走正常族）。`req` 必须**常驻**：
TA 会长期持有这个缓冲区里的指针（在 `plan` 里把同一个 `req` 释放掉会直接
SIGILL）。

新增模式：

```sh
./ta_host32 goodix.ta req  1007 enroll.bin 0 1     # 单条带 payload
./ta_host32 goodix.ta plan plan.txt                # 一串命令，同一进程
# plan.txt 每行: <cmd> <payload 文件|-> [out_cap] [sub]
```

这也是为什么以前 `DETECT`/`INIT` 必须分两次进程就做不成事：TA 的初始化状态
在同一进程内累积。

### M1.31 ✅ 真 HMAC-SHA256，enroll token 通过了

`gf_ta_enroll` 的流程（`0x171d0` 起）：

```
memcpy(sp+0x1c, struct+0x6e, 0x45)      ; token 块
memset(sp+0x41, 0, 0x20)
token_auth(sp+0x1c)                     ; 0xa830 → 0xa538
memcmp(sp+0x41, struct+0x93, 0x20)      ; 不相等 → "token authenticate failed"
```

`token_auth` 里走的是 memcrypto 导入：

```
ut_pf_cp_open(&ctx, 3, 0x3003)          ; TEE_ALG_HMAC_SHA256
ut_pf_cp_mc_start(ctx, key, 0x20)       ; key 来自 ut_pf_km_get_hmac_key
ut_pf_cp_mc_update(ctx, msg, 0x25)
ut_pf_cp_mc_finish(ctx, out, &len)
```

所以：

* `ut_pf_cp_mc_*` 必须是真的 HMAC-SHA256（原来返回固定 32 字节 `0x11`，
  永远过不了）；
* `ut_pf_km_get_hmac_key` 的真实签名是 `(key_out, &key_len)`，第二个参数是
  **长度指针**（调用点 `0xa55c`），不是长度；
* token 块 = `msg[0x25] || HMAC(msg)[0x20]`，放在 enroll payload 的 `+0x6e`。

自检：`./ta_host32 goodix.ta hmactest` 打印 RFC 4231 test 1 的结果。

`mk_enroll.py` 生成这份 180 字节的 payload（含 token）。

### M1.32 ✅ 受信存储的 ABI 是反的（原来一直是错的）

TA 的封装（`0x9c0c` / `0x9d18` / …）**直接把 r0 当句柄/大小用**：

```
handle = ut_pf_ts_cp_open(name, flags, &slot);   0 → GF_ERROR_OPEN_SECURE_OBJECT_FAILED (1033)
size   = ut_pf_ts_cp_size(handle, &size_slot);   0 → GF_ERROR_READ_SECURE_OBJECT_FAILED (1034)
nread  = ut_pf_ts_cp_read(handle, buf, len);
nwrite = ut_pf_ts_cp_write(handle, buf, len);
```

之前实现成"成功返回 0"，于是每个"对象不存在"的探测都被当成"大小为 0 的对象"，
后面一堆 `size=4294901761` 的怪现象都是这个。改完之后 `so(...) not exist`
路径才是正常的。

### M1.33 ✅ pre_enroll 的 challenge，以及新的卡点

`hal_milan_an_series_pre_enroll`（和 `gf_hal_common_pre_enroll`）用的是
**cmd 1006**，112 字节缓冲。TA 会把 `ut_pf_cp_rd_random` 的结果写到
`buffer+0x68`（我们 stub 填 `0x42`，所以是 `4242424242424242`）。

enroll（cmd **1007**）的 token 里 `msg[0x3]` 是版本、`msg[4..7]` 是 challenge：

| token 内容 | 结果 |
| --- | --- |
| `msg[0..3]=0x42424242` | `invalid hat version code` (1055) |
| `msg[0..3]=0`、challenge 放 `+4` | **token + challenge 都过** |
| challenge 放 `+12/+16/…` | `invalid challenge detected` (1056) |
| 没有先跑 1006 | `invalid challenge detected` |

顺序必须是 `DETECT(1000) → INIT(1001) → PRE_ENROLL(1006) → ENROLL(1007)`，
而且**同一进程**。

### 现在的唯一卡点：fingerlist 的 `max_count = 0`

```
[gf_fingerlist][gf_fingerlist_is_enroll_allowed] up to the limit, used_count:0, max_count:0
→ GF_ERROR_REACH_FINGERS_UPLIMIT (1005)
```

`max_count` 是 `[0x2b9cc8 + 0x104]`（= `0x2b9dcc`）。实测：

* DETECT、INIT 之后这个字段**仍然是 0**（不是初始化阶段写进去的）；
* 更准确地说：先把它写成 `0xA5A5A5A5`，跑完 DETECT 再看，它又变回 **0**
  —— 说明 DETECT/INIT 会清/写这个字段，只是写进去的值来自一份"全零的配置"；
* 用 `GF_POKE=0x2b9dcc=5` 手工写 5，enroll 立刻越过这个检查，继续往下走
  （又发了 273 字节配置写），然后 `SIGILL` 跳到宿主 musl 的代码里——因为
  `finger_0_0.so` 不存在 → fingerlist 为 NULL →
  `[gf_ta] fingerlist is null set mode key`，TA 在这种状态下走了一条我们没
  实现的异常/回调路径。

所以下一步很清楚：

1. 弄清 `max_count` 该由谁写。现在怀疑和 `gf_memmgr_config.so` 同源
   （配置全零）。可以静态找 `0x2b9cc8+0x104` 的写入点，或者在
   `gf_fingerlist_init` 的调用链上找参数来源。
2. 让 fingerlist 变成"合法空表"。TA 的第一次录入本来应该自己创建
   `finger_0_0.so`，但需要 `max_count > 0` 且列表指针非空。可能需要：
   * 先跑一次会创建列表的命令（`gf_fingerlist_add` 在 post_enroll 之后，
     鸡生蛋问题）；
   * 或者把 `gf_so_save_persistent_object` / TS 的 create 路径
     （`flags & 1`）确认走通，让 TA 有机会落盘。
3. 一旦 fingerlist 合法，enroll 就会进入真正的采集（`camera_capture` 内部
   会重新过 `milan_check_finger_up`），那时才需要判断"门 + 真图"是否都正常。

### 本轮新增的工具一览

| 工具 | 作用 |
| --- | --- |
| `ta_xrun.py <qemu.log>` | qemu 逐指令 trace + 反汇编 + 寄存器注解 |
| `ta_find.py "<字符串>"` | 找引用某 rodata 字符串的代码 |
| `hal_dis.py <符号>` | 反汇编 `libgf_hal.so`（aarch64），符号来自 debugdata |
| `mk_enroll.py` | 生成 180 字节 enroll payload（含 token/HMAC） |
| `ta_host32 req/plan` | 带 payload 的单条/整串命令 |
| `GF_POKE=<addr>=<val>` | 每条命令前写 TA 数据段（INIT 会重置全局，所以必须每命令前写） |

### 本轮已验证的原始数据

* `spi2.rec` / `work/dac-dev.rec` 里 `0x00a2` 的**第一次**读永远是
  `00 40 10 20 10 40 …`（不是真实像素），**第二次**读才是
  `00 01 89 01 66 01 4d …`。第一次读发生在 `get FDT average data`，
  第二次发生在 check_finger_up 的失败尾巴里。不要拿第一次的数据判断画质。
* 设备上 `0x0220 → 0x0c68`、`0x0236 → 0xc7`、`0x0238 → 0xc6`、
  `0x023a → 0xc6`：DAC 校验用 `[0x2ab324+0xe4]=0xc6` 与读回值比对，
  在真机上**是通过的**（离线回放曾经误报 dac error，因为回放按序号对齐，
  请求 `f0 02 20` 拿到了录制里 `f0 00 a2` 的应答——回放工具的这个偏差要记住）。

---

## 2026-11-08 深夜（第三轮）：enroll 跑通，卡在 capture 的派发

这一轮把 enroll 从"报错"推进到"会话能开、token 能过、SPI 能配传感器"，
并且**修掉了一个一直伪装成 TA 崩溃的宿主 bug**。剩下的是 TA 自己的
capture 派发。

### M1.34 ⭐ "Illegal instruction" 是 musl 的堆损坏陷阱，不是 TA 的错

现象：enroll 走到最后总是

```
[cp] open type=0x7 algo=0x7004
Illegal instruction      (rc=132/139)
pc=0x10005954  lr=0
```

`0x10005954` 处是 `.word 0xe7f000f0`（ARM `udf #0`），这是 **musl 的
`a_crash()`**：`free()` 检测到堆 chunk 损坏时主动陷阱。也就是说崩溃点在
**宿主自己的堆**，跟 TA 的异常路径无关。

根因：我们在 `req` / `plan` 里按 `in_len + out_cap` 分配请求缓冲区，
而 `out_cap = 0`（或很小）时，TA 仍会往输出区写东西 → 越界写进 musl 堆。
**给 4 KiB 之后崩溃彻底消失**，enroll 正常返回 0。

```
# 现在 req/plan/enroll 默认 out_cap = 4096
printf '1000 - 4096\n1001 - 4096\n1005 - 4096\n' | ...
```

> 这也是"崩溃地址每次都不一样、而且总落在宿主 musl 里"的原因。

### M1.35 命令 ABI 的两个补充

1. **第三个参数是请求长度**，不是指针。`ut_pf_fp_invoke_command(desc, req,
   len)`，TA 内部用 `len - 0x10` 定位 payload；`len` 的取值会改变派发分支
   （给 0 和给 100 走的路不同，后者报 `GF_ERROR_BAD_PARAMS`）。
   `plan` 现在支持第 5 列显式指定它：`1021 - 4096 1 100`。
2. **`desc` 的形态也有影响**：`descriptor`（`&cmd` + `&origin`，即 `run`/`cap`
   模式）和 `request`（16 字节头 + payload）在 TA 里会走不同分支，
   capture 尤其明显（见 M1.41）。

### M1.36 ⭐ `cmd 1005` 是算法初始化（我们一直在跳过）

之前所有实验都漏了这条命令。它会打印：

```
[gf_algo_common][gf_algo_common_init] chip_series=1
[gf_algo_common][gf_algo_common_init] flag_chip_info=304611589
[gf_algo_common][gf_algo_common_init] thr_select_bmp=600
[gf_algo_common][gf_algo_common_init] sensor_id=0x47,0x5A,0x59,0x34,0x38,0x31,0x30,0x39,...   → "GZY48109"
[gf_algo_common][gf_algo_common_init] max_fingers=32
[gf_algo_common][gf_algo_common_init] max_fingers_per_user=5
```

也就是说 **`max_fingers_per_user` 本来就应该由 1005 设置**——之前用
`GF_POKE=0x2b9dcc=5` 是绕路，正确做法是先调 1005。1005 自己会做 31 次 SPI。

所以正确的启动序列是：

```
DETECT(1000) → INIT(1001) → ALGO_INIT(1005) → PRE_ENROLL(1006) → ENROLL(1007)
```

### M1.37 enroll 是一个会话，不是一条命令

从设备日志抓到的关键行：

```
[gf_algo_common][gf_algo_common_pre_enroll] g_enrolling_finger is not NULL
```

`ENROLL(1007)` 成功一次之后，TA 把会话记在 `g_enrolling_finger` 里，**之后的
`PRE_ENROLL` 全部被拒**；同时 `gf_hal_common_enroll`（0x1f4bc / 0x1f60c）在
enroll 成功后会立刻再发一条 **1021（camera_capture）**。

所以 per-touch 的循环应该是：

```
PRE_ENROLL → ENROLL            （只做一次，开会话）
for each finger touch:
        wait IRQ → CAMERA_CAPTURE(1021)
POST_ENROLL(1008)              （落盘）
```

challenge 是**一次性**的：复用同一个 token 再 enroll 会报
`GF_ERROR_INVALID_CHALLENGE (1056)`；因为我们的 `ut_pf_cp_rd_random` 固定返回
0x42，challenge 恒为 `0x42424242`，放在 token 的 `msg+4`。

### M1.38 `1008` 是 POST_ENROLL

`gf_hal_common_post_enroll` = `mov w0,#0x3f0` → **cmd 1008**（`0x1f6d8`）。

### M1.39 其它命令的结论（省得再试）

| cmd | 结论 |
| --- | --- |
| 1025 / 1026 | `GF_ERROR_UNKNOWN_CMD (1070)`，此固件不支持 |
| 1016 | 会走 `gf_hal_invoke_command(…, 0x15c)` 的同一族，但裸调会 SEGV（需要 payload） |
| 1010 | ESD 检查（`gf_hal_check_esd`），+4 SPI |
| 1020 | `set_safe_class`，104 字节 buffer，`class` 在 `+0x64` |
| 0x401 / 0x402 | 不是 TA 命令：HAL 的 IRQ 路径用它们调 `gf_hal_notify_acquired_info` / `gf_hal_notify_error_info` |

HAL 侧对应关系（`libgf_hal.so`）：

```
gf_hal_common_camera_capture → gf_hal_invoke_command_ex(0x3fd)   100 字节请求
gf_hal_common_enroll        → invoke(1, 0x3ef, buf, 0xb4) 然后 0x3fd
gf_hal_common_pre_enroll    → invoke(1, 0x3ee, buf, 0x70)       112 字节
gf_hal_common_post_enroll   → gf_hal_invoke_command_ex(0x3f0)
hal_milan_an_series_init    → gf_hal_invoke_command(0x3e9, buf, 0x15c) 然后 0x3f8
```

### M1.40 INIT 的 348 字节 payload 是什么

`hal_milan_an_series_init` 里：malloc(0x15c) → 清零 → `gf_hal_get_fw_info()`
→ `gf_hal_common_load_otp_info_from_sdcard(buf+0x90, …)` → `memcpy` 0xc0 字节。
也就是 **fw_info + /sdcard 上的 OTP 备份**；没有备份时这份 payload 本来就基本
全零，所以我们现在传全零是**正常**的（不是 capture 不工作的原因）。

### M1.41 当前唯一卡点：`1021` 在派发阶段就返回

逐指令 trace（`-d exec` 全量，7.8 GB，取尾部）显示，capture 调用执行的最后
几条 TA 指令是 `0x176b6 → 0x176d2 → 0x17620`（`b 0x17620` 就是返回），
即 **`0x175d0` 这个通用入口里 `(CONST + cmd) > 0x5b` 的"不支持"分支**，
而真正按 cmd 查表跳转的 `0x176d4`（`ldr r0,[r7,r1]; mov pc,r0`）根本没执行到。

实测三种形态：

| 调用形态 | 结果 |
| --- | --- |
| `desc{0,0,1}` + `&cmd` + `&origin`（`cap`/`run` 模式） | **+20 SPI**（把传感器切到 FTD down），然后返回 |
| 同上，但已经先跑过 `1005` | **+0 SPI**，一个 stub 都不碰 |
| `req` 形态（16 字节头 + payload） | +0 SPI，不碰传感器 |
| `desc[0]=1` 或 `2` | +0 SPI |
| `desc[2]=2` | `GF_ERROR_BAD_PARAMS` |
| `desc[2]=3` | `GF_ERROR_UNKNOWN_CMD` |

也就是说：**1021 在我们这个状态下只会做一次"arm"（配置传感器 + 切 FTD down），
之后就什么都不做。** 手指按下时 IRQ 正常（设备上 6/6 次都触发），但 capture 不读图。

下一步要解决的正是这个"arm 之后谁去真正取样"的问题。两个方向：

1. **把 `0x175d0` 的派发读透**：`CONST + cmd` 里的 CONST 决定走哪条路，
   而 `0x176d4` 的跳转表在 `0x1793c`（`table[(CONST+cmd)]`）。先静态把
   `0x1764a` 那个 pool 常量取出来（注意 Thumb ldr 的 PC 对齐），就能知道
   为什么 1021 被判"不支持"，以及正确的入口是什么。
2. **看 TA 期望谁在 IRQ 之后调用它**：`hal_milan_irq` / `irq_image_enroll_or_authenticate`
   里除了通知调用还有 `hal_milan_authenticate_success` 这类函数，读一遍就能
   知道"手指沿到来"之后 HAL 实际发的是哪条命令。

### 本轮新增/改动的工具能力

| 能力 | 用法 |
| --- | --- |
| 完整 enroll 流程（IRQ 驱动） | `./ta_host32 goodix.ta enroll 6 20000` |
| 指定要试的采集命令 | `GF_CAPS=1021,1005 ./ta_host32 … enroll 1 20000` |
| 每轮重新 pre_enroll（调试用） | `GF_ENROLL_EACH=1 ./ta_host32 … enroll …` |
| plan 指定请求长度 | `plan.txt`: `<cmd> <file\|-> <out_cap> <sub> <len_hint>` |
| 每条命令之后看最后碰到的 stub | plan 输出里的 `last=` |
| 崩溃时打印 TA 调用栈 | 自动（扫描栈上落在 TA 代码段的值） |
| 加密上下文重复 close 保护 | 自动（`[cp] close: stray ctx ...`） |

---

## 2026-11-09 凌晨（第四轮）：三个硬件级 bug，第一次采到真图

这一轮的结论全部在**真机**上验证过。三个 bug 之前一直互相掩盖，导致前面几轮
的很多推论是建立在坏数据上的。

### M1.42 ⭐ 传感器需要开机复位，否则 TA 读什么都是 0

`ta_host32` 一直只是 `open("/dev/goodix_fp")` 就开始跑，而原厂 HAL 每次启动
都会 `ioctl(GF_IOC_RESET)`。不复位的后果：

```
[spi#1] f0 00 00 → 00 00 00 00 00      (芯片 ID 全 0)
→ DETECT 里 milan_check_finger_up 空指针崩溃
   pc=0x25774  addr=0x210
```

写了一个不经过 TA 的探针 `fpread.c`（直接走 `/dev/goodix_fp` 的
`GOODIX_FP_IOC_XFER`）来隔离问题：**复位后** `chip id = 00 10 a0 00 25`、
`0x58 = 02` —— 传感器本身是好的。所以是"没复位"，不是"传感器坏了"。

> `fpread` 的坑：`struct goodix_fp_xfer` 必须 `rx_len = tx_len` 且 `rx` 指向
> 另一个缓冲区；`rx_len = 0` 时驱动什么都不回填（第一次写探针就被这个骗了，
> 误以为传感器全 0）。
>
> 现在 `ta_host32` 默认在打开设备后复位一次（`GF_NO_RESET=1` 关闭），
> 另外有 `./ta_host32 goodix.ta reset [次数]` 可以单独复位。

### M1.43 ⭐⭐ `0xa2` 读回 `40 10 20…` 的真因：采样太早，不是 SPI 时序

这是从 M1.24 一直悬着的问题。现在的解释是确定的：

* TA 把地址相（`f0 00 a2`）和数据相（`f1 ff …`）拆成**两次传输**，各占一个
  CS 窗口；273 字节配置写完之后传感器还在做 ADC 转换，立刻读回来的是
  **移位寄存器里被挤出来的配置字节**（配置里就含 `…40 10 20 10…`，
  所以读回是 `00 40 10 20 10 40 10 20 …`，周期正好 3）。
* 加一个传输间隙就好了。实测阈值（同一份代码、同一台机器）：

| `GF_SPI_DELAY_US` | `0x00a2` 读回 |
| --- | --- |
| 0 | `00 40 10 20 10 40 10 20 10 40`（配置残留） |
| 500 / 600 / 800 | `00 00 00 …`（ADC 还没出数） |
| **1000** | `00 01 8f 01 6c 01 51 01 45 …`（**真实 12 位采样**） |
| 2000 | `00 01 8f 01 6c 01 50 01 45 …`（同上，每次略有漂移） |

现在默认 1000 µs（`GF_SPI_DELAY_US=0` 关闭），代价是 init 从 1.2 s 变成
约 2.5 s。

**注意**：`milan_check_finger_up` 的判据是
`down_i - delta_up >= avg_i` → 判"手指按着"。在坏数据下 12 个区域同时
"按着"，于是门永远不过——这也解释了为什么之前所有"寄存器 spoof"都无效。

### M1.44 `enroll` 模式漏了 `1005`

`enroll` 只 prime 了 `1000`+`1001`，而 `max_fingers_per_user` 是 `1005`
（算法初始化）设的，结果 enroll 被 `REACH_FINGERS_UPLIMIT (1005)` 拒掉。
补上之后 `ENROLL` 真正做 `+20` 次 SPI 并返回 0。

### M1.45 脏校准对象会让 capture 直接返回

在坏数据时代写下的 `gf_fdt_base.so` 会被判无效：

```
[gf_base][base_check_fdt_base_validation] exit. err=GF_ERROR_INVALID_BASE, errno=1058
```

之后每次 `1021` 都是 `+0` 次 SPI（一个导入 stub 都不碰）。删掉
`gf_fdt_base.so` / `gf_calibration.so` / `gf_nav_base.so` 让它重建才行。

### M1.46 ⭐ 第一次采到真图

清掉脏对象后跑一次 `enroll`，TA 在 `ENROLL` 内部自己做校准并**读了一帧**：

```
[spi#1248] tl=27365 rl=0 n=27365 desc=…00000007 00000007… f1 ff ff …
           rx=00 75 1b 44 5b 69 75 85 f4 14 …
[gf_cali_common][gf_cali_common_get_state] g_calibration_data.calibration_type=0
```

`00 75 1b 44 5b 69 75 85 f4 14…` 是典型的 12 位像素分布（值域宽、相邻不相关），
和之前 `40 10 20` 那种周期垃圾完全不同。同时
`gf_calibration.so`（23168 B）被写下。

### 现在的状态与下一步

* 链路：`复位 → DETECT(578) → INIT(1190) → ALGO_INIT(1279) → PRE_ENROLL →
  ENROLL(+20，内部校准并读一帧 27365 B) → 每手指 CAPTURE → POST_ENROLL`。
* **`CAPTURE(1021)` 在 enroll 会话里仍然是 `+0`**：数据已经没问题，但状态机
  还是把它丢掉。已知 `gf_state_execute_operation(capture=11)` 会检查
  `gf_state_get_current_operation()`（存在 `0x205d38`），把 operation poke 成
  `OPERATION_ENROLL(0x31)` 后 capture 会去做 arm(+20)，但读图仍不发生。
* 下一步建议：
  1. 反 `gf_state_execute_operation` 里 capture 分支的完整条件
     （`0xd5828` 起，比较 `0x35`/`0x3c` 之后走 `0xd520d`）；
  2. 看 `gf_cali_*` 是怎么触发那帧 27365 B 的读的——沿着
     `gf_algo_get_ee_image_info` 往上找"谁发起读图"，把那条路径在
     capture 里复现；
  3. 也可以先做"认证/匹配"方向：`OPERATION_AUTHENTICATE_IMAGE` 可能比
     enroll 的会话更容易走通。

### M1.47 ✅ capture 被丢弃的完整判据（读出来了）

`gf_state_execute_operation(op, buf)` 在 `0xd5828`，判据是三层：

```c
cur = gf_state_get_current_operation();      // [0x205d38]
if (f(cur) - f(op) < 0)   reject;            // 0xd5898 bmi 0xd58f0
else if (h(cur) == 0)     reject;            // 0xd58a6 beq 0xd58da   ← 卡这里
else if (j(op) == 0)      reject;            // 0xd58b8 bne 0xd5904
else                      proceed → gf_state_common_execute(op, buf)  (0xd520d)
```

| 东西 | 地址 |
| --- | --- |
| `gf_state_execute_operation` | `0xd5828` |
| `gf_state_get_current_operation` | `0xd51c4`（`return [0x205d38]`） |
| `op → 优先级` 表（`f`） | 函数 `0xe2338`，表 `0x11ae60`，8 字节/项 |
| `op → 模式` 表（`h`） | 函数 `0xe2344`，表 `0x11b12c`，4 字节/项（`op-5` 索引，`op<5` 返回 0） |
| `op → 名称` 表 | 函数 `0x18389`，名字串在 `0xf9878` 起 |
| 放行分支 | `0xd5904` → `blx 0xd520d` |

实测表值：

```
f(0)=1  f(1)=1  f(2..3)=1  f(4)=0  f(5)=3  f(6)=0  f(7)=3  f(8)=3  f(9..15)=0
h(5)=1  h(6)=0  h(7)=1  h(8)=1  h(其余)=0
```

`[0x205d38]` 的实际取值轨迹（真机 dump）：

```
DETECT 后 0x3c   INIT 后 0x3c   ALGO_INIT(1005) 后 0x09   PRE_ENROLL 后 0x09   ENROLL 后 0x00
```

**结论**：enroll 结束后 `cur = 0` → `h(0) = 0` → capture 直接走 `0xd58da`
（只打日志返回，一个导入 stub 都不碰）。把 `[0x205d38]` poke 成
`5`（`h=1`）之后，capture 变成**每轮稳定 `+20` 次 SPI**（= 读 FDT 基线），
说明状态机这一关过了。

还有一个坑：`GF_POKE_ONLY` 依赖 `g_cur_cmd`，而 `enroll` 模式原先没维护它，
导致"设 operation"实际上被跳过、看起来像"poke 无效"。现在每个命令都会设。

### 下一步（很近，但还差一步）

`+20` 是 `milan_read_fdt_base`（写 cmd `0xC0`、读 `0x58`/`0xa2`、DAC 三连、
写 273 字节配置），也就是"取一次基线"。真正的图像读是
**寄存器 `0xAAAA`、27,365 字节**（校准阶段出现过一次）。所以剩下要做的是：

1. 读 `0xd5904 → gf_state_common_execute(0xd520d)` 之后的流程，看它为什么在
   "读基线"之后返回，而不是继续 `0xAAAA`；
2. 大概率是"基线必须先在**无手指**时建立，之后检测到手指才读图"——也就是
   capture 要按 `arm(无手指) → 等 IRQ → capture(有手指)` 的顺序，
   而我们现在每轮都在 arm。

### M1.48 ⭐ 新线索：FDT 数据只回来 3 个样本（12 个里）

把当前完整流程在真机上录了一份（`cap.rec`，1293 条），离线可回放。里面
capture 的 `0x00a2`（FDT 数据）读出来是：

```
00 01 90 01 6c 01 53 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
   ↑0x190  ↑0x16c  ↑0x153  ↑ 后面 9 个样本全是 0
```

也就是 12 个区域只有**前 3 个**有真实转换结果，其余是 0。这直接解释了
`milan_check_finger_up` 为什么总判"手指已离开"（判据
`down_i - delta_up >= avg_i`：`0xa0 >= 0x190` 不成立 → 判"没手指"）。

按时间算：24 字节 @1 MHz ≈ 192 µs，而每 64 µs 才出一个样本 → 正好约 3 个。
所以最可能的解释是**数据相时钟太快，传感器还没转换完就把后面的字节移出去了**。

下一步实验（工具已就绪，只差一个开关）：

```c
/* ut_pf_spi_send_and_receive 里，现在只用 desc[0] 反算 speed_hz，
 * 而 TA 对这次读给的是 desc[2]=0x36（≈0.93 MHz）。加一个
 * GF_SPI_SPEED_HZ 覆盖，按 250k/500k/925k 各试一次，看 12 个样本是否齐全。 */
```

如果放慢之后 12 个样本齐全，`milan_check_finger_up` 就会正确判断"手指在"，
capture 也就会继续走到读 `0xAAAA` 那一帧（27,365 字节）——这条链就通了。

### M1.49 FDT 基线的格式与极性（都测出来了）

**极性**：手指按下会让 12 个区域的值**大幅下降**（真机同一寄存器对比）：

| 状态 | `0x00a2` 一次读回 |
| --- | --- |
| 没手指 | `01 90 01 6d 01 54 01 46 01 53 01 88 01 66 01 45 01 3b 01 3f` |
| 按住 | `01 87 01 5d 01 41 01 2b 01 3a 01 7c 01 57 01 34 01 1f 01 1f` |
| 持续按住 | `01 19 00 f6 00 db 00 c8 00 d7 01 33 01 11 00 f1 00 e2 00 e7` |

**基线文件格式**：`gf_fdt_base.so` 28 字节 = 4 字节头 + **12 × u16 小端**：

```
8f f3 7e fc | 87 00 74 00 67 00 60 00 68 00 93 00 | 81 00 71 00 6d 00 6d 00 | 00 00 00 00
  (头/CRC)     135     116     103      96     104     147     129     113     109     109
```

注意这些值和原始读数的关系是 **`基线 ≈ 原始读数 - 0x100`**（原始 `0x187` ↔ 基线 `0x87`），
和 `milan_check_finger_up` 里 `ldrb [config+0x424+i]`（字节）一致——基线是按字节存的。

所以门的完整判据在真实数据下是：

```
基线_i (≈ 无手指读数 - 0x100)  -  delta_up(0x13)
     vs  fdt_average_data[i]  (≈ 当前读数 - 0x100)

没手指: 135-19=116  vs  135  → 116 >= 135 不成立 → 判"手指已离开"（正常）
按住:   135-19=116  vs   30  → 116 >= 30  成立   → 判"手指按下"
```

也就是说：**只要 capture 真的调用 `milan_check_finger_up`，手指按下时它就会正确
判"手指在"**。但 hook 日志显示 capture 期间**根本没有调用这个函数**——
`+20` 那一段是 `gf_hw_set_mode`（写 `c0`/273 字节配置/`c1`/`a1`/`c0`、读
`0xa2`+`0x58`+DAC 三连、写 `0x228`），属于"把传感器切到 FTD down"的配置动作。

所以最后一块拼图是：**谁在 arm 之后调用 `milan_check_finger_up` / 读 `0xAAAA`**
（`0x256d5` 就是 FDT 读，`0x25b18` 是那个 12 区域判断）。已知它出现在
`ENROLL` 的校准路径里（那里读到了完整的 27,365 字节帧），把那条调用链
（`gf_algo_get_ee_image_info` → … → `0x25b18`）和 capture 的调用链对比一下就能
找到差异。

### M1.50 图像通道：数据是噪声，且模式分派已解到"mode 1 的 handler"

**识图结论（关键）**：把 `0xAAAA` 读回的 27,365 字节按 160×88（8 位）和
160×114（12 位打包）两种几何还原成 PNG 看，都是**均匀雪花/随机条纹**，
没有任何脊线结构。统计上也吻合：字节近乎均匀分布（mean≈120–136、210+ 个
不同值、几乎没有 0），整段 512 字节分块统计无衰减（不是 FIFO 溢出）。

对照：`0x00a2`（FDT，12 区域）是**真实模拟量**且随按压大幅变化。
结论：**总线本身没问题，是传感器在那次传输里没有驱动 MISO**（浮空噪声）。
因此之前当成"基线图"的 27 KB 其实也是噪声——这也解释了
`base_update_all` 为什么老是误判手指。

**模式分派（已解）**

```asm
gf_hw_set_mode   0x2bd3c   push {r4,r5,r6,r7,lr}
  0x2bd5c  cmp mode, #0x10 ; bls 0x2bde2      ; 模式空间 0..0x10
  0x2bde2  ...
  0x2be10  cmp mode, #8    ; bls 0x2be9a      ; 0..8 走跳表
  0x2be9a  ldr r3,=0x1001b4 ; ldr r0,[r3, mode*4] ; mov pc, r0   ; 跳表
```

跳表 `0x1001b4`：

| mode | handler | 用途（推断） |
| --- | --- | --- |
| 0, 1, 3, 4 | `0x2bed4` | idle / image / ... |
| 2 | `0x2bf34` | |
| 5 | `0x2becc` | |
| 6, 7 | `0x2bea2` | |
| 8 | `0x2bf2c` | |

`capture(1021)` 传的是 **mode = 1** → handler `0x2bed4`。handler 内部：
`blx` 两个 helper 得到 `fp`，然后 `fp==0` → `0x2bf44`、`fp==特定值` → `0x2c240`、
否则 `[r6+0x68]=1` 并继续（mode==3 → `0x2c18c`；mode==0 → `0x2befe`；
**mode==1 → `0x2c19c`**）。

**下一步**：读 `0x2c19c` 起的 mode-1 分支，看它在读 `0xAAAA` 之前写了什么
（启动转换）以及是否轮询就绪位。这是"图像通道悬空"最可能的答案所在。

### M1.51 本阶段可用的工具一览（都在 port/fingerprint/ta32）

| 工具/开关 | 作用 |
| --- | --- |
| `GF_BKPT=<addr>[,..]` | 在 TA 地址写 `bkpt`，命中时打印 **caller lr** 并恢复继续（已修 I-cache 刷新，一次命中） |
| `GF_SPI_DELAY_US`（默认 2000） | 传输间隙；<1000 时 FDT 只回来 3/5 个样本 |
| `GF_SPI_SPEED_HZ` | 强制 SPI 时钟 |
| `GF_SPI_LOG=1` | 逐条 SPI 日志（**默认关**，开着会改变传感器行为） |
| `frame_N.bin` | 每次 >1000 字节的读自动落盘到 `/root/ta/` |
| `GF_POKE`/`GF_POKE_ONLY`/`GF_POKE_AFTER` | 写 TA 数据字；`_ONLY` 限定命令，`_AFTER` 只在该命令后写一次 |
| `GF_DUMP3=<addr>:<len>` | 每条命令后 dump TA 内存 |
| `fpread [noreset]` | 绕过 TA 直接读传感器（判断"硬件好不好"） |
| `replay.sh` + `ta_xrun.py` + `ta_at.py` | 离线回放 / 轨迹注解 / 反汇编 |

### M1.52 图像通道结案报告（本轮为止的确定结论）

**传输层完全清白**（用计时证明）：

| 传输 | 耗时 | 推算速率 |
| --- | --- | --- |
| 2,405 B | 19.6 ms | 0.98 MHz |
| 27,365 B | **222 ms** | 0.985 MHz |

222 ms 正好是 1 MHz 下 27,365 字节的时间 → **内核把每个字节都时钟输出去了，没有截断**。
（此前怀疑 MTK SPI 的 DMA 在 4096 处截断，计时结果否定了这一点，内核无需改动。）

**传感器只驱动前 4096 字节**：第 0–4095 字节是数据（噪声），4096 之后全 0。
重发地址相、加 10/40 ms 间隔都不能让它继续吐数据。

**DAC 确实被写入**（不是"没配置"）：

```
#1169 读 0x0220 -> 0c 68
#1171 写 0x0220 = 0c 68
#1184 写 0x0236 = 00 c7
#1185 写 0x0238 = 00 c6
#1186 写 0x023a = 00 c6
```

**两条取图路径，只用了错的那条**：

| 函数 | 入口 | 是否被调用 |
| --- | --- | --- |
| `gf_milan_get_image_mult_frame` | `0x2c510` | ✅ 被 `base_get_image_base`（`0x1aa55`）调用一次 |
| `gf_milan_polling_get_image` | `0x2cdb8` | ❌ **从未被调用** |

`mult_frame` 内部先做一次 27,365 字节大读，然后还有一个 288/292 字节的
**小分块轮询循环**（`0x2c700` 起）——真正的图像很可能是靠这个循环一块块取的，
而它依赖前面的状态判断，我们没走到。

**本轮修掉的真 bug**：`ut_pf_spi_send_and_receive` 之前不清空接收缓冲区，
驱动没填满的部分会把宿主进程的残留内存当成"传感器数据"。加上 memset 之后
才看清真相（4096 有数据 + 23 KB 全 0）。这一条解释了之前多轮的"噪声"误判。

**待办（按优先级）**：
1. 读 `0x2c700` 起的分块轮询循环的进入条件（`[global+0x94]` 与计数器的比较），
   让 `mult_frame` 走完整个循环而不是只做一次大读；
2. 或者找出让 TA 选择 `polling_get_image` 的开关（配置来自 OTP/config）。

### M1.53 分块轮询循环的真相：它不是"分块读图"，是"帧数"

用 `GF_BKPT` 确认 `0x2c700`（循环）、`0x2c6f4`（大读调用点）、`0x2c774`（循环出口）
都会执行，而且循环**立刻退出**。

退出条件是 `[global+0x94] - 1 <= counter`，其中：

* `global` = `0x26c0f8`（TA 的 RW 段），所以字段地址是 **`0x26c18c`**
* 运行时 `[0x26c18c] = 1`

把它 poke 成 `94`（27,364 / 292 ≈ 94）之后：**循环真的跑起来了**（读了很多 292 字节的块），
但随后 SIGSEGV（`pc=0xb3cc`，越界写 `0xf7fdc000`）——因为 TA 的图像缓冲区
是按这个计数分配的，只改计数会造成越界。

结论：这个计数是**"帧数/缓冲区块数"**，不是"图像分块数"，所以不能靠改它来拿到图像。
大读（27,365 字节，只回来 4096 字节）仍然是 TA 唯一的取图方式。
