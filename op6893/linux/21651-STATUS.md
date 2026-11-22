# 21651 主线支持状态

21651 与 20615 共享内核源码，但使用独立板级设备树和启动镜像。
两机的对照见[功能矩阵](README.md)。下文按镜像记录实测；原 7.3.0
表格保留当时的结果。

## 7.3.3 共享分支对齐

截至 2026-11-22，开发分支 `7.3.3-mt6893-oplus-op6893` 的记录点为
`eaffb21fe`，对应实测镜像 `fd5d000e481b`。该镜像从 UFS userdata ext4
进入 Debian/systemd，USB 和 Wi-Fi SSH 均可用。镜像采用可选
`typec-otg` 配置（GPIO I²C5、PD 禁用）；基带在此镜像中关闭，
另用诊断镜像验收。

早期对齐检查点为 `65238ba80`。
外设镜像 `cb13009fe3cf` 的 MT6359 连续 16 次读回通过，电池 ADC 约
4.30 V，五个 LVTS 温区有读数，PMIC 按键、触摸和 GPIO 按键均注册。
本轮未采集实际按键和触摸事件。

## 21651 电量计

镜像 `a0ebea39e11a` 用主线 `bq27xxx` 驱动读取 I²C7 上地址 `0x55`
的电量计。原厂运行设备树沿用 `bq27541` 节点名，但设置了
`qcom,batt_bq28z610 = <1>`。板级设备树用 GPIO124（SCL）和 GPIO125
（SDA）做开漏 I²C，不挂载同总线的快充芯片；构建关闭电量计 NVM 更新。

实机在开发机供电时，`/sys/class/power_supply/bq28z610-0` 多次报告
89%、电池包约 8.56 V、充电电流约 98–103 mA、35.2 °C，状态为 `Charging`。
`charge_full` 为 1413 mAh，是电量计的当时读数，尚未做容量校准或续航验证。
USB 和 Wi-Fi SSH 均通过，日志见 `21651/dist/a0ebea39e11a/battery-*.log`。
同镜像实测开发机 USB 线拔除和复接：TCPM 在 181.481 秒检测到断开，
电量计在下一条采样（182.55 秒）报告 `Discharging`，后续放电电流为
194–288 mA。203.021 秒重新检测到 VBUS，204.71 秒恢复 `Charging`，
电池充电电流约 93 mA。MP2650 的 `0x13` 随之从 `0x3b` 变为 `0x00`、
再恢复 `0x3b`。接回后 TCPM 已通过 `power_supply` 报告 `online=1`；
MP2650 输入 ADC 约 4.60–4.63 V，配置寄存器未改动。`0x00=0x12`
按厂商换算为第一阶段输入限流 900 mA，这个配置值不能作为实际 USB
输入电流或端口供电能力。有效输入限流和温控尚未验收。

Wi-Fi SSH 全程可用。复接时 USB gadget 已枚举，但 ECM 一度无法通信。
通过 Wi-Fi 重新绑定 configfs 的 UDC 后，USB SSH 恢复，无需重启。
这只是恢复办法，尚未定位 ECM 复接失败的根因。日志见
`21651/dist/a0ebea39e11a/battery-cable-*.log`。

后续插拔在 `a0ebea39e11a` 上未重现 ECM 无法通信，但确认了另一个问题：
TCPM 检测到 CC 和 VBUS 断开后，gadget 仍报告 `configured`，端点仍占用。
MTU3 为该板强制内部 VBUS 指示，因此现在从 Type-C 角色切换通知 UDC 结束会话。
同时修复了 UDC 启动前处理 VBUS 通知时只更新软件状态、导致绑定时跳过 D+ 上拉的问题。

修正版 `e3b9d6ebf446` 已在插着开发机线的情况下正常启动，USB 和 Wi-Fi SSH 均可用。
三轮软件断开以及一次实体拔线都使 gadget 降为 `not attached`、占用端点降为 0；
接回后 USB SSH 自动恢复，无需重绑 UDC。见 `21651/dist/e3b9d6ebf446/ecm-*.log`。
同一镜像也通过了下文的 OTG 读盘回归；此前偶发的 ECM 故障尚需长期观察。

在 `mt6893-direct` 目录下，可通过已验证的 Wi-Fi SSH 以 root 运行
`python3 -u - --seconds 90`，并将 `21651/battery-cable-sample.py`
从标准输入送到设备。脚本只读取电量计、TCPM 和指定 MP2650 寄存器。

