# ares · Linux 路线

内核 7.3.3 上的 ares 移植。**当前状态：内核在 ares 上极早期失败并重启，未定位。**

## 已完成

| 项 | 内容 | 位置 |
| --- | --- | --- |
| 分支 | `ares`，基于 `7.3.3-mt6893-xiaomi-chopin` | `git/chopin-linux-7.3` |
| 板级 DTS | 用 `fdtoverlay` 把 ares 的 dtbo 叠加到 SoC DTB 上，再反编译（10.7k 行） | `arch/arm64/boot/dts/mediatek/mt6893-xiaomi-ares.dts` |
| 面板驱动 | 厂商 `panel-k10-42-02-0a-dsc-cmd.c` 的表搬进 chopin 主线驱动的骨架；命令模式 | `drivers/gpu/drm/panel/panel-k10-42-02-0a-dsc-cmd.c` |
| 编译 | 内核 + DTB 均编译通过 | — |

## 关键方法：怎么拿到 ares 的板级树

```
boot.img  ──extract-stock-dtb.py──▶ stock SoC DTB (201,784 B，与 chopin 相同)
dtbo 分区 ──DTBO 容器解析────────▶ 1 个 overlay (82,674 B, model="ARES")
                    │
                    └─fdtoverlay 叠加──▶ 261,250 B 完整树 ──dtc──▶ merged.dts
```

这就是 LK 在启动时做的事，所以得到的树是设备真实使用的板级树。

## 失败现象与已排除项

| 现象 | 结论 |
| --- | --- |
| 刷入我们的镜像 → 第一屏后**自动重启**，串口无 `/dev/ttyACM*` | 内核没跑到 USB 初始化 |
| 换 Alpine initramfs → 同样重启 | 与 initramfs 无关 |
| 换**chopin 的 v209 镜像** → **同样重启** | ❌ 与 ares 面板驱动/板级 DTS 无关 |
| 镜像头与 stock 完全一致（`0x40080000` / `0x51100000` / `0x47c80000`，v2） | ❌ 不是打包格式问题 |

## 下一步（按信息量）

1. **对比保留内存**：`reserved-memory` / `mblock-*` 在 `merged.dts` 与 SoC 树里的差异——
   ares 的 overlay 多出 82 KB，若它声明的保留区覆盖内核载入区（`0x40080000` 起 18.5 MB），
   就会在极早期踩内存。
2. **pstore 抓早期日志**：cmdline 里 ramoops 已配（`0x48090000`，`0x8f000`），
   一旦有能启动的镜像就读 `/sys/fs/pstore/console-ramoops-0`。
3. **确认 ares 的 DRAM 布局**：K40G 有 6/8/12 GB 版本，`/memory` 节点与 `CONFIG_ARM64_PA_BITS`
   是否匹配。

## 还没有驱动的东西

| 组件 | 现状 |
| --- | --- |
| Focaltech 触摸 | 只有 `CONFIG_TOUCHSCREEN_EDT_FT5X06=m`（别家 edt 驱动），未移植 |
| 指纹 | 未移植（侧边电容式） |
| 摄像头 | 未移植 |

## 产物

| 文件 | 说明 |
| --- | --- |
| `work/ares/stock-base.dtb` | 从 ares boot.img 抽出的 SoC DTB |
| `work/ares/ares-board-0.dtb` / `.dts` | dtbo 里的板级 overlay |
| `work/ares/merged.dtb` / `merged.dts` | 叠加后的完整树 |
| `out-ares-min/` | 内核 + 极简 init（init 会镜像日志到 ttyGS0） |
| `out-ares-v1/` | 第一个构建 |
| `k40g/boot_a.img` | 原厂 Android 内核（回滚用） |
