# ares 启动链（2026-11-15 实测）

> 材料：`work/ares/hw/expdb2.img`（40 MiB expdb，含 14 次启动的 PL/LK/GZ 日志）、
> `work/ares/hw/lk_a.img` / `lk.bin`（lk_a 分区反汇编）、`work/ares/hw/fdt.dtb`（运行时设备树）、
> `k40g/boot_a.img`（原厂 boot 镜像）。
> 解析工具：`port/tools/parse-expdb.py`、`port/tools/build-ares-images.sh`。

## 0. 总览

```
BROM
 └─ preloader (Preloader_a, 每 1 ms 一条日志)
     ├─ DRAM / UFS / PMIC 初始化，清 UFS SWP
     ├─ 读 A/B 元数据（misc 0x804 "BCAB"）
     ├─ 加载 atf(241,152) / tee(2,345,984) / gz(2,115,056) / lk(1,603,652)
     └─ [BLDR] jump to 0x48200000        (LK 的 ARM 向量表 0xEA000007)
         └─ LK  (AArch32, 0x48200000)
             ├─ load_lk_dtb(): lk_main_dtb 201,784 B（来自 lk 分区，**不是** boot.img）
             ├─ dtbo_a: 82,674 B  → ufdt_apply_overlay()
             ├─ logo / DSI / 背光
             ├─ AVB: img_auth_required=0, boot state: orange
             ├─ load boot image (boot_a, 64 MiB 分区) — 1.1 s
             └─ jump → GZ
                 └─ GZ / GenieZone (EL2)
                     ├─ RKP 加载失败 → *DISABLE*
                     ├─ EL2 stage-2 页表（0x38204800 / pa 0x87ec04800）
                     └─ Linux (EL1)，x0 = FDT 288,607 B
```

## 1. preloader

关键日志（`expdb2`）：

```
[PMIC]Preloader Start
... DRAMC/DCM dump, bq27z561_get_vbat: vbat: 4203, mt6360_enable_wdt: en = 0
[PART] Image with header, name: lk,  size:1603652
[PART] Image with header, name: atf, size:241152
[PART] Image with header, name: tee, size:2345984
[PART] Image with header, name: gz,  size:2115056
[ufs_set_write_protect]: RPMB partition = 1, wp_type = 4 ...
[ufshcd_swp_conf_block_write_prepare]: Write swpcb
swpcb lun 0 / entry[0] wpt_wpf 0x0          <- 清软件写保护 (lun 0/1/2)
[AB] Current boot: Preloader_a
metadata.slot_info[0].priority = 15, 14
metadata.slot_info[0].retry = 1, 1
[TZ_TAGS] res_mem_cfg.start(...): 0x7E4B0000 / 0x7E6B0000 / 0x7E7A0000 ...
[BLDR] Others, jump to ATF
[BLDR] jump to 0x48200000
[BLDR] <0x48200000>=0xEA000007
[BLDR] <0x48200004>=0xEA009A28
[BLDR] next is GZ at 0x000000087EA00000
Log Turned Off.
```

* `0xEA000007` 是 ARM `b` 指令 —— 0x48200000 处是 **LK 的异常向量表**，所以 LK 就是被
  preloader 直接跳到 0x48200000 的（`jump to ATF` 是 MTK 的通用提示语）。
* `Log Turned Off.` 之后 LK 不再往 UART 打日志；后面所有内容都靠 LK 的 log_store 环
  （`0x7ffbf000`）转存进 expdb。**ares 的 expdb 没有转存我们的内核写入**（见 §7）。
* preloader **每次启动都会清掉 UFS 软件写保护**（`swpcb entry[0] wpt_wpf 0x0`，lun 0/1/2），
  这是后面理解"写保护"的关键。

## 2. LK

* 链接在 **0x48200000**，**AArch32/Thumb**，lk_a 分区 = 512 B MTK 头 + 1,603,652 B 负载
  （`work/ares/hw/lk.bin`）。断言函数在 0x48252830，调用点 0x4822bc40。
* **基树来自 lk 分区里的 `lk_main_dtb`**，不是 boot.img 的 dtb：

```
[LK_BOOT] lk_a name : lk_main_dtb
[LK_BOOT] lk_a size : 201784
load_lk_dtb(): load lk_main_dtb dtb_img_size=201784
dtbo_total_len: 0x14332
dtbo_size : 82674
[PROFILE] ::: lvl(3) dtb_overlay takes 15 ms
```

* logo/显示：

```
[lk logo: mt_disp_show_logo_by_index 153]index = 1
mt_get_logo_db_addr: 0x76620000      <- LK 的 logo 缓冲（mblock-19 起）
mt_get_tempfb_addr: 0x75c20000
[decompress_logo ...] out=0x75c20000, logolen=10444800   (1080x2400 ARGB)
[show_logo_common]dst_addr= 0xff000000 ...
mt_disp_update: 0x78a0c000, 0x00000000
```

* AVB（**失败但不阻塞**）：

