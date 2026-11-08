# 开机 / 会话问题交接（2026-11-08 夜）

> 状态：**未彻底解决**。已修掉一个真实 bug，但"不插 USB 进不去 / 插拔后要插回"的
> 现象仍在复现。本文把已确证的事实、已验证的修改、仍未解的疑点全部记下来，
> 避免下次从零开始。

## 1. 现象（用户描述）

* 开机后屏幕停在内核日志控制台，denial 不出现；
* **插上 USB 就能进 denial**，拔掉又"停"；
* 有时行有时不行（间歇）；
* 曾经观察到：拔掉 USB 后 ping 通、但 SSH 被拒绝（用户态已停止）。

## 2. 已经**排除**的方向（都有实测证据）

| 方向 | 结论 | 证据 |
| --- | --- | --- |
| 电池/供电 | 正常 | `bq27z561`: Full 100% / 4.95 Ah / 4.338 V；拔掉 USB 后设备仍靠电池运行并联网（`mt6360-chg: Discharging`） |
| initramfs init | 不依赖 USB | 用户提供的 `init.c` 与 `port/initramfs-rootfs/init.c` **字节一致**（sha256 `8984c555…`），重新编译与镜像内 `init` 哈希一致（`43f53c71…`）；init 只做 mount pseudo → 等 `/dev/sdc75` → 挂 ext4 → switch_root |
| 根分区选错 | 排除 | `sda/sdb/sdc` 是同一颗 UFS 的三个 LUN（`0:0:0:0/1/2`），`/dev/sdc75` 名字稳定，不受 USB 影响 |
| 电源键 | 正常 | `event0` 抓到完整 `KEY_POWER(0x74)` 按下+松开；`event1` 有 `KEY_VOLUMEUP` |
| 内核显示栈 | 已修两处（见 §3） | — |
| `logind` 按键/空闲关机 | 已排除 | `logind.conf.d/10-chopin-no-suspend.conf`: `IdleAction=ignore`、`HandlePowerKey=ignore` |

## 3. 已经修掉的东西（都有实测验证）

### 3.1 ⭐ `/dev/console` flock 争用（**真实 bug，已修**）

现场 `/proc/locks`：

```
1: FLOCK ADVISORY WRITE 411 00:09:14    ← serial-getty@ttyGS0 的 agetty 持有
1: -> FLOCK ADVISORY WRITE 408 00:09:14 ← getty@tty1 的 agetty 排队
```

`getty@tty1` 的 agetty `wchan=locks_lock_inode_wait`，**永远拿不到锁**：

* autologin 不执行 → `user@1000.service` inactive → denial 永不启动
* 插拔 USB 会重启 `serial-getty@ttyGS0`，锁短暂释放 → tty1 才有机会 → "有时行有时不行"

修法（已落地）：

```ini
# /etc/systemd/system/serial-getty@ttyGS0.service.d/no-flock.conf
[Service]
ExecStart=
ExecStart=-/bin/bash -l          # 串口调试保留，不再用 agetty → 不抢锁
```

```sh
systemctl mask serial-getty@ttyS0.service   # generator 从 console=ttyS0 自动生成，disable 无效
```

验证：停掉串口 getty 后，tty1 的 autologin 立刻成功，`deniald` 起来了（PID 759，
session 9 = kily@tty1）。

> ⚠️ 教训：**不要在这些 unit 运行中热改 `ExecStart` 再停别的 getty**。实测会让
> systemd 自己卡死（`systemctl` 报 `Connection timed out`；内核活着、ping 通、
> SSH 被拒绝），只能长按电源键硬重启。

### 3.2 面板 refcount（内核 `dd3da5e` / 镜像 v208）

`panel-k10a-bringup.c` 只用 `devm_kzalloc` 手写 panel 初始化，漏了 `kref_init()`，
而 `drm_panel_add()` 内部要 `kref_get()`：

```
refcount_t: addition on 0 → refcount_t: saturated
→ failed to attach primary display(DSI intf)
```

