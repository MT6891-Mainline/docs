# op6893 · Linux 路线

20615 和 21651 均已从 LK 启动到 Debian，板级镜像与实机验收分别记录。
21651 当前测试基线是 7.3.3 共享树的 `fd5d000e481b` 镜像；20615 一栏
保留原 7.3.0 的实测结果。20615 未在这轮做整机回归，Chopin 也仅做了
相关对象的构建检查。

| 功能 | 20615（原开发机） | 21651（当前主要开发机） |
| --- | --- | --- |
| 启动、UFS、根文件系统 | 八核在线，`cache` 上的 SquashFS 只读运行；已验证[从 Linux 更新 boot 后重启](BOOT-PARTITION.md)。 | 从 `userdata` ext4 启动 [Debian/systemd](21651-SYSTEMD.md)；[自选根文件系统接口](21651-USERDATA-BOOT.md)已验证 BusyBox 与 systemd 入口。 |
| USB | ACM 串口与 ECM/SSH 可用；主机模式未验收。 | ACM/ECM 和拔插后 USB SSH 自动恢复；可选 OTG 镜像已验证 USB 2.0 扩展坞后 U 盘只读、带电拔除关断和回到从机；USB 3.x、长期带载未验收。 |
| 显示、触摸、GPU | LK 显示缓冲、simpledrm、FT3518 与 Panfrost/Mesa 已实测。 | 已实测 FT3518 与两个音量键；Panfrost 扫描缓冲同步修复后黑色毛刺不再复现。两机均未验收完整面板接管、GPU 调频和运行时挂起。 |
| 电池、充电、温度 | MT6360 充电与电压遥测；电量为电压估算，负载下明显偏低。 | BQ28Z610 电量计、MP2650 输入遥测、五个 LVTS 温区有实测。250 mA 档有效；自动限流、USB 挂起取电、PD/PPS 与充电温控未验收。 |
| Wi-Fi / Bluetooth | 见[连接状态](CONNECTIVITY-STATUS.md)：开放 Wi-Fi 和 BlueZ 扫描、配对与 PAN 已实测；蓝牙音频未验收。 | 开放 Wi-Fi 上 IPv6 链路本地 SSH 可用，已实测经典与低功耗蓝牙扫描；DHCP、网络认证、蓝牙配对与 PAN 未验收。 |
| 蜂窝 | [双卡 LTE 注册、LTE/NR 双栈短测和受控重连](MODEM-SIM-RAT.md)已实测；热启、语音、短信、IMS 与 GNSS 未验收。 | [基带诊断镜像](21651-MODEM.md)通过 HS2、CCISM、AT 往返与 DPMAIF 环初始化；正式 60 秒窗口仍超时，单卡注册、PDP 激活与数据收发尚未通过。 |

逐项证据见[外设支持状态](PERIPHERAL-STATUS.md)、[21651 支持状态](21651-STATUS.md)、
[电池待办](BATTERY-TODO.md)和[图形待办](GRAPHICS-TODO.md)。移植过程见
[时间线](TIMELINE.md)；[归档索引](archive/README.md)收录早期方案、协议细节和实测日志。