镜像 `fd5d000e481b` 为 MP2650 新增只读 `hwmon` 接口。查找 `name` 为
`mp2650` 的 hwmon 设备，`in0_input` 以 mV 报告输入电压，`curr1_input`
以 mA 报告输入电流。开发机供电时，十次连续采样约为 4500–4550 mV、
863–875 mA，与相邻原始寄存器样本的换算一致。这里是充电芯片输入端，
与 BQ28Z610 的电池电流是不同测点。充电参数未改动。
同镜像实测 OTG：71 个升压样本的两个输入属性均返回 `ENODATA`，
U 盘直接只读 64 KiB 成功。升压约 18 秒后，拔线触发 CC 断开，约 21 ms
后 TCPM 报告 VBUS 关闭，未触发 30 秒保护。接回开发机后，输入读数和
USB SSH 自动恢复。日志见 `21651/dist/fd5d000e481b/mp2650-*.log`。

开发机这条链路被 TCPM 识别为默认 Rp，且本驱动没有
为默认 Rp 实现 `get_current_limit`，因此 `current_max=0` 表示这里没有
可用的电流额度。MP2650 的两个输入限流寄存器仍为 `0x12`（各 900 mA）。

开发机基线已通过主机端的 USB 描述符确认：高速 ACM/ECM 配置的
`MaxPower` 为 250 mA；同时 TCPM 上报 `current_max=0`，MP2650 两级
限流均为 900 mA，输入 ADC 约 870 mA。检查代码发现 composite gadget
在枚举成功、挂起、恢复、复位和断开时调用 `usb_gadget_vbus_draw()`，但
MTU3 未实现这个回调。改变 `MaxPower` 只会改变描述符，不能限制 MP2650。

同镜像 `fd5d000e481b` 在固定受电、PD 禁用时完成 C-to-C 充电器对照。
TCPM 从 CC 识别到 5 V / 3 A 的电流声明（`current_max=3000000`），
没有建立 PD/PPS 合同。充电器接入的 23 个样本中，MP2650 输入 ADC 为
4.85–5.05 V、863–900 mA，两级限流保持 900 mA，gadget 未枚举。
接回开发机后，`current_max` 回到 0，gadget 恢复 `configured`，
USB SSH 自动恢复。日志见 `21651/dist/fd5d000e481b/charge-source-after-power-key*.log`。

同镜像下用一次性模块验证了 MP2650 的输入限流。开发机保持默认 Rp，
USB 高速 gadget 全程保持 `configured`。结果如下：

| MP2650 设置 | 芯片输入 ADC |
| --- | ---: |
| 原配置，两级均为 900 mA（`0x12`） | 约 880 mA |
| 两级均为 250 mA（`0x05`） | 225–231 mA |
| 两级均为 100 mA（`0x02`） | 163–175 mA |
| 维持 100 mA 设置，并置位 `LEARN_EN`（`0x08` 的 bit 3） | 25 mA |

250 mA 档实测低于 gadget 声明的 250 mA，但在此次测试负载下电池已转为
放电。100 mA 档不能作为可靠的 100 mA 上限。`LEARN_EN` 明显降低了取电，
但 USB 总线侧尚无独立电流实测，不能将芯片的 25 mA 读数视为已满足 USB
挂起额度，也不能将其当作零点偏移扣除。模块已卸载，原值 `12/16/12`
（`0x00/0x08/0x0f`）已读回确认，USB SSH 正常。日志见
`21651/dist/fd5d000e481b/charge-input-limit-trial*.log`。

实现时区分两种额度来源：TCPM 已识别的 Type-C 电流声明，以及默认 Rp
时 USB gadget 状态对应的额度。前者要随 CC 变化和断开更新，后者要处理
枚举前、配置后与挂起。MP2650 的两级限流需要在同一把锁下更新，并和
OTG 升压互斥；不要在 gadget 回调里直接发 I²C 事务。已枚举的 250 mA
路径可用作自动限流的第一条验证路径。枚举前与挂起需要单独验证输入隔离：
用总线侧电流表区分实际余量和芯片 ADC 读数，并确认隔离不影响 TCPC 的
VBUS 检测或 gadget 枚举。测试时同时核对两级寄存器与输入 ADC。
`21651/battery-cable-sample.py` 已补齐两级配置、`current_max` 以及
UDC 状态和速度的时间序列。

## 连接与其他外设

