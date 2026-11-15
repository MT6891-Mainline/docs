# chopin 启动链：从上电到 denial

把 Redmi Note 10 Pro（MT6891 / chopin）从上电到桌面的**每一步**写清楚：谁在什么
时候做了什么、失败时屏幕/日志会是什么样、以及已知的问题出在哪一环。

配套文档：

* 根文件系统的制作 → [`DEBIAN-ROOTFS.md`](DEBIAN-ROOTFS.md)
* 开机/会话当前未解决的问题 → [`BOOT-SESSION-HANDOFF.md`](BOOT-SESSION-HANDOFF.md)
* initramfs 源码 → `git/chopin-initramfs`（`rootfs` 分支）

---

## 0. 一张图

```
上电
 │
 ├─(1) LK / bootloader ── 读 boot.img（mkbootimg v2）、应用 DTBO、加载内核
 │
 ├─(2) 内核早期 ──────── 设备树、reserved-memory、控制台、ramoops
 │      ~3.5 s
 │
 ├─(3) initramfs PID 1 ─ 挂 /dev/sdc75 → 拷固件 → switch_root → exec /sbin/init
 │      （我们的 init.c，557 行）
 │
 ├─(4) systemd ───────── 目标 multi-user ~7.2 s；chopin-scp 加载 SCP 固件
 │
 ├─(5) getty@tty1 ────── agetty --autologin kily → denial-session → deniald
 │      （denial 桌面起来）
 │
 └─(6) 工作状态
```

正常一次冷启动的实测基线（`systemd-analyze`）：

```
Startup finished in 3.564s (kernel) + 7.264s (userspace) = 10.828s
multi-user.target reached after 7.172s in userspace
```

---

## 1. LK / bootloader

**它做的事**：读 `boot` 分区 → 解 mkbootimg v2 头 → 取 kernel/ramdisk/dtb →
应用 dtbo 容器 → 跳进内核。

**我们的镜像**（`port/tools/build-images.sh`，`mkbootimg --header_version 2`）：

| 产物 | DTB | 用途 |
| --- | --- | --- |
| `boot-stockdt.img` | 从原厂 boot.img 抽出来的 stock DTB | **只刷这个**：LK 眼里和原厂一致 |
| `boot-stockdt-wrapped.img` | stock DTB 包进 Android DTBO 容器 | LK 按 DTBO 解析时会用到 |

> ❌ **不要用自己编译的 DTB 出镜像**。早期的 `boot-owndtb.img` 和
> `boot-owndtb-rawdtb.img` 两个变体**在本板上都开不了机**：raw 的那份 LK 直接拒绝
> （`ufdt_apply_overlay() failed`，内核根本不启动），包了容器的那份能过 LK、但死在
> 内核早期。`port/tools/build-images.sh` 已经把这两个产物移除，只出 stock-DT 两个。

> 关键点：LK 会把 **dtbo 分区**叠加到 base tree 上。所以 stock DTB 那条路最稳——
> 原厂 LK 认识它，不会因为不认识我们的树而报错。

**cmdline 的归属**：内核配了 `CONFIG_CMDLINE_FORCE=y`，所以 **LK 传的 cmdline 被完全
忽略**，实际生效的是内核里编进去的那串：

```
console=tty0 console=ttyS0,921600n8 earlycon=mtk8250,mmio32,0x11002000
clk_ignore_unused regulator_ignore_unused pd_ignore_unused
ignore_loglevel panic=0 printk.devkmsg=on
chopinport=v2onehundredfifth log_buf_len=4194304 pstore.kmsg_bytes=262144
ramoops.mem_address=0x48090000 ramoops.mem_size=0x8f000
ramoops.record_size=0x1000 ramoops.console_size=0x40000
```

含义：

* `console=tty0` —— 内核日志同时打到**屏幕**（fbcon）；这也是"卡住时只有屏幕能看"的原因；
* `console=ttyS0` —— SoC UART（手机上接不出来，仅供内核早期）；
* `ignore_loglevel` —— 连 debug 级也打，所以屏幕很吵；
* `ramoops.*` —— 崩溃/卡死日志的持久化（见 §2）；
* `chopinport=v2*` —— 版本标记，每次出镜像递增，用来在 `dmesg` 里确认刷的是哪版。

---

## 2. 内核早期

**设备树**：`git/chopin-linux-7.3` 的 `mt6893.dtsi` 是从原厂 DTB **反编译**得来的
（上游 7.3.3 没有 mt6893 的 DTS），这是整个移植的基础。

