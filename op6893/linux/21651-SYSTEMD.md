# 21651：systemd 作为切根后的启动管理器

21651 的内嵌 BusyBox initramfs 负责启动 USB ACM/ECM、校验 `userdata`、
保护其他 UFS 分区，然后交出 PID 1。交出之前由 initramfs 决定用哪个
init；本机镜像的默认是 rootfs 上的 systemd。早期救援参数仍为
`op6893.rescue=1`。

## init 选择

`userdata/init` 的 `select_init()` 按下面顺序决定 `init_path`：

| 条件 | 结果 |
| --- | --- |
| 镜像 profile 为 `systemd`（initramfs 内 `/init-default`） | 默认 systemd |
| 内核命令行含 `op6893.init=busybox` | BusyBox 监督脚本 |
| rootfs 缺 systemd、缺 `op6893-21651-bringup.target`，或 `/sbin/init` 不是指向 systemd 的符号链接 | BusyBox 监督脚本，`reason=rootfs-not-ready` |
| `/etc/op6893/boot-attempts` 已达 2 | BusyBox 监督脚本，`reason=boot-attempts` |

选中 systemd 时，initramfs 先把尝试计数加一并刷盘，再 `switch_root` 到
`/sbin/init`（即 `/usr/lib/systemd/systemd`）。rootfs 上的
`op6893-21651-healthy.service` 在 `multi-user.target` 起来后执行
`/usr/local/sbin/op6893-21651-boot-ok`：确认 PID 1 是 systemd、记录
看门狗与失败单元数，然后删除计数文件。

因此一次正常启动会留下 `BOOT_OK ... failed=0` 和
`BOOT_ATTEMPTS_CLEARED`；连续两次没能走到 systemd 完成状态（例如
systemd 卡死、被 30 秒硬件看门狗复位）时，第三次启动自动落到 BusyBox
监督脚本并保留计数。要重新试 systemd：

```sh
rm /etc/op6893/boot-attempts
```

## rootfs 侧安装

`21651/userdata/systemd/install.sh <rootfs> <source>` 只做文件和符号链接，
可重复执行。它安装单元、`/usr/local/sbin/op6893-21651-boot-ok`，把
`/sbin/init` 指向 `/usr/lib/systemd/systemd`，把 `default.target` 指向
`multi-user.target`，并在 `multi-user.target.wants` 中启用 USB SSH、ACM
和 healthy 三个单元。原来的 BusyBox 监督脚本保留为
`/sbin/op6893-21651-init`。

发行版自带的 `ssh.service` 会从 `/etc/systemd/system/multi-user.target.wants`
里移除：它和 `op6893-21651-usb-ssh.service` 共用
`/etc/ssh/sshd_config`，而后者固定 `ListenAddress 192.0.2.2`，两者同时
启用会抢同一个端口。

## 构建与实测

在 `mt6893-direct` 工作目录使用
`21651/build.sh userdata radio graphics systemd` 构建；这个 profile 使用
独立的 `21651/out-systemd`，并把 `systemd` 写进 initramfs 的
`/init-default`。

实测镜像 `b3afe1f2e993`（Debian 13.7、systemd 257）：

| 项目 | 结果 |
| --- | --- |
| PID 1 | `/usr/lib/systemd/systemd` |
| `systemctl is-system-running` | `running` |
| `systemctl --failed` | 空 |
| 启动耗时 | 4.754 s 内核 + 3.513 s 用户态，`multi-user.target` 在用户态 3.427 s 到达 |
| 看门狗 | `RuntimeWatchdogUSec=30s`、`RebootWatchdogUSec=30s`，`/dev/watchdog0` 存在 |
| 运行单元 | journald、udevd、dbus、logind、hostnamed、bluetooth、`getty@tty1`、USB SSH、ACM |
| 计数文件 | 启动后不存在（healthy 已清除） |

两条回退路径都在真机验证过：

- 写入 `2` 到 `/etc/op6893/boot-attempts` 后重启，initramfs 记录
  `INIT=busybox reason=boot-attempts attempts=2`，PID 1 为
  `/usr/bin/busybox`，`/etc/op6893/boot-attempts` 保留。
- 删除该文件后重启，恢复 `INIT=systemd attempt=1` → `BOOT_OK ... failed=0`
  → `BOOT_ATTEMPTS_CLEARED`。

## 已知边界

- 只有 `systemd` 与 `standard` profile 的镜像会启用 cgroups 等 systemd
  必需配置；`busybox` profile 始终使用监督脚本。
- `standard` profile 执行 rootfs 自己的 `/sbin/init`；因为本机 rootfs 的
  `/sbin/init` 现在指向 systemd，这个 profile 也会进入 systemd。
- USB SSH 单元固定 `ListenAddress 192.0.2.2`。要让 Wi-Fi 也能登录，需要
  另加监听地址或换用发行版 sshd。
- Debian 自带的 `apt-daily*`、`fstrim` 定时器仍处于启用状态，尚未在
  无网络场景下验收。
- 从 systemd 进入 LK fastboot 仍需先 `insmod` 对应 release 的
  `rtc_boot_probe.ko`，再 `systemctl --reboot-argument=bootloader reboot`；
  已实测 `fastboot getvar is-userspace` 为 `no`。BusyBox 下用
  `op6893-21651-reboot-fastboot`。
