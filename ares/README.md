# ares — Redmi K40 Gaming（MT6893）

同 SoC 的第二台机器（chopin 是 Redmi Note 10 Pro）。移植刚开始。

> **2026-11-15**：失败点已定位到 **LK**（不是内核）：LK 在交出控制权前
> `panic: ASSERT boot_info.c:616 g_boot_info.hdr_loaded`，详见
> [`linux/STATUS-2026-11-15.md`](linux/STATUS-2026-11-15.md)。

| 路线 | 目录 | 状态 |
| --- | --- | --- |
| Linux | [`linux/`](linux/README.md) | ⏳ 内核还没被执行：LK 在加载 boot image 后 panic（见 STATUS-2026-11-15） |

## 与 chopin 的差异（已确认）

| 部件 | chopin | ares |
| --- | --- | --- |
| 面板 | `k10a-36-02-0a-dsc-vdo`（视频模式） | **`k10-42-02-0a-dsc-cmd`（命令模式，120 Hz）** |
| 触摸 | Novatek NT36672E @ spi0 | **Focaltech @ spi6** |
| 电源 | — | 多 `mt6315_3` / `mt6315_6` 两路 buck、`tboard_thermistor6` |
| 指纹 | Goodix（屏下） | 侧边电容式（未移植） |

SoC、PMIC、UFS、USB、Wi-Fi/BT 部分与 chopin 相同——`boot.img` 里的 SoC DTB
与 chopin 的**逐字节相同**（`f9e4af23…`），板级信息在 **dtbo 分区**里。

## 相关仓库

| 仓库 | 内容 |
| --- | --- |
| `git/chopin-linux-7.3` 的 **`ares` 分支** | ares 的 DTS 与面板驱动 |
| `git/docs` | 本仓库 |