**关键 reserved-memory**：

| 节点 | 用途 |
| --- | --- |
| `mblock-19-framebuffer` | LK 传下来的显示缓冲（约 40 MB @ `0x77a20000`） |
| `mblock-17-pstore` | pstore/ramoops 区（`0x48090000`，`0x8f000`） |

**控制台与日志**：

* 屏幕（fbcon）＝ `console=tty0`：**系统卡死时唯一的观测通道**；
* `ramoops`：`CONFIG_PSTORE_RAM=y` + `CONFIG_PSTORE_CONSOLE=y`，启动日志里能看到

  ```
  printk: legacy console [ramoops-1] enabled
  pstore: Registered ramoops as persistent store backend
  ```

  ⚠️ 但 **长按电源键断电会丢失 DRAM 内容**，pstore 也就空了。只有"热重启/看门狗复位"
  才能保住它。详见 [`BOOT-SESSION-HANDOFF.md`](BOOT-SESSION-HANDOFF.md)。

---

## 3. initramfs（PID 1）

**源码**：`git/chopin-initramfs` 的 `rootfs` 分支（本地副本 `port/initramfs-rootfs/init.c`，
557 行，静态编译，707 KB）。

**做了什么**（`main()`）：

```c
mount_pseudo();                     // /dev /proc /sys /run /tmp、devpts
msg("chopin init: boot partition %s", ROOT_DEV);   // ROOT_DEV = /dev/sdc75
dev = root_device();                // 优先按 PARTLABEL 找，兜底 /dev/sdc75
for (i = 0; i < 100; i++) {         // 最多等 10 秒设备节点出现
        if (access(dev, F_OK) == 0) break;
        usleep(100000);
}
switch_root(dev);                   // 见下
for (;;) sleep(3600);               // 失败：静默挂着，没有串口就看不到原因
```

**`switch_root()` 里按顺序做了 8 件事**：

| # | 动作 | 说明 |
| --- | --- | --- |
| 1 | `mount(dev, /newroot, ext4, RO)` | 失败再试 f2fs |
| 2 | 找 `/sbin/init`（或 `/init`、`/usr/lib/systemd/systemd`） | 都没有就放弃 |
| 3 | `MS_REMOUNT` 成读写 | |
| 4 | `fwload_nvdata()` | 挂 `/dev/sdc6`（nvdata）只读，把 `APCFG/APRDEB/WIFI`、`BT_Addr` 拷进 rootfs |
| 5 | `fwload_vendor()` | 把 `super` 分区（`/dev/sdc61`）**按 sector 偏移 loop 挂载**（`13246464*512`），取厂商固件 |
| 6 | `install_firmware()` / `install_gpu_driver()` | 把 initramfs 里的固件（mali、novatek 触摸等）装进 rootfs 的 `/lib/firmware` |
| 7 | `copy_tree("/lib/modules", ...)` | 模块树拷进 rootfs（bind mount 会让这块板重启，所以用拷贝） |
| 8 | mount proc/sys/dev/devpts/run/tmp → `MS_MOVE` → `chroot` → `exec /sbin/init` | |

**日志标志**（判断卡在哪一步最直接）：

```
chopin init: boot partition /dev/sdc75
booting rootfs /dev/sdc75
exec /sbin/init
```

* 只看到第一行 → 卡在等设备或 `switch_root`；
* 看到 `rootfs boot failed: ...` → 挂载/找 init 失败，之后 `sleep(3600)` 永久静默；
* 看到 `exec /sbin/init` → initramfs 的活干完了，问题在 systemd 之后。

---

## 4. systemd

**时间基线**：multi-user 在 7.2 s 左右到达。

**启动链上真正重要的 unit**：

| unit | 作用 | 备注 |
| --- | --- | --- |
| `chopin-scp.service` | 加载 SCP 固件（传感器/音频协处理器） | 约 2.2 s，启动耗时前几名 |
| `systemd-udev-trigger.service` | 触发 udev | 约 1.5 s |
| `chopin-charge-tune.service` | 提高充电电流限制 | 由 udev 在充电器设备事件上拉起 |
| `getty@tty1.service` | **denial 的唯一入口** | 见 §5 |

---

## 5. 会话：getty@tty1 → denial