基带的 120 秒诊断镜像已通过 SCP READY、MD1、CCCI/CCISM 握手、AT 往返和
DPMAIF 下行环初始化；60 秒配置仍在 HS2 超时。漫游卡能识别但未注册；
电信卡两槽的有效复测均未通过识卡，数据面尚未验收。详见[基带记录](21651-MODEM.md)。
Wi-Fi 镜像 `64e35d01098c` 已在 21651 上验证：从只读 nvdata
加载本机校准，从 proinfo 读取的出厂地址与 wlan0 一致；扫描、关联开放热点
和 IPv6 链路本地公钥 SSH 均通过，USB SSH 同时可用。尚未验收 DHCP、网络认证
和互联网访问。无线驱动修复了 PMIC 包装器寄存器映射取错，以及禁用系统挂起时
唤醒源为空导致的非法访存。测试使用 `21651/build.sh userdata wifi off systemd`，
进入系统后手动加载校准并启动 Wi-Fi。

联合测试镜像 `166acad4be1d` 已在 21651 验证：
- 蓝牙使用标准 HCI 初始化，启动后控制器报告版本 12、厂商 70；
  出厂地址与 proinfo 一致。最终镜像收到 78 条低功耗发现事件和 1 条经典蓝牙发现，
  测试期间 Wi-Fi 可用，经典蓝牙测试是经无线 SSH 发起的。配对、PAN 和反复开关尚未验收。
- Panfrost 通过 OP6893 的六个 MFG 电源域启动，识别 Mali-G77 MC9，
  首次运行 kmscube 完成 250 帧，未见任务超时或故障。启用的首次任务前复位
  复用了 Chopin 的处理；GPU 冷却设备已注册，但未重测完整频率、电压与温控。
- 用户后来在 `166acad4be1d` 上观察到黑色毛刺。同机 `kmsflat` 复现
  138/180 帧有错误像素。修正版 `fe2f0f6e654d` 补回原 7.3.0 的
  GEM 导出缓冲区缓存同步，实测 500 帧没有错误像素。
  `kmscube` 完成 1199 帧，未见 GPU 任务超时或故障；用户目视复测确认黑色毛刺消失。
- Chopin 的 Panfrost 对象已按其现有配置重新构建；20615 只验证了早期启动对象。
  本轮没有这两台设备的真机回归结果。

21651 的可选 Type-C 受电测试镜像 `8cb1c8c5c97c` 通过 boot 写后校验，从
userdata 进入 systemd。构建命令为
`21651/build.sh userdata radio graphics systemd normal off typec-sink`，指向共享内核树。
MT6360 TCPC 使用原厂 GPIO25 中断，MTU3 保持高速从机，关闭 PD 和 VBUS 输出。
开机时若已接 USB 线，ECM 可用，但 TCPM 未登记对端；拔下并翻面插回后观察到
CC 和 VBUS 中断、`port0-partner`、`orientation=reverse`、`SNK_READY` 及
USB SSH 恢复。Wi-Fi IPv6 SSH 在拔插期间保持可用。

修正版 `08f0bc6a7ce3`（TCPM 提交 `a0bfeba1d`）在固定受电端复位结束、
重新设为 Rd 后主动重读 CC。真机保持同一根线插着重启，约 1.73 秒进入
`SNK_READY`，登记 `port0-partner` 并报告 `orientation=reverse`；
ECM 自动恢复。

Chopin 的 MT6360 TCPC、MTU3 和 xHCI 可作共用基础。21651 原厂运行 DTB
在 I²C11 声明 MP2650（0x5c，GPIO133）和独立 divider（0x68），并声明双电芯。
厂商驱动在双电芯、未使用双向电荷泵时走 buck-boost OTG 分支，驱动
CHGVIN（GPIO9）为高电平以隔离原输入。MP2650 驱动还控制 GPIO133 和芯片
内部升压。仍需原厂真机记录确认是否走该分支，以及 divider 是否参与。
不能照搬老机的 BQ2597x 或 Chopin 的 MT6360 升压路径。

镜像 `465e4da5852d`（提交 `241a270c5`，构建选项 `typec-drd-probe`）
以 GPIO126/SCL 和 GPIO127/SDA 的开漏 I²C 访问原厂充电总线，临时避开
I²C11 原厂 FIFO 模式。真机在 `0x5c` 读到稳定的配置字：
`0x06=0x05`、`0x07=0x14`、`0x08=0x16`、`0x09=0x4c`。
状态字 `0x13` 插线时为 `0x0f`，拔线后三次读数均为 `0x00`，重新插线后三次
均为 `0x3b`；MT6360 同时报告 VBUS 恢复，ECM SSH 重新可用。所有样本的
`0x08[5]`（OTG 使能）均为 0。没有写充电寄存器，也没有操作 GPIO9/133。
这验证了芯片通信和输入状态随插拔变化；单独的 `0x13` 读数不能作为测量 5 V
输出的依据。重新插线后又通过 USB SSH 复读一次，仍为 `0x3b`，Type-C
报告 `device` / `sink`。

