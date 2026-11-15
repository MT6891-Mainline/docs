# 13 · 构建：内核基线、设备树、镜像

## 内核基线

| 阶段 | 版本 |
| --- | --- |
| 最初 | Linux 7.2.8 |
| 迁移 | 7.3（[`archive/7.3-MIGRATION.md`](archive/7.3-MIGRATION.md)） |
| 现在 | **7.3.3**（[`archive/KERNEL-7.3.3-PORT.md`](archive/KERNEL-7.3.3-PORT.md)） |

分支约定（`git/chopin-linux-7.3`）：

* `7.3.3-mt6893-xiaomi-chopin` —— 主线：板级支持、固件搬运、显示/USB 修复；
* `fp` —— 指纹工作，rebase 在主线之上。

## 设备树

* `mt6893.dtsi` 是从**原厂 DTB 反编译**来的（上游 7.3.3 没有 mt6893 的 DTS）；
* `mt6893-xiaomi-chopin.dts` 是板级文件（面板、触摸、指纹、充电、音频……）；
* 上游 mt6893 文件对齐情况：`pinctrl-mt6893.c` 与 binding 有两处**有意修正**
  （8 个寄存器窗口名对齐 binding 和 DT；yaml 里 `0x1000b000` 对齐硬件），
  `mt6893-pinfunc.h` 已补回（未被引用，纯对齐）。
* DTS 重整计划：[`archive/DTS-RESTRUCTURE.md`](archive/DTS-RESTRUCTURE.md)。

## 构建命令

```sh
# 内核
bash port/tools/build-kernel.sh

# 启动镜像（只出 stock-DT 两个变体）
INITRAMFS_MODE=rootfs KERNEL=... DTB=... ./port/tools/build-images.sh out-dir
```

产物：

| 镜像 | 用途 |
| --- | --- |
| `boot-stockdt.img` | **刷这个** |
| `boot-stockdt-wrapped.img` | stock DTB 包 DTBO 容器 |

> 自己编译 DTB 的两个变体（`boot-owndtb*.img`）在本板**都开不了机**，已从脚本移除。

刷写：`fastboot flash boot_a <img>`（用户自行操作）。