```
getty@tty1
  └─ agetty --autologin kily --noclear tty1
       └─ login kily
            └─ bash → /usr/bin/denial-session --start-locked
                 └─ deniald --device /dev/dri/card1 --render-device /dev/dri/renderD128
                      └─ denial 桌面（锁屏态）
```

**`denial-session` 启动前会等 DRM 设备**（`/dev/dri/card1` + `renderD128`，最多 30 s），
避免和 `mediatek-drm` 的 probe 抢跑。

**三个 console getty 的分工**（这一处曾经是"开机进不去桌面"的元凶）：

| 单元 | 现在 | 原因 |
| --- | --- | --- |
| `getty@tty1` | agetty（唯一） | 它要 `/dev/console` 的 **flock**，不能被别人抢 |
| `serial-getty@ttyGS0` | `ExecStart=-/bin/bash -l` | 改用 shell 而非 agetty，**不参与抢锁**；USB 串口插上即 root shell |
| `serial-getty@ttyS0` | **masked** | 由 `console=ttyS0` 自动生成，UART 接不出来，只会添乱 |

> 事故记录：agetty 之间用 `/dev/console` 的独占 flock 互斥。当串口那份先拿到锁，
> tty1 的 agetty 会永远卡在 `flock()`（`wchan=locks_lock_inode_wait`），autologin
> 不执行 ⇒ denial 永远不启动，屏幕停在内核日志。详见
> [`BOOT-SESSION-HANDOFF.md`](BOOT-SESSION-HANDOFF.md) §3.1。

**会话失败后的走向**：denial 以普通用户 `kily` 运行，PAM 登录会 `setuid` 失败
（`pam_unix(login:account): setuid failed`），失败路径可能请求 logind 重启/关机。

---

## 6. 关机链

```
systemctl reboot
  └─ systemd: 停 unit → umount → systemd-shutdown
       └─ "Sending SIGTERM to remaining processes..."
            └─ "Journal stopped"
                 └─ reboot(2) → PSCI SYSTEM_RESET → LK → 回到 §1
```

正常情况（插着 USB）实测约 1~2 秒完成。

⚠️ **已知未解决问题**：**不插 USB 时关机/重启会卡住**——服务已全部停止（sshd 已死，
SSH 被拒绝），但内核还活着（ping 通），永远不会自己重启。只能用长按电源键断电。
详见 [`BOOT-SESSION-HANDOFF.md`](BOOT-SESSION-HANDOFF.md) §4.4。

---

## 7. 排查速查表

| 现象 | 先看什么 | 结论方向 |
| --- | --- | --- |
| 屏幕上停在内核日志、但 SSH 能进 | `pgrep -a deniald`、`systemctl is-active user@1000`、`grep FLOCK /proc/locks` | 会话没起来（抢锁 / 设备没就绪） |
| 屏幕停在内核日志、SSH 也进不去 | 用串口（`/dev/ttyACM0`，插上即 shell） | 用户态已停：关机流程卡住 |
| 卡在 `chopin init: boot partition ...` | `ls /dev/sdc75`、`mount` | 根分区没出现 / 挂载失败 |
| `rootfs boot failed:` | 上一行的错误码 | 挂载或找不到 `/sbin/init` |
| 起来但黑屏 | `dmesg \| grep -i "panel\|dsi\|underrun"` | 显示链路（见 [`../display/DRM-CLOCK-GAP.md`](../display/DRM-CLOCK-GAP.md)） |
| 想确认刷的是哪版 | `dmesg \| grep chopinport` | 版本标记 |

**关键前提**：卡住时 **WiFi 往往还活着**，先试 SSH，别急着重启（重启会丢现场）。

---

## 8. 文件位置

| 内容 | 路径 |
| --- | --- |
| initramfs init 源码 | `git/chopin-initramfs`（`rootfs` 分支）/ 本地 `port/initramfs-rootfs/init.c` |
| 启动镜像打包 | `port/tools/build-images.sh`、`port/tools/wrap-dtb.py`、`port/tools/extract-stock-dtb.py` |
| 内核 cmdline | `CONFIG_CMDLINE`（`CONFIG_CMDLINE_FORCE=y`） |
| 根分区设备 | `ROOT_DEV=/dev/sdc75`（UFS 的 **LUN2**，`0:0:0:2`） |
| nvdata | `/dev/sdc6` |
| vendor super | `/dev/sdc61`，偏移 `13246464*512` |
| 会话入口 | `getty@tty1` → `/usr/bin/denial-session` → `/usr/bin/deniald` |