本地 4.19 参考源码（`kernel_oddo_mt6893` 的 `9e2043b0b`）和 6.6
参考源码中的相关路径一致：extcon 分别响应供电与数据角色事件。buck-boost
供电分支先停充，再调用 MP2650 使能（GPIO133 拉高，最后设置芯片的 OTG
位），然后把 CHGVIN（GPIO9）拉高。关断时先拉低 GPIO133 并清除 OTG 位，
再拉低 GPIO9。该分支没有直接调用 divider 驱动；还不能排除它在别处参与。

原厂验收应在同一台 21651 上抓一次 OTG 插入和拔除的日志，核对
`source vbus = 5000mv` / `0mv`、`typec vbus_on`、`chgvin_gpio`
及 xHCI 枚举；若日志无法看到成功调用，可用相应驱动符号的函数跟踪。
USB ADB 会随拔线断开，可在复接后再读本次启动的内核日志。当前
`userdata` 是 Linux 的 ext4 根分区，不能只刷旧 boot 就把 Android
当作可直接回退的采样环境。
同时用 USB‑C 电压表或总线供电外设确认实际输出，并核对拔除后的关断。

实验镜像 `c6e054125acd`（构建选项 `typec-otg`，源代码分支提交
`be535e719` 及前置提交）在 21651 上把 MP2650
作为 MT6360 TCPC 的外部 VBUS regulator，使用已验证的 GPIO I²C 总线。
TCPM 在切换到主机前开启该供电，停止供电时清除 OTG 位并恢复充电和看门狗的
原有使能设置；USB PD 仍关闭。镜像写入后读回 SHA256 一致。带开发机线启动
时，TCPM 进入 `SNK_READY`，端口自动成为受电从机，USB 和 Wi-Fi SSH 均可用，
且 `usb-vbus` 为 disabled、使用计数为 0。插入由手机供电的实际 USB 外设后，
还要验证主机枚举和拔除关断。原厂 Android 的运行时记录尚未补采。

首轮插拔发现 I²C 控制器通过 OF 子节点查找误拿到了 TCPC 的 `vbus-supply`，
导致访问控制器寄存器时也可能开启升压。修正版 `5250144ebaa5`
（提交 `ec1ba9f9c`）改用独立的 `vbus-source-supply`。实测拔掉开发机线时
升压保持关闭；插入 OTG 外设后 TCPM 进入 `SRC_READY`，MT6360 检测到 VBUS，
xHCI 注册 USB 2.0 根集线器，但没有枚举外设，也没有实测电压。
拔除外设后仍停在供电主机状态。通过 Wi-Fi 强制切换到受电角色后，MP2650
报告 disabled、使用计数为 0，但随后 TCPC 的 VBUS 转换返回 `-110`，USB
没有恢复。需分别采集 MT6360 的 I²C 传输和 CC/IRQ 状态，以及 xHCI 端口连接
状态，定位尚未通过的环节。已写回 `465e4da5852d`，重启后 Wi-Fi 和 USB SSH
均已确认可用。

诊断镜像 `a5a4a31841d2`（`OP6893_OTG_TRACE=1`）使用
`21651/otg-trace.sh` 同步采集 TCPC I²C、IRQ 和 xHCI 端口。进入
`SRC_READY` 后，xHCI 的 `PORTSC` 确实置位 CCS，停在 Polling 且
未识别速度，外设没有完成枚举。拔除阶段 GPIO25 的 TCPC 中断仍到达
（502.104 秒），但 I²C5 读取 MT6360 的 ALERT（地址 `0x4e`，寄存器
`0x10`）反复返回 `-110`。TCPM 因此没有完成拔除处理。独立的 15 秒
诊断关断于 504.552 秒成功关闭 MP2650，之后 TCPC 读取仍超时。已回退
到 `465e4da5852d` 并验证 Wi-Fi SSH。原始记录在
`21651/dist/a5a4a31841d2/otg-retest{,-tcpm}.log`。下一步优先验证
I²C5 的传输及恢复路径，同时用 xHCI 中断和端口状态定位枚举失败。