```
[avb] img_auth_required = 0
avb_vbmeta_image.c: ERROR: Magic is incorrect.
avb_slot_verify.c: ERROR: boot_a: Error verifying vbmeta image: invalid vbmeta header
[avb] avb_ret = 6
[LK] check_ota_result = 0
[LK] ota-fail
boot state: orange
```

* 加载 boot image：

```
[1110] [DISP] ... DSI_set_cmdq ...
[2111] [PROFILE] ::: lvl(1) load boot image takes 1127 ms
[2111] panic (caller 0x4822bc45): ASSERT at (platform/common/boot/boot_info.c:616): g_boot_info.hdr_loaded
       ^^^ 这一行只在镜像不合约时出现；合约正确时这里是跳 GZ
```

### 2.1 boot image 契约（**实测自 stock，和 chopin 不同**）

| 组件 | stock 大小 | 地址 |
| --- | --- | --- |
| kernel | 13,327,208（gzip，text_offset 0x80000） | 0x40080000 |
| ramdisk | 35,695,024 | 0x51100000 |
| tags | — | 0x47c80000 |
| dtb | 201,848（**64 B DTBO 容器头 + 201,784 B FDT**） | **0x47c80000** |
| header | v2, page 2048, 内核 0x40080000+0x80000 | |

`boot_a` 分区 = sdc32 = 131,072 扇区 = **64 MiB**（不是 chopin 那份 dump 的 128 MiB 文件大小）。

**核心差异**：ares 的 `dtb_addr` 是 `0x47c80000`（= tags 地址）；chopin 的 stock 头是
`0x41f78000`。照抄 chopin 的值时，`0x41f78000` 落在我们 49 MiB 内核区间
（`0x40080000`+49 MiB = `0x431d0000`）里，LK 在 `boot_linux()` 里调
`boot_info_get_*()` 时 `g_boot_info.hdr_loaded == 0` → 直接 panic。

反汇编佐证（`lk.bin`）：

```
0x48252830: ldr r3,[pc,#36] ; = &g_boot_info
0x48252834: ldr r2,[r3,#4]  ; hdr_loaded
0x48252836: cbz r2, panic   ; -> mov.w r3,#616 ; "ASSERT at boot_info.c:616"
0x48252838: ldr r0,[r3,#40] ; 正常返回字段 +0x28
```

调用它的 `boot_linux()` 在 0x4822bc40（返回地址 0x4822bc45，与日志完全一致）。

## 3. GZ / GenieZone（EL2）

```
====> GZ (0)[    0.025765][memlog_static_init] register pa0x7e800000, size 0x41000
====> GZ (0)[    0.025769]INIT: cpu 0, calling hook ... (KTEE_LK_Init)
====> GZ (0)[    0.025808][MTEE SYS] 112: gz-md shared buffer is not configured!
====> GZ (0)[    0.025941]Kernel table root va=0x38204800, pa=0x87ec04800 num_entries=64
====> GZ (0)[    0.030675][Kernel-server] 1207: start kernel service main
[GZINIT] ERROR, load RKP failed, *DISABLE* RKP feature
```

* GZ 日志写到 `mblock-4-gz`（0x7ea00000，0x1400000）和 `mblock-5-gz-log`（0x7e800000，0x41000），
  由 LK 的 kedump 转存进 expdb —— 所以我们在 expdb 里能看到 GZ 的日志。
* 启动参数：`androidboot.tee_type=1`、`androidboot.secureboot=1`。
* GZ 运行在 EL2，Linux 在 EL1；GZ 自己占 20 MB（mblock-4），另有 gz-log/pstore/tee 等 mblock。
* **RKP 被禁用**（`load RKP failed`）—— 和 ares 上跑非签名镜像有关，不影响启动。

## 4. 内核看到的世界

* `x0` = LK 叠加 dtbo 后的 FDT：**288,607 字节**（`adb shell su -c cat /sys/firmware/fdt`）。
  里面有 32 个 `mblock-*` 保留区（清单见 `STATUS-2026-11-15.md` §2）—— 这份树才是设备真实布局，
  我们早期用 "boot.img dtb + dtbo 用 fdtoverlay 合成" 得到的树**一个 mblock 都没有**。
* cmdline 是 LK 拼出来的（约 1.6 KB），包含 `console=tty0 console=ttyS0,921600n1 root=/dev/ram`、
  `ramoops.mem_address=0x48090000 mem_size=0xe0000`、`androidboot.*`、
  `mtk_printk_ctrl.disable_uart=1`（**内核不能往物理串口打日志**）。
* chopin 的内核在 `setup_arch()` 里把 LK 的 FDT 换成内嵌的板级树（`chopin-dtb.S`）；ares 走同一套机制，
  但现在内嵌树是从 dtbo 合成的，缺 mblock —— 需要换成 `fdt.dts` 那份。

## 5. 写保护：Part_WP / UFS SWP

LK 里有一整套写保护代码：

```
set_write_protect
Bypass write protect!
write protect Done!
[Part_WP]invalid blksz
[Part_WP]Not found start partition %s info
Partition is write protected
[LK ERROR] UFS clear write protect fail!!!
```

