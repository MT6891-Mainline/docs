# 01 · 启动链

从上电到 denial 桌面。完整版（含每个参数、每个日志标志）在
[`archive/BOOT-CHAIN.md`](archive/BOOT-CHAIN.md)；未结案的开机/会话问题在
[`archive/BOOT-SESSION-HANDOFF.md`](archive/BOOT-SESSION-HANDOFF.md)。

## 流程

```
上电
 ├─ LK / bootloader    读 boot.img（mkbootimg v2）、应用 DTBO、跳内核
 ├─ 内核早期           DT、reserved-memory、控制台、ramoops        ~3.5 s
 ├─ initramfs PID 1    挂 /dev/sdc75 → 拷固件 → switch_root
 ├─ systemd            multi-user                                  ~7.2 s
 ├─ getty@tty1         agetty --autologin kily → denial-session → deniald
 └─ denial 桌面
```

实测基线：`3.564s (kernel) + 7.264s (userspace) = 10.828s`。

## 几个非显然的点

**cmdline 由内核说了算。** `CONFIG_CMDLINE_FORCE=y`，LK 传的会被完全忽略，实际生效的
是编进去的那串（`console=tty0 console=ttyS0 ... chopinport=v2*`）。想知道刷的是哪版：
`dmesg | grep chopinport`。

**镜像只用 stock DTB。** `boot-stockdt.img`（原厂 DTB）是唯一能启动的形态；自己编译的
DTB 两个变体都开不了机（raw 被 LK 拒，包了 DTBO 容器的死在内核早期），已从构建脚本移除。

**initramfs 做 8 件事**：挂 ext4(RO) → 找 init → remount(RW) → 拷 nvdata 固件 →
loop 挂 `super` 分区（偏移 `13246464*512`）→ 装固件/GPU 驱动 → 拷 `/lib/modules` →
`MS_MOVE`+`chroot`+`exec /sbin/init`。

**三个 console getty 必须协调**（曾经是"进不去桌面"的元凶）：

| 单元 | 现在 | 原因 |
| --- | --- | --- |
| `getty@tty1` | agetty（唯一） | 它要 `/dev/console` 的 flock |
| `serial-getty@ttyGS0` | `bash -l` | 不用 agetty ⇒ 不抢锁；USB 串口插上即 root shell |
| `serial-getty@ttyS0` | masked | 由 `console=ttyS0` 自动生成，UART 接不出来 |

> 事故：agetty 之间用 `/dev/console` 独占 flock 互斥。串口那份先拿到锁时，tty1 的
> agetty 会永远卡在 `flock()`，autologin 不执行 ⇒ denial 永远不启动，屏幕停在内核日志。

## 已知问题

| 问题 | 状态 | 说明 |
| --- | --- | --- |
| **不插 USB 关机/重启会卡死** | ❌ 未解 | 服务全停（SSH 拒绝）、内核还活着（ping 通）、永不重启，只能长按断电 |
| denial 解锁 PAM `setuid` 失败 | ❌ 未解 | 会话以 `kily` 运行，PAM 无法 setuid，失败路径可能请求 logind 重启 |
| 屏幕上仍会刷内核日志 | 设计如此 | `console=tty0` + `ignore_loglevel`，fbcon 会一直打到合成器接管 |

**取证要点**：卡住时 **WiFi 往往还活着**，先 SSH（`pgrep -a deniald`、
`grep FLOCK /proc/locks`、`journalctl -b -1`），别急着重启——重启会丢现场。
ramoops 虽在写，但**长按断电会丢**。

## 触摸

Novatek NT36672E（SPI），固件 `novatek_nt36672e_*.bin` 必需——
`CONFIG_EXTRA_FIRMWARE` 已关，改为启动时从 `/lib/firmware` 加载
（`git/chopin-linux-7.3` 提交 `12bcdd4`）。
