# op6893 — realme GT Neo（MT6893）

本目录记录 OP6893 的两个板级项目号：20615（RMX3031，原开发机）和
21651（当前主要开发机）。两者共用内核移植代码，各用自己的设备树、启动镜像
和无线／基带校准数据。

| 项目号 | 已验证的启动方案 | 状态入口 |
| --- | --- | --- |
| 20615 | 原机 LK 启动 Linux；Debian 根文件系统放在 Android 的 `cache` 分区，不占用 userdata | [外设支持状态](linux/PERIPHERAL-STATUS.md) |
| 21651 | LK 启动 Linux；Debian 从内部 UFS 的 `userdata` ext4 启动，可由 systemd 接管 | [21651 支持状态](linux/21651-STATUS.md) |

[Linux 功能矩阵](linux/README.md)按机型对照实测范围；[移植时间线](linux/TIMELINE.md)
记录各阶段的结论。21651 的适配基于 MT6891-Mainline/linux 的 7.3.3 共享树，
开发分支为 `7.3.3-mt6893-oplus-op6893`。启动封装与板级设备树见
[LK 交接记录](linux/OP6893-LK-DTB.md)。
