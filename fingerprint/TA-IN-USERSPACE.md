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

### M2 SPI 桥 + 第一条真实命令

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
