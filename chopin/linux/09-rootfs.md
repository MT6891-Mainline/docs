# 09 · rootfs（Debian 13）

设备上的发行版是 Debian 13 (trixie)，装在 UFS 的 userdata 分区（`/dev/sdc75`）。

## 架构：两件互不干扰的东西

| 东西 | 作用 |
| --- | --- |
| initramfs（PID 1） | 只负责挂 rootfs、拷固件、`switch_root`（[`01-boot.md`](01-boot.md)） |
| Debian rootfs | 真正的系统：systemd、denial、所有服务 |

initramfs **不带任何用户态**（没有 shell、busybox、诊断），所以它出问题时屏幕上
只有内核日志——这是"卡在控制台"这类现象的根源。

## 构建

```sh
# rootfs（debootstrap + 定制）
# 启动镜像
INITRAMFS_MODE=rootfs KERNEL=... DTB=... STOCK_DTB=... ./tools/build-images.sh out
```

完整步骤（debootstrap、源、包选择、GNOME/denial 配置、串口控制台）：
[`archive/DEBIAN-ROOTFS.md`](archive/DEBIAN-ROOTFS.md)。

## 几个坑

* `/sbin/init` 可能不存在（Arch 把 systemd 装在 `/usr/lib/systemd/systemd`）——
  initramfs 里已加回退；
* rootfs 里的 `/lib/modules` 是 initramfs **拷贝**过去的（bind mount 会让板子重启）；
* 串口控制台：`serial-getty@ttyGS0` 改成 `bash -l`（[`01-boot.md`](01-boot.md)）。
