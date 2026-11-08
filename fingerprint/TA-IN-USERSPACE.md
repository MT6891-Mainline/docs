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