对照镜像 `ff7dcdf484ef`（提交 `4edf6434b`，构建时同时设置
`OP6893_OTG_GPIO=1 OP6893_OTG_TRACE=1`）用 GPIO202/203 开漏 I²C 访问
同一颗 MT6360，绕开原生 I²C5、其时钟和 APDMA。拔除时 ALERT 读取成功，
TCPM 处理了 CC 断开，MP2650 在 15 秒保护触发前自动关闭，xHCI 退出。
接回开发机后进入 `SNK_READY`，USB SSH 恢复。这个对照将超时范围缩小到
原生 I²C5 路径及其配置，尚未区分时钟、DMA 与时序。外设仍在
`Polling`，没有识别速度或完成枚举；采集脚本已补上 xHCI IRQ、事件环
与运行时电源状态。日志位于 `21651/dist/ff7dcdf484ef/otg-gpio-*.log`。
设备保留此镜像，运行时端口设为固定受电。

随后用同一组实物（手机 → 扩展坞 → U 盘）比较默认设置与禁用
自动挂起（`usbcore.autosuspend=-1`）。对照轮中控制器和根集线器
均处于 active，仍未枚举。端口快照有 CCS，但一度停在 Polling、
速度字段为 0；采样期间 IRQ 301 计数和事件环读指针未前进，
`IMAN.IP` 未置位。内核日志另记录分配设备槽位超时，随后中止
命令环也超时。现有证据不足以判定 GIC 丢中断。
`ERDP=0x109942000` 高于 4 GiB，待对照低地址 DMA。注册时还有
`invalid index 59`：当前精简时钟提供者未导出 xHCI 时钟引用，
但复查时原厂对应的 INFRA 门及 TOP 时钟均为开启状态，不能据此
认定时钟是枚举失败的原因。采集已退出，自动挂起值还原为 2，
端口已设回固定受电且升压关闭；该轮接回开发机后未重新枚举 ECM，
Wi-Fi SSH 仍可用。记录见 `21651/dist/ff7dcdf484ef/otg-xhci-*.log`。

低地址 DMA 对照镜像 `d76a775be9bf` 仅在 21651 的 GPIO I²C OTG
测试配置中给 xHCI 子设备加了 4 GiB DMA 窗口。重新启动后 USB SSH 恢复。
同一扩展坞和 U 盘仍只识别到根集线器；ERDP 高半部为 0，端口仍停在
Polling。连接阶段 IRQ 301 从 0 增长至 255991（约每秒 2.5 万次），
但未捕获到 xHCI 事件处理，采样时 `USBSTS` 通常为 `0x10`。
因此低地址窗口不能解决枚举失败。拔除后升压与 xHCI 自动关闭，接回
开发机已验证 USB SSH，采集退出并恢复默认自动挂起。原始记录见
`21651/dist/d76a775be9bf/otg-xhci-{dma32,journal}.log`。
下轮宜用直连 USB 2.0 外设，同时查看命令和事件环的 TRB；采集脚本
已停止逐条记录 xHCI IRQ，以免中断风暴覆盖插入时的记录。

串接电压电流表后，参考 Android 空载无电压、接负载时有电压；
21651 主线最初外测无电压，随后在同一镜像的供电采样中，用户确认
表上出现电压。启用阶段 MP2650 的 OTG ADC 持续约 5.0 V（内部测点），
未读到 OTG 故障位；MT6360 的 VBUS ADC 在进入主机后大多为 0，
不能将其读数直接等同于连接器电压。首次启动仍未枚举；15 秒保护
关断后重新进入主机，连续两次枚举了四口高速集线器 `1-1` 和下游
高速设备 `1-1.3`。该镜像未启用 `CONFIG_USB_STORAGE`，未验证 U 盘
块设备或读盘；表上电压与各次重连的精确对应时刻亦未记录。已恢复
固定受电。日志：`21651/dist/d76a775be9bf/otg-vbus-meter*.log`。

镜像 `94444f36a1c2` 保留低地址窗口，加上 USB 存储驱动和设备 ID 日志。
表端读到稳定 5 V 后掉电；本轮内核记录限时保护在 71.393 秒执行（距
首次进入主机约 15 秒），然后重新进入主机。两次都只出现根集线器，
没有 USB 块设备，说明之前的下游枚举仍不可稳定复现。拔除能关断升压；
接回开发机时能枚举 gadget，但 ECM 数据曾卡住，重新绑定 gadget
也未恢复。同镜像重启后 USB 与 Wi-Fi SSH 均恢复，已确认固定受电、
升压 GPIO 为低。日志：`21651/dist/94444f36a1c2/otg-vbus-storage*.log`。