已补 `ctx->panel.container = ctx; kref_init(&ctx->panel.refcount);`。
（`refcount_t` 警告在修复后应消失，但**对本问题的复现没有改善**。）

### 3.3 fbdev 扫描缓冲未清零（内核 `f1c79a7` / 镜像 v207）

开机 logo 上叠着横向彩条 → 根因是 `drm_gem_dma_create()` 分配的 fbdev 缓冲未清零，
且 fbcon 未绑定。已在 `drm_fbdev_dma.c` 的 probe tail 里 `memset` 清零。已确认彩条消失。

### 3.4 denial-session 等 DRM 设备

`/usr/bin/denial-session` 在 exec deniald 前等 `/dev/dri/card1` + `renderD128`（≤30 s），
避免和 mediatek-drm 的 probe 抢跑。备份：`/usr/bin/denial-session.bak-20261108`。

## 4. 仍然未解的关键疑点

### 4.1 重启是**用户态通过 logind 请求**的

上一次 boot 的日志：

```
08:06:48 systemd-logind: The system will reboot now!
08:06:49 systemd[435]: Finished systemd-exit.service - Exit the Session   ← kily 会话退出
08:07:06 systemd[1]: Finished systemd-reboot.service
```

不是按键、不是空闲（两者都是 ignore），是**某个客户端调了
`org.freedesktop.login1.Manager.Reboot`**。`deniald` 二进制里确实有
`org.freedesktop.login1`（3 处）和 `PowerOff`。

**待查**：谁调的、为什么在拔 USB 时调。方法：

* `busctl monitor org.freedesktop.login1` 挂着，然后拔 USB，看谁发 Reboot；
* 或给 logind 开 debug（`systemd.log_level=debug`）看调用者。

### 4.2 拔 USB 后"用户态消失、内核还在"的机制

`ping` 通但 SSH `Connection refused` = sshd 已停止、内核还在 → 典型的**关机流程卡在
poweroff 之前**。需要 `journalctl -b -1` 确认关机序列的最后一步。

### 4.3 为什么修了 3.1 之后仍"没改观"

可能原因（未验证）：

* 3.1 只在一个特定 boot 上被证实；如果 tty1 仍偶发卡住，需要在**卡住的现场**
  再抓一次 `/proc/locks` + `pgrep -a agetty` + `cat /proc/*/wchan`；
* 或者"卡住"的其实是 §4.1 的重启（会话被 logind 干掉），而不是 autologin 失败；
* 或者 denial 自身的 PAM `setuid failed`（kily 非 root）导致会话失败 → 走了重启路径。

## 5. 下次现场如何取证（重要）

**关键前提：卡住时 WiFi 可能还活着**（本次就成功 SSH 进去过）。所以先试 SSH，别急着重启。

```sh
# 1. 谁在跑
pgrep -a deniald; systemctl is-active user@1000.service getty@tty1
loginctl list-sessions

# 2. 谁卡住了 / 谁拿着锁
ps -eo pid,stat,wchan:24,args | grep -E "agetty|login|denial"
grep FLOCK /proc/locks
cat /proc/<pid>/wchan

# 3. 上一次 boot 是怎么没的
journalctl -b -1 | tail -40
journalctl -b -1 | grep -iE "reboot|poweroff|logind|session|usb|gadget"

# 4. 实时抓 Reboot 调用者
busctl monitor org.freedesktop.login1   # 然后拔 USB
```

## 6. 相关文件位置

| 内容 | 路径 |
| --- | --- |
| initramfs init 源码 | `port/initramfs-rootfs/init.c`（sha256 `8984c555…`） |
| 串口 getty drop-in | `/etc/systemd/system/serial-getty@ttyGS0.service.d/no-flock.conf` |
| 会话脚本（已加 DRM 等待） | `/usr/bin/denial-session`，备份 `.bak-20261108` |
| 内核改动 | `fp` 分支：`dd3da5e`（panel refcount）、`f1c79a7`（fbdev 清零）、`12bcdd4`（固件搬出内核） |
| 镜像 | `out-scp-v208-fp/boot-stockdt.img`（含以上全部） |
