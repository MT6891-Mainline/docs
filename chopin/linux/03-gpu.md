# 03 · GPU（Panfrost / Mali-G77）

能跑 3D，**但锁在 350 MHz，没有 DVFS**。这是当前接受的现状，不是待办。

## 现状

| 项 | 状态 |
| --- | --- |
| Panfrost 驱动 | ✅ 工作 |
| 频率 | ❌ 固定 350 MHz（硬件能到 810） |
| DVFS | ❌ 没有（缺 OPP/regulator 集成） |
| 提频尝试 | 三次，全部失败（[`archive/GPU-PERF.md`](archive/GPU-PERF.md)） |

## 为什么没有 DVFS

GPU 的供电/时钟由 MFG 域的 regulator + 时钟树控制，厂商实现把 OPP 表放在 SCP/电源固件里。
主线没有那条通路，硬写寄存器提频会掉压/挂死。

## 历史

* [`archive/GPU-PORT.md`](archive/GPU-PORT.md) — 最初移植：结论"没打通"，仍走 Mesa 软渲。
* [`archive/GPU-HANDOFF.md`](archive/GPU-HANDOFF.md) — 交接（新会话读这篇）。
* [`archive/GPU-PERF.md`](archive/GPU-PERF.md) — 提频分析：**三次尝试、结论保持 350 MHz**。