PHY 角色对照 `840212fae8eb` 保留存储与低地址 DMA，在 21651
双角色路径按厂商做法显式设定 T-PHY 模式。实测 USB2 PHY 的 DTM1
在主机时为 `0x224`、回从机时为 `0x226`，确认 IDDIG 随角色变动。
同一扩展坞仍未枚举。主机阶段抓到端口 `Polling`、速度 0，
命令环有 `Enable Slot`，事件环首五项均为 0，IRQ 301 计数为 0。
PHY 角色改动未解决问题，已从源码撤回；诊断补丁与原始日志留在
`21651/dist/840212fae8eb/`。下一步检查 xHCI 命令与事件路径，优先把
原厂的 SSUSB/xHCI 时钟引用接入当前启动配置，并在同一轮复测。

时钟对照 `9abe4fb2ad1f` 接入两路原厂 INFRA 门及其 TOP 父时钟，
UNIVPLL 仅读回。运行时确认 MTU3 和 xHCI 持有实际的 124.8 MHz 时钟；
xHCI 模式下时钟门引用计数为 2，未再出现无效时钟索引 59。首次
插入仍只出现根集线器；15 秒关断后，后两次均枚举了扩展坞
`1a86:8095` 和 Netac U 盘 `0dd8:3b00`。已从 U 盘块设备直接
只读 64 KiB 成功，未挂载。首次插入故障仍存在，时钟依赖修正保留；
不能仅据此认定时钟缺失是首次失败的原因。日志位于
`21651/dist/9abe4fb2ad1f/otg-{clock*,vbus-clock}.log`。

同镜像再次冷插，保持 VBUS 供电并在约 4 秒时仅重新绑定 xHCI；
根集线器重新创建，仍未枚举外设。约 15 秒时保护切断 VBUS，
TCPM 重新进入供电主机后，扩展坞和 U 盘才枚举成功，再次只读
64 KiB 通过。此对照排除了「只需重新绑定 xHCI」作为修复；
整次角色重建同时改变了供电和 PHY/端口状态，尚不能区分哪一步
使重试成功。后续应对照完整的角色交接顺序。日志：
`21651/dist/9abe4fb2ad1f/otg-vbus-reprobe*.log`。

保持供电的数据角色对照复测：约 831.1 秒进入主机，835.1 秒仅将
MTU3 数据角色切到从机再切回主机，MP2650 保持使能。内核确认
xHCI 确实移除并重建，但仍只出现根集线器，无 U 盘块设备；
随后限时关断后的自动重试也未成功，命令环中止超时。
因此这次未复现读盘，且不能把早期成功归因于断电本身。结束后
恢复固定受电，升压关闭。接回开发机后 gadget 已枚举但 ECM 不通，
同镜像重启后 USB 和 Wi-Fi SSH 均恢复。记录见
`21651/dist/9abe4fb2ad1f/otg-data-role-retry.6j7lSn*.log`。

共享控制器复位镜像 `fff341740671` 在切入主机时停止 gadget、
同步其中断，复位 MTU3/xHCI 共享控制器并完成主机时钟检查，再注册
xHCI；回从机时恢复 QMU 描述符格式。首次插入约 0.4 秒枚举扩展坞、
2.6 秒出现 U 盘块设备，直接只读 64 KiB 成功，早于 15 秒保护。
保持接线期间保护触发后又自动进入主机五次，每次都重新识别了 U 盘。
恢复固定受电后接回开发机，USB SSH 可用，无需重启。该轮仍沿用
低地址 DMA 诊断窗口，尚未验证去掉该窗口后的结果。日志见
`21651/dist/fff341740671/otg-ip-reset*.log`。

本轮未验证带电物理拔除。最后一次保护在启动后 120.550 秒触发，
121.179 秒重新进入主机。采集脚本于 130.072 秒退出并切回 `sink`，
TCPM 进入 `PORT_RESET`，在 130.095 秒报告 VBUS 关闭。操作者确认
等到不再自动上电才拔线；143.973 秒记录到开发机重新供电。因此这轮
只验证了软件关断与接回开发机后的恢复。时序见 `otg-ip-reset-stop.log`。

镜像 `5835595060c8` 移除了临时 `dma-ranges` 限制，保留 `CONFIG_USB_STORAGE=y`。
用同一套电压表、扩展坞和 Netac U 盘实测：首次进入主机后约 0.53 秒
枚举扩展坞、2.60 秒出现 U 盘块设备，直接只读 64 KiB 成功，未挂载。
持续供电约 23.1 秒后，操作者确认电压表仍显示 5 V 时拔下手机端。
TCPM 在 99.885 秒检测到 CC 断开，约 21 ms 后报告 VBUS 关闭；
MP2650 升压和隔离 GPIO 均恢复低电平，30 秒测试保护未触发。
接回开发机后，双角色模式下自动恢复受电、USB 从机和 SSH；随后结束
采集，恢复固定受电。日志见 `21651/dist/5835595060c8/otg-no-dma-window*.log`。
本轮覆盖 USB 2.0 扩展坞和这只 U 盘；长时间带载、其他外设及 USB 3.x
尚未验收。

