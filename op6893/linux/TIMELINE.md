# op6893 Linux 移植时间线

| 阶段 | 验证与记录 |
| --- | --- |
| 2026-11-02，attempt 1 | fastboot 接受镜像，主线执行到 `setup_arch`；从恢复后的早期日志确认崩溃点，还未到用户态。[第一次运行](archive/evidence/attempt-1.md) |
| attempt 25–39 | 验证 GPIO/PMIC 按键、USB ACM/ECM、八核和 UFS 上的外置 rootfs。[按键](archive/evidence/attempt-29.md) · [启动与 SSH](archive/evidence/attempt-39.md) |
| attempt 42–97 | 逐步验证时钟、热传感器及 CONN 资源；早期 CONN 结果主要是读回和恢复。[外设状态](PERIPHERAL-STATUS.md) · [无线前置条件](archive/WIFI-INITIALIZATION.md) |
| 2026-11-10，attempt 98–109 | 用厂商连接栈打通 Wi-Fi 和蓝牙；USB 之外有了 Wi-Fi SSH 路径。[连接状态](CONNECTIVITY-STATUS.md) |
| attempt 110–124 | 依次验证早期显示、触摸、充电与电池估算，随后在 Mali-G77 上用 Panfrost/Mesa 渲染。[显示](archive/evidence/attempt-110-111.md) · [触摸](archive/evidence/attempt-115.md) · [电池](archive/evidence/attempt-118.md) · [GPU](archive/evidence/attempt-124.md) |
| 2026-11-14 至 11-17 | 从 MD1 供电、SCP 和 CCIF 资源预检推进到首次放行，随后验证 FS/RPC、握手和 CCISM。[基带计划](archive/MODEM-PLAN.md) · [放行记录](archive/MODEM-BOOT-RELEASE.md) |
| 2026-11-18 至 11-19 | 接通 DPMAIF 数据面，实测 LTE/NR 与双栈；核对 PDP 激活给出的通道分配。[数据面](archive/MODEM-BOOT-RELEASE.md) · [SIM/RAT](MODEM-SIM-RAT.md) |
| 2026-11-20 起 | 在 `cache` 做写盘试验后，直接从主线更新 `boot` 并验证重启；继续调查同一次 AP 启动中的基带热启。[启动分区](BOOT-PARTITION.md) · [SCP/MD 恢复](archive/MODEM-SCP-RESET-RESTORE.md) |
| 2026-11-22，21651 | 从临时 cache 根切换到 userdata ext4，验证 systemd 和自选根文件系统的 init 交接；从 Linux 请求重启到 LK fastboot。[启动状态](21651-STATUS.md) · [自选根文件系统](21651-USERDATA-BOOT.md) |
| 2026-11-22，7.3.3 共享树 | 21651 恢复 Wi-Fi、蓝牙扫描和 Panfrost，修复扫描缓冲同步后复测无黑色毛刺；基带诊断镜像通过 HS2 和 AT，但未注册。[外设](21651-STATUS.md#连接与其他外设) · [基带](21651-MODEM.md) |
| 2026-11-22，USB 与充电 | 21651 的可选 OTG 镜像验证读盘、带电拔除和回接 USB SSH；新增电量计与输入遥测，验证 250 mA 限流并发现低档误差。[21651 实测](21651-STATUS.md) · [充电待办](BATTERY-TODO.md) |

[逐轮实测与日志](archive/evidence/) · [归档索引](archive/README.md)
