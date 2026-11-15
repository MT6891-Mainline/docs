# 07 · 传感器与自动旋转

**结论：加速度计 ICM40607 挂在 SCP（传感器协处理器）后面，不在 AP 的 I²C 总线上。**
AP 侧直接读不到，必须走 SCP 的协议栈——而那条传输栈 mainline 里完全没有。

## 状态

| 项 | 状态 |
| --- | --- |
| SCP 本身在跑 | ✅（LK 启动它，`chopin-scp.service` 加载固件） |
| AP ↔ SCP 通信 | ⚠️ 基础通道有，传感器通道未打通 |
| 加速度计数据 | ❌ 拿不到 |
| 自动旋转 | ❌（因此桌面上是手动锁定） |

## 分阶段计划（未完成）

1. 在 SCP 固件里确认 ICM40607 的驱动存在并初始化；
2. 打通 AP 侧的 sensor 通道（厂商叫 `sensorhub`/`sensor_comm`，见串口日志里的
   `sensor_comm disabling sensor comms after 12 consecutive failures`）；
3. 在 AP 侧实现一个标准的 IIO 加速计设备，喂给桌面。

## 文档

* [`archive/SENSOR-HANDOFF.md`](archive/SENSOR-HANDOFF.md) — 目标与传输栈
* [`archive/AUTO-ROTATION-STATUS.md`](archive/AUTO-ROTATION-STATUS.md) — 第二轮状态
* [`archive/HANDOFF-ACCEL-2026-10-26.md`](archive/HANDOFF-ACCEL-2026-10-26.md) — 协议/固件逆向
* [`archive/HANDOFF-ACCEL-2026-10-27.md`](archive/HANDOFF-ACCEL-2026-10-27.md) — 推进版
* [`archive/REVIEW-ACCEL-2026-10-27.md`](archive/REVIEW-ACCEL-2026-10-27.md) — 驱动逻辑复核