gadget 会话修正版 `e3b9d6ebf446` 用同一套设备复测通过：首次进入主机后
约 0.54 秒枚举扩展坞，2.65 秒出现 U 盘块设备，直接只读 64 KiB 成功，
未挂载。VBUS 持续约 19 秒，操作者确认电压表仍为 5 V 时拔下手机端。
TCPM 检测到 CC 断开约 22 ms 后报告 VBUS 关闭，未触发 30 秒保护。
接回开发机后 USB SSH 自动恢复；采集结束后恢复固定受电，升压关闭。
日志见 `21651/dist/e3b9d6ebf446/otg-session-regression*.log`。

在 `mt6893-direct` 下用以下命令构建 21651 的 OTG 镜像。`typec-otg`
默认使用已验证的 GPIO I²C5，保留 USB 存储驱动，关闭额外的函数跟踪。
产物的 `otg-tcpc-bus` 应为 `gpio`。构建会拒绝 USB 控制器上的临时
`dma-ranges`；普通构建的默认 USB 配置仍是 `peripheral`。

```sh
OP6893_KERNEL_SRC=../linux-mainline \
OP6893_KERNEL_OUT="$(realpath 21651/shared-build/op6893-package)" \
21651/build.sh userdata radio graphics systemd normal off typec-otg
```

插拔复测时通过 Wi-Fi SSH 以 root 运行 `otg-vbus-sample.py`。使用
`--cutoff-seconds 30` 可复现这轮验收的保护窗口；在电压表仍显示 5 V 时
拔下手机端。采集退出时记录何时主动切回 `sink`，用来区分软件关断和
物理拔除。原生 I²C5 和函数跟踪仅在对照时分别用
`OP6893_OTG_GPIO=0` 和 `OP6893_OTG_TRACE=1` 启用。

常用构建产物 `c3b6034bf6a8` 已刷入 21651，boot 写后哈希一致。
实测从 userdata 进入 systemd，开发机线保持连接时自动进入受电从机，
USB 和 Wi-Fi SSH 均恢复。采集脚本能设置 30 秒保护并切回固定受电；
结束时 `usb-vbus` 已关闭。这次复核启动与救援通道，未重复实体 OTG 插拔。

双角色实验镜像 `8e88c6d6626e` 能进 Linux 登录界面，但 USB 未枚举；
已用 fastboot 恢复 `08f0bc6a7ce3`。修正版 `0824701286fb` 将 xHCI
移到启动后加载，并还原 gadget 的内部 VBUS 检测补偿，USB SSH 与 Wi-Fi SSH
均恢复。xHCI（0x11200000、SPI 97）在从机状态下绑定超时 `-110`；拔线后先切
主机再绑定，出现 480 Mb/s、一个端口的 USB 2.0 根集线器。没有接外设，
也没有驱动 VBUS 输出。切回从机发现硬件撤除 D+ 时清掉了 gadget 的连接请求。

镜像 `28559ae18e40` 保留 gadget 的连接请求。已通过 Wi-Fi 实测从机 →
主机 → 从机；主机阶段注册同一个 USB 2.0 根集线器，解绑 xHCI 并切回从机时
日志确认自动恢复 D+。重新接线后开发机重新枚举，ECM SSH 恢复。
镜像可用 `21651/build.sh userdata radio graphics systemd normal off typec-drd-late`
构建，测试用的 Wi-Fi 恢复服务保留在本机 userdata。本轮 Type-C 仍然是固定受电，
手工切换仅用于验证数据侧。
镜像 `6ea289477cba` 使用内建 xHCI（构建选项 `typec-drd`）。驱动切到
host 后创建 xHCI 子设备，切回 device 前移除，避免从机状态下探测超时。
21651 断线时通过 Wi-Fi 连续三轮手动发起角色切换：每次都出现 480 Mb/s、
单端口根集线器，每次返回均移除 xHCI 并恢复 D+。最后重新接线后，USB 重新
枚举且 ECM SSH 成功登录。内核提交为 `fa45fb848`。当前尚未验证由 Type-C
自动选择角色、外设枚举或实际 VBUS 供电。

