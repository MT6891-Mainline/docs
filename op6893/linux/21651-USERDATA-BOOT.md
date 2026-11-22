# 21651：使用自选的 userdata 根文件系统

板级 boot 镜像提供内核、DT、LK 镜像封装和内嵌 initramfs。用户将 ARM64
发行版安装到内部 UFS 的 `userdata` ext4 分区，该发行版提供可执行的
`/sbin/init` 和 `/dev`、`/proc`、`/sys`、`/run` 目录。initramfs
查找唯一的 `userdata`，将其他内部 UFS 分区设为只读，并把上述挂载移交给
用户空间。更换 ext4 UUID 或 userdata 容量不要求重建 boot。内核模块必须
与 boot 中的内核版本匹配；看门狗和正常启动后的 USB、网络、图形等服务
由 rootfs 自行配置。

在 `mt6893-direct` 中构建：

| 命令 | 用途 |
| --- | --- |
| `21651/build.sh userdata radio graphics standard` | 正常启动，不预设 USB gadget 或开放 ACM shell |
| `21651/build.sh userdata radio graphics standard usb` | 调试：早期配置 ACM/ECM，根挂载或检查失败时打开 ACM 诊断 shell |
| `21651/build.sh userdata radio graphics standard rescue` | 显式救援，不挂载 userdata，开放物理 ACM 诊断 shell |

如果启动参数能提供 `root=PARTUUID=...`，可进一步限定匹配；目标仍须是内部
UFS 上标为 userdata 的分区。当前 LK 启动链使用内置 DT 的启动参数，不能依赖
从 fastboot 临时追加参数；无需指定 PARTUUID 即可走默认路径。独立救援镜像
也不依赖追加参数。正常镜像在无法移交时会记录 `STANDARD_ROOT_UNAVAILABLE`；
如果选定的 init 缺少动态加载器或运行时依赖，须通过救援镜像修复 rootfs。

2026-11-22 实测：USB 调试变体 `18cba00d308b` 在同一 userdata 上，仅替换
`/sbin/init` 链接即可分别启动 BusyBox 与 Debian systemd 257；两次都挂载
`userdata` ext4 并接管看门狗。它验证的是 init 切换接口，不是两个完整
发行版的兼容性。正常和独立救援变体已构建并通过镜像校验，尚未上机。
