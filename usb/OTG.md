# USB OTG / 主机模式（未完成）

> 状态：**停在这里**。供电链路已通，端口使能失败，属主线 MT6893 USB host 的空白区。

## 1. 现象

| 阶段 | 现象 |
| --- | --- |
| LK → 内核早期 | OTG 设备指示灯**亮**，`xhci-mtk` 认到设备（`new high-speed USB device`） |
| 进 Linux 系统后 | 灯灭（VBUS 被关） |
| 修了角色/供电后 | 灯**闪烁** —— 设备在反复重试枚举 |

```
usb 1-1: new high-speed USB device number 17 using xhci-mtk
usb 1-1: device not accepting address 17, error -22
usb 1-1: device descriptor read/8, error -110
```

## 2. 供电链路（已打通）

### 2.1 硬件路径

```
TCPM 检测到 OTG 适配器 → source 角色
    → role switch → mtu3 进入 host
    → ssusb_host_enable()
    → ssusb_set_vbus(1)            (mtu3_host.c:353)
    → usb-otg-vbus regulator 使能
    → VBUS 5V 输出
```

**手工验证过**：

```sh
echo host > /sys/class/usb_role/11201000.usb-role-switch/role
→ usb-otg-vbus state=enabled num_users=1
→ VBUS(ADC) = 5043 mV      ✅
```

### 2.2 两个必须修的 bug

**bug 1：TCPM 拿不到 role switch**

`usb_role_switch_get()` 的两条路径都不成立：

```c
sw = usb_role_switch_is_parent(dev_fwnode(dev));   // tcpc 的父节点是 MT6360 MFD
if (!sw)
    sw = device_connection_find_match(dev, "usb-role-switch", ...);  // 缺这个属性
```

→ `port->role_sw = NULL`，TCPM 只能改电源角色，无法请求 host 数据角色。

修法（两步都要）：

```dts
tcpc {
	usb-role-switch = <&{/usb@11201000}>;
};
```

```c
/* tcpm.c: tcpm_register_port() 只查一次，而 tcpc 比 USB 控制器先探测 */
static void tcpm_role_sw_lookup(struct tcpm_port *port)
{
	if (!port->role_sw)
		port->role_sw = usb_role_switch_get(port->dev);
}
```

**踩坑**：不要在 `tcpci_mt6360` 里做 `usb_role_switch_get()`/`put()` 来触发
deferred probe —— TCPM 自己也持有引用，多出来的 get/put 会触发

```
refcount_t: addition on 0; use-after-free
```

必须在 TCPM 内部做惰性查找，让所有权只有一个地方。

**bug 2：厂商 `usb_type_c@4e` 节点与主线 TCPC 抢 I2C 地址**

```
i2c i2c-5: Failed to register i2c client usb_type_c at 0x4e (-16)
```

→ `status = "disabled"`（主线用 `tcpci_mt6360`）。

## 3. 未解决的问题：端口使能失败

设备被认到但无法完成地址分配：

```
device not accepting address N, error -22     (EINVAL)
device descriptor read/8, error -110          (ETIMEDOUT)
```

**这是老问题**：开机那一刻（t=4.25s，还没进系统）就已经是同样的错误，与
角色/供电修复无关。

### 候选原因

| 原因 | 依据 |
| --- | --- |
| **PHY 未切到 host 模式** | `grep -r phy_set_mode drivers/usb/mtu3/` → **一处都没有**。MTK T-PHY 有 host/device 不同的模拟配置 |
| OTG 升压带不动瞬时电流 | 枚举瞬间设备拉电流，MT6360 OTG boost 可能限流 |
| 端口电气配置（眼图/驱动强度） | `error -22` 出现在地址阶段，典型是信号问题 |

### 下一步需要

1. 给 `mtu3_host_setup()` 加 `phy_set_mode(PHY_MODE_USB_HOST)`，对照厂商
   `mtu3` 的实现
2. 或查 MT6360 OTG boost 的限流配置（`usb-otg-vbus` regulator 的 current limit）
3. 需要厂商 `usb_type_c`/PHY 的初始化序列做参考

## 4. 结论

**USB host 对这台机器是加分项，不是必需功能。** 供电链路已经修好并提交，
但端口使能需要更深的 PHY 层工作，且每次验证都要刷机。

已提交的修复本身有价值（PD 快充的角色协商、`usb_type_c` 冲突消除），
建议在此收尾，把 host 模式记为已知未完成项。
