# 12 · USB

## 设备模式（gadget）：可用

USB 串口控制台（`g_serial` → `/dev/ttyGS0`）是调试的主要通道：插上就是 root shell
（getty 已改成 `bash -l`，见 [`01-boot.md`](01-boot.md)）。

启动时日志里会有两条**误导性报错**，都不是故障：

| 日志 | 真相 |
| --- | --- |
| `UDC core: g_serial: couldn't find an available UDC` | 1.58 s 时 mtu3（1.90 s）还没注册，随后绑定成功 |
| `mt6360-tcpc: Failed to register tcpci port` | 第一次探测 `-EPROBE_DEFER`（要等 mtu3 的 role-switch），内核静默重试成功；`/sys/class/typec/port0` 就是这个设备注册的 |

后者已在内核里修成"只对真错误打印"（提交 `1958eff`）。

## 主机模式（OTG）：未完成

| 阶段 | 现象 |
| --- | --- |
| LK → 内核早期 | OTG 灯**亮**，`xhci-mtk` 认到设备 |
| 进 Linux 后 | 灯灭（VBUS 被关） |
| 修了角色/供电后 | 灯**闪烁**——设备反复重试枚举 |

**结论**：供电链路已通，**端口使能失败**，属主线 MT6893 USB host 的空白区。
细节：[`archive/OTG.md`](archive/OTG.md)。

## Type-C

`mt6360-tcpc` 注册成功，`/sys/class/typec/port0` 可用（`port_type=[dual]`）。
`usb_type_c@4e` 在 DT 里是 `disabled`（厂商子模块，未启用）。
