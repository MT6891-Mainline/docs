# chopin 移植时间线

Redmi Note 10 Pro（MT6891 / "chopin"）从 Android 厂商内核换到主线 Linux 的完整过程，
按阶段排列。每一阶段只讲"什么时候、为什么、结果如何"，细节指向 `archive/` 里的原始文档。

> 想按**功能**看而不是按时间看：回到 [`README.md`](README.md) 的功能状态矩阵。
> 时间以文档最后写下的日期为准；阶段之间的顺序就是工作推进的顺序，
> 同一阶段内的顺序是"先发现问题 → 再定位 → 再修"。

| 阶段 | 时间 | 一句话 |
| --- | --- | --- |
| [0](#阶段-0让内核先能说话2026-10-01) | 2026-10-01 | 让内核真正把日志吐出来——后面所有工作都依赖它 |
| [1](#阶段-1第一次能开机2026-10-02--10-05) | 2026-10-02 ~ 10-05 | 7.2.8 主线第一次上机；确定改走厂商 `mediatek_v2` 显示栈 |
| [2](#阶段-2v2-厂商显示栈交付2026-10-10) | 2026-10-10 | `chopinport=v2ninetyfifth` 交付 |
| [3](#阶段-3交接与-wi-fi2026-10-18) | 2026-10-18 | 交接文档成型；Wi-Fi 打通 |
| [4](#阶段-4gpu-尝试与电源2026-10-21--10-23) | 2026-10-21 ~ 10-23 | GPU 尝试失败并定论；PMIC IRQ/挂起笔记；成果总结 |
| [5](#阶段-5迁到-73-音频-蓝牙2026-10-24) | 2026-10-24 | 基线 7.2.8 → 7.3；音频、蓝牙打通 |
| [6](#阶段-6传感器与自动旋转2026-10-25--10-27) | 2026-10-25 ~ 10-27 | 加速度计（ICM40607 / SCP）三轮推进 |
| [7](#阶段-7显示根因充电挂起定论2026-10-28--10-31) | 2026-10-28 ~ 10-31 | 花屏/underflow 真正原因；充电；挂起永久放弃 |
| [8](#阶段-8windows-on-arm-支线2026-11-01) | 2026-11-01 | UEFI 移植探路（支线，未成为主线） |
| [9](#阶段-9桌面-显示-指纹-rootfs2026-11-06--11-08) | 2026-11-06 ~ 11-08 | 能日常用的桌面；指纹 TA 上用户态；rootfs 与开机问题 |

---

## 阶段 0：让内核先能"说话"（2026-10-01）

移植的第一步不是驱动，而是**可观测性**：内核日志出不来，后面每一步都是盲改。

| 日期 | 文档 | 内容 |
| --- | --- | --- |
| 2026-10-01 | [`PLAN.md`](archive/PLAN.md) | 移植后续工作计划（P0/P1/P2 排期） |
| 2026-10-01 | [`NEXT.md`](archive/NEXT.md) | 找到"看不见"的真正根因：内核 `log_store` 驱动的 SRAM 头 |

**结果**：`dmesg` 可用。这是后面一切的前提。

---

## 阶段 1：第一次能开机（2026-10-02 ~ 10-05）

| 日期 | 文档 | 内容 |
| --- | --- | --- |
| 2026-10-03 | [`DELIVERY.md`](archive/DELIVERY.md) | Linux 7.2.8 主线移植的交付说明（第一版可用镜像） |
| 2026-10-03 | [`DSC-REFERENCE.md`](archive/DSC-REFERENCE.md) | 从 **LK 正在运行的实例**里读出 DSC 寄存器，作为黄金参考 |
| 2026-10-03 | [`V2-IMPORT.md`](archive/V2-IMPORT.md) | 决策：改用厂商 `mediatek_v2` 显示栈（自研 DRM 路线评估后放弃） |
| 2026-10-05 | [`CLEANUP-PLAN.md`](archive/CLEANUP-PLAN.md) | 内核仓库历史整理 |

**为什么改路线**：自研 DRM 在 DSC/DSI 上卡住；厂商栈虽大（73 个 C 文件、11 万行），
但**能亮**是第一优先级。

---

## 阶段 2：v2（厂商显示栈）交付（2026-10-10）

| 日期 | 文档 | 内容 |
| --- | --- | --- |
| 2026-10-10 | [`V2-HANDOVER.md`](archive/V2-HANDOVER.md) | `chopinport=v2ninetyfifth` 的完整交付文档 |

---

## 阶段 3：交接与 Wi-Fi（2026-10-18）

| 日期 | 文档 | 内容 |
| --- | --- | --- |
| 2026-10-18 | [`HANDOFF.md`](archive/HANDOFF.md) | 面向下个会话的总交接（27+ 轮历史索引） |
| 2026-10-18 | [`WIFI-PORT.md`](archive/WIFI-PORT.md) | MT6893 Wi-Fi（CONNAC2X2 SOC3_0 / conninfra）从 4.14 搬到主线 |

---

## 阶段 4：GPU 尝试与电源（2026-10-21 ~ 10-23）

| 日期 | 文档 | 内容 |
| --- | --- | --- |
| 2026-10-21 | [`GPU-PORT.md`](archive/GPU-PORT.md) | Panfrost / Mali-G77 移植——结论先行：**没打通** |
| 2026-10-23 | [`GPU-HANDOFF.md`](archive/GPU-HANDOFF.md) | GPU 交接（新会话读这一篇） |
| 2026-10-23 | [`PMIC-IRQ-NOTES.md`](archive/PMIC-IRQ-NOTES.md) | PMIC IRQ / 电源键笔记（音量键正常，电源键另论） |
| 2026-10-23 | [`SUSPEND-NOTES.md`](archive/SUSPEND-NOTES.md) | suspend 初探：`mtu3` gadget 让系统进不了 deep |
| 2026-10-23 | [`SUMMARY.md`](archive/SUMMARY.md) | 阶段成果总结（内核基线 7.2.8） |

---

## 阶段 5：迁到 7.3、音频、蓝牙（2026-10-24）

| 日期 | 文档 | 内容 |
| --- | --- | --- |
| 2026-10-24 | [`7.3-MIGRATION.md`](archive/7.3-MIGRATION.md) | 把 7.2.8 上的全部移植（音频/蓝牙/Wi-Fi/GPU/显示/DT）迁到 7.3 |
| 2026-10-24 | [`AUDIO-HANDOFF.md`](archive/AUDIO-HANDOFF.md) | 音频（无 3.5mm 口，外放链路） |
| 2026-10-24 | [`BT-HANDOFF.md`](archive/BT-HANDOFF.md) | 蓝牙（MT66xx / BTIF / connac2） |

---

## 阶段 6：传感器与自动旋转（2026-10-25 ~ 10-27）

| 日期 | 文档 | 内容 |
| --- | --- | --- |
| 2026-10-25 | [`SENSOR-HANDOFF.md`](archive/SENSOR-HANDOFF.md) | 自动旋转 / SCP 传感器 bring-up 目标 |
| 2026-10-26 | [`AUTO-ROTATION-STATUS.md`](archive/AUTO-ROTATION-STATUS.md) | 第二轮状态交接 |
| 2026-10-26 | [`HANDOFF-ACCEL-2026-10-26.md`](archive/HANDOFF-ACCEL-2026-10-26.md) | 加速度计（ICM40607 / SCP）协议与固件逆向 |
| 2026-10-27 | [`HANDOFF-ACCEL-2026-10-27.md`](archive/HANDOFF-ACCEL-2026-10-27.md) | 上一份的推进版 |
| 2026-10-27 | [`REVIEW-ACCEL-2026-10-27.md`](archive/REVIEW-ACCEL-2026-10-27.md) | 驱动逻辑复核 |

---

## 阶段 7：显示根因、充电、挂起定论（2026-10-28 ~ 10-31）

这一阶段把三件"玄学"变成了结论。

| 日期 | 文档 | 内容 |
| --- | --- | --- |
| 2026-10-28 | [`DRM-CLOCK-GAP.md`](archive/DRM-CLOCK-GAP.md) | **underflow / 花屏的真正原因**（不是"DRM 不完善"这种笼统说法） |
| 2026-10-30 | [`V2-DRM-AUDIT.md`](archive/V2-DRM-AUDIT.md) | 厂商 DRM 审计：73 个 C 文件、110,610 行 |
| 2026-10-30 | [`GPU-PERF.md`](archive/GPU-PERF.md) | GPU 固定 350 MHz，无法安全提频 → 保持现状 |
| 2026-10-31 | [`CHARGING.md`](archive/CHARGING.md) | 充电与电量（`capacity=65535` 那类问题的定位） |
| 2026-10-31 | [`SUSPEND-ANALYSIS.md`](archive/SUSPEND-ANALYSIS.md) | **⛔ 禁止在这台设备上继续测试挂起**（会烧屏） |
| 2026-10-31 | [`STATUS.md`](archive/STATUS.md) | 整体状态盘点 |

---

## 阶段 8：Windows on ARM 支线（2026-11-01）

| 日期 | 文档 | 内容 |
| --- | --- | --- |
| 2026-11-01 | [`UEFI-PORT.md`](../uefi/archive/UEFI-PORT.md) | Mu-Silicium UEFI 移植（参考 MT6789Pkg） |
| 2026-11-01 | [`CHOPIN-WOA-HANDOFF.md`](../uefi/archive/CHOPIN-WOA-HANDOFF.md) | WoA 交接（固件基线 r81） |

**结果**：探路性质，未成为主线（主线仍是 Debian + denial 桌面）。

---

## 阶段 9：桌面、显示、指纹、rootfs（2026-11-06 ~ 11-08）

| 日期 | 文档 | 内容 |
| --- | --- | --- |
| 2026-11-06 | [`BT-AUDIO.md`](archive/BT-AUDIO.md)、[`BT-HID.md`](archive/BT-HID.md) | 蓝牙音频与 HID |
| 2026-11-06 | [`KVM-EL2.md`](archive/KVM-EL2.md) | KVM / EL2 可行性 |
| 2026-11-07 | [`DTS-RESTRUCTURE.md`](archive/DTS-RESTRUCTURE.md) | 设备树重构 |
| 2026-11-07 | [`DROIDLOOM.md`](archive/DROIDLOOM.md) | 桌面（droidloom / denial 路线） |
| 2026-11-07 | [`MAINLINE-DRM.md`](archive/MAINLINE-DRM.md) | 主线 DRM 现状 |
| 2026-11-07 | [`VDEC-PORT.md`](archive/VDEC-PORT.md) | 视频解码 |
| 2026-11-08 | [`KERNEL-7.3.3-PORT.md`](archive/KERNEL-7.3.3-PORT.md) | 7.3.3 基线移植 |
| 2026-11-08 | [`FINGERPRINT-PORT.md`](archive/FINGERPRINT-PORT.md) | 指纹移植总览 |
| 2026-11-08 | [`TA-IN-USERSPACE.md`](archive/TA-IN-USERSPACE.md) | 厂商 Goodix trustlet 在用户态跑（M1.42–M1.53） |
| 2026-11-08 | [`HANDOFF-2026-11-08.md`](archive/HANDOFF-2026-11-08.md) | 指纹交接 |
| 2026-11-08 | [`CHARGING-STATUS.md`](archive/CHARGING-STATUS.md) | 充电状态收尾 |
| 2026-11-08 | [`DEBIAN-ROOTFS.md`](archive/DEBIAN-ROOTFS.md) | Debian 13 rootfs 制作与启动 |
| 2026-11-08 | [`BOOT-SESSION-HANDOFF.md`](archive/BOOT-SESSION-HANDOFF.md) | 开机/会话问题（未结案，附取证方法） |

这一阶段的两条主线：

1. **桌面可用**：DRM/fbdev 修到能点亮、能进 denial 桌面；
2. **指纹**：走"厂商 TA 在用户态跑"的路线（`ta_host32` + MicroTrust stub），
   匹配在 TA 里完成，采集通道仍在攻坚。

---

## 附：启动链

开机链路本身已经单独成文：[`rootfs/BOOT-CHAIN.md`](archive/BOOT-CHAIN.md)
（LK → 内核早期 → initramfs PID 1 → systemd → getty → denial → 关机）。

## 附：文档目录对照

| 主题 | 目录 |
| --- | --- |
| 历史归档（早期交付/计划/总结） | [`archive/`](../archive/) |
| 构建与内核基线迁移 | [`build/`](../build/) |
| 显示 / DRM / DSC / UEFI | [`display/`](../display/) |
| GPU | [`gpu/`](../gpu/) |
| 电源 / 充电 / 挂起 / KVM | [`power/`](../power/) |
| 音频 | [`audio/`](../audio/) |
| 连接（Wi-Fi / 蓝牙） | [`connectivity/`](../connectivity/) |
| 传感器 / 自动旋转 | [`sensors/`](../sensors/) |
| 桌面环境 | [`desktop/`](../desktop/) |
| 媒体（VDEC） | [`media/`](../media/) |
| 指纹 | [`fingerprint/`](../fingerprint/) |
| rootfs / 启动 / 会话 | [`rootfs/`](../rootfs/) |
| USB / OTG | [`usb/`](../usb/) |
| 当前状态 | [`status/`](../status/) |