实测（Android 里 root）：

| 分区 | 写 | 说明 |
| --- | --- | --- |
| `expdb` (sdc3) | ✅ | 我写随机数据回读校验通过 |
| `boot_a` (sdc32) | ❌ `Sense Key 0x7 ASC=0x27`（WRITE PROTECTED，Buffer I/O error ... lost async page write） | LK 启动 Linux 前给引导分区加了保护 |
| `boot_para` (sdc41) | ❌ 同上 | |

**结论**：Android 里的 `dd` 写 boot 分区**永远不算数**（写入进页缓存、落盘时被 UFS 拒绝）；
只有 fastboot（LK 自己写）才是真的。这也解释了为什么"在 Android 里 dd 了 t6，但设备还是启旧镜像"。

## 6. A/B

* `ro.boot.slot_suffix=_a`；preloader：`metadata.slot_info[0].priority = 15, 14`、`retry = 1, 1`。
* LK fastboot 支持 `current-slot`、`slot-count`、`slot-successful:a/b`、`slot-unbootable:a/b`、
  `slot-retry-count:a/b`、`set_active:`。
* 建议：**实验镜像写 `boot_b` + `fastboot --set-active=b`**，失败几次后 A/B 自动回退到 `boot_a`，
  `boot_a` 始终保持 stock，就不会再出现"刷坏回不来"。

## 7. 和 chopin 的差异（已确认）

| 项 | chopin | ares |
| --- | --- | --- |
| LK | 另一版（1.7 MB 量级） | 1,603,652 B @ 0x48200000 |
| **dtb_addr** | **0x41f78000** | **0x47c80000** |
| boot 分区 | 128 MiB（dump 尺寸） | **64 MiB**（sdc32） |
| 基树来源 | lk 分区 `lk_main_dtb` | 同（201,784 B） |
| dtbo | 有 | 82,674 B，overlay 15 ms |
| GZ | 有 | 2,115,056 B，`tee_type=1`，RKP 禁用 |
| 写保护 | 有 Part_WP | 有；boot_a/boot_para 被保护 |
| 面板 | k10a 视频模式 | k10-42-02-0a **命令模式** |
| 触摸 | Novatek spi0 | Focaltech spi6 |
| expdb 转存内核日志 | ✅（log_store 环） | ❌ 实测没有（我们的 printk 钩子写了，但 LK 不转存） |

## 8. 失败判读表

| 现象 | 含义 |
| --- | --- |
| logo 后 ~2 s 自动重启，`bootreason=lk_crash` | LK panic。expdb 里看是不是 `boot_info.c:616` |
| `boot_info.c:616 g_boot_info.hdr_loaded` | **boot image 头/契约不对**（dtb_addr 是已知触发） |
| 进 fastboot 而不是系统 | LK 镜像头解析失败，或 misc 里有 `bootonce-bootloader` |
| 出现 `/dev/ttyACM*` 且有内核日志 | 内核活了（`console=ttyGS0` 生效） |
| 出现 ACM 但无输出 | 内核活了，但 console 没接 `ttyGS0`（或 init 还没跑） |
| 彩色横条 | 内核 + framebuffer 都活了（`fb_progress`） |

## 9. 镜像清单（dtb_addr 审计）

`port/tools/build-ares-images.sh` 之前没有，旧镜像是用 chopin 的脚本打的，`dtb_addr` 全是
`0x41f78000` —— **这些镜像必然 panic**：

| 镜像 | dtb_addr | 结果 |
| --- | --- | --- |
| `out-ares-v1/boot-stockdt*.img` | 0x41f78000 | ❌ panic |
| `out-ares-min/boot-stockdt*.img` | 0x41f78000 | ❌ panic |
| `out-ares-alpine/boot-stockdt*.img` | 0x41f78000 | ❌ panic |
| `out-ares-diag/boot-stockdt*.img` | 0x41f78000 | ❌ panic |
| `out-ares-t1..t6`, `out-ares-dbg`, `out-ares-shell` | 0x47c80000 | ✅ t1/t6 实测启动 |
| **`out-ares/boot-{min,diag,shell}.img`**（新脚本） | 0x47c80000 | 待测 |

> 所以"换调试 init 就重启"并不是 init 的问题：那份调试镜像（`out-ares-diag/boot-stockdt.img`）
> 还是旧头，被 LK 拒了。用 `out-ares/boot-diag.img` 才是对的头。

## 10. 还没验证的

1. `out-ares/boot-diag.img`（57 MB，diag initramfs）在正确头下能否启动 —— 之前那次失败是旧头。
2. `out-ares/boot-min.img`（= t6 等价，19 MB）复测。
3. 内核日志回传：LK 不转存我们的 log_store 环，得靠 `console=ttyGS0` 实时看，或者让 init 把
   dmesg 写到某个可读分区。
4. 用运行时 `fdt.dts` 替换内嵌板级树（补 32 个 mblock）。