显示可共用 MT6893 的 MMSYS、显示互斥器及 DSI 主控，面板命令、供电和背光需按
机型核实。SCP 调试可对照寄存器定义和日志采集，OP6893 基带路径仍需单独验证
固件交接和资源所有权。

## 原 7.3.0 实测基线

| 功能 | 21651 当时的实测 | 当时未验收 |
| --- | --- | --- |
| 启动与 UFS | 运行镜像 24be483817b9，八核在线；Debian 13.7 从 userdata ext4 启动，重启后保留数据。PID 1 现由 systemd 接管（镜像 b3afe1f2e993），`multi-user.target` 在用户态 3.4 s 到达，无失败单元 | 未测试从主线直接写 boot |
| USB | ACM 救援、ECM 公钥 SSH，重启后自动恢复 | USB Host、角色切换未验收 |
| 显示 | 沿用 LK 扫描缓冲，simpledrm + Mesa Panfrost（Mali-G77 MC9）约 350 MHz。kmscube 输出 1199 帧；另 250 帧逐像素校验无差异 | 完整面板接管、DVFS、runtime suspend、长时间压力测试未验收 |
| 按键 | 两个音量键都采集到完整按下、松开 | 电源键尚未验收 |
| 触摸 | FT3518 已绑定；I²C 芯片 ID 0x5452，固件版本 0x4f。按 21651 厂商配置改下降沿后，旧板电平触发造成的中断风暴消失 | 分段读实测点击、滑动、双指和全部抬起；长按与手势未验收 |
| 电池与温度 | 两路 PMIC ADC 都读到约 4.26 V；LVTS AP/MCU 使用本机校准，五个温区约 44.5–49.6 °C | MP2650/divider/BQ27541 尚未接管 |
| Wi-Fi | 使用本机固件、校准及 MAC；开放 AP 上实测关联、IPv6 link-local ping 和公钥 SSH | 暂未验收网络放行、DHCP 或互联网访问；Wi-Fi 需手动启动 |
| Bluetooth | BT/Wi-Fi 预校准成功；BR/EDR 发现 2 台设备，显式开启 LE 后扫描记录 107 条发现事件 | BlueZ 配对、PAN 与并发稳定性未验收；早期 BlueZ 试跑有固件 ownership 超时 |
| 基带 | 120 秒诊断镜像 `011fb03f4847` 收到 HS2（HS1 后约 68 秒），CCISM 达到 RUNNING，MD/SCP 正常停止 | 复验冷启动、确认正式时限；初始 AT 帧已到，尚未验证 AT 往返、SIM、注册或数据面 |

无线镜像通过工作区的 21651/build.sh userdata radio 构建。无线预校准
在切换根文件系统前发生，通用固件放在 initramfs；本机 Wi-Fi 校准在
运行时通过 op6893-21651-wifi-calibration 从只读 nvdata 读取和验证。
通过 USB SSH 执行 op6893-21651-wifi-link-local 并传入开放网络 SSID，
会得到用于 SSH 的 IPv6 link-local 地址。开发主机连接时应附上本机无线
接口作用域，使用和 USB 相同的固定主机密钥。

图形版通过 `21651/build.sh userdata radio graphics` 构建，Panfrost
模块与内核版本配套，已存入 userdata 根文件系统。重启后可运行
`/usr/local/sbin/op6893-21651-load-gpu` 加载；simpledrm 已内建。

Linux 下以 root 执行 `/usr/local/sbin/op6893-21651-reboot-fastboot`，
可直接进入 LK fastboot。21651 原厂 DTS 指定的 TOPRGU reboot-mode 单独
使用时，本机重启仍回到 Linux；加入 MT6359 的 RTC fastboot 位后两次成功，
主机确认 `getvar is-userspace` 为 `no`。从 fastboot 普通重启可回到 Debian；
测试没有写入 misc。工具只在显式调用时加载匹配内核版本的 RTC 模块，
后续构建的模块与用户空间工具一起输出到镜像目录的 `reboot/`。
Recovery 和 fastbootd 尚未验证。

历史 BusyBox 基线的 [systemd 一次性启动选项](21651-SYSTEMD.md) 已通过 USB、
看门狗、正常重启回退和进入 LK fastboot 的试验。本文当前实测镜像使用显式的
`systemd` 构建配置。

另有独立的[自选 userdata rootfs 启动方案](21651-USERDATA-BOOT.md)，
同一测试镜像已分别通过 BusyBox 与 systemd 的标准入口。
