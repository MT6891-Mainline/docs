# 充电与电池：移植状态

2026-11-08，设备运行 Linux 7.3.3（v198）。

## 当前工作正常的部分

```
bq27z561-0       status=Charging   capacity=99%   voltage=4.317V
                 current=256mA     health=Good   temp=32.7°C
                 technology=Li-ion
mt6360-chg       status=Charging   online=1      charge_type=Fast
tcpm-source      5V / 3A           online=1
```

| 功能 | 驱动 | 状态 |
| --- | --- | --- |
| 电量计 | `bq27xxx-battery`（主线） | ✅ 电量、电压、电流、温度、健康全对 |
| 充电器 | `mt6360-chg`（主线） | ✅ 充电中 |
| 快充 | `mt6360-chg` + `tcpm` | ✅ `charge_type=Fast`、5V/3A |
| Type-C PD 供电 | `mt6360-tcpc`（主线） | ⚠️ 端口注册失败，但供电协商正常 |
| 稳压器 | `mt6360-regulator` / `mt6359-regulator` | ✅ 8 路输出 |

绑定的驱动全部是主线实现：

```
mt6360-chg        mt6360-regulator    mt6360-tcpc      mt6360-adc
mt6359-regulator  mt6359-auxadc       mt6359-sound
bq27xxx-battery   chopin-adc-battery
```

## 三个未完成项

### 1. `subpmic_pmic@1a` / `subpmic_ldo@64` 地址冲突

```
i2c i2c-5: Failed to register i2c client subpmic_pmic at 0x1a (-16)
i2c i2c-5: Failed to register i2c client subpmic_ldo at 0x64 (-16)
```

`-16` = `-EBUSY`。DTS 里厂商的 `mt6360_pmu_dts` / `mt6360_pmic_dts` /
`mt6360_ldo_dts` 三个容器占着这两个地址，与主线 `subpmic@34` 冲突。

**尝试过清理，失败**：删除这三个容器后，`mt6893.dtsi` 里有 3 处消费者引用
它们的 regulator label：

```
/msdc@11240000:               Reference to non-existent node "mt_pmic_vmch_ldo_reg"
/spi1@11010000/fingerprint@0: Reference to non-existent node "mt_pmic_vmc_ldo_reg"
/usb@11201000:                Reference to non-existent node "mt6360_pmu"
```

而主线的 `subpmic@34` 里只有 `charger` / `adc` / `tcpc`，**没有 regulators
子节点**。所以要清理必须先补 `mt6360-regulator` 的 8 路输出定义，是完整改造
而非简单删除。改动已回退。

### 2. `mt6360-led` 找不到 of_node

```
mt6360-led: Failed to locate of_node [id: -2]
```

DTS 里没有 LED 子节点。补一个即可，当前没有 LED 使用需求。

### 3. `bq27z561` 的 data memory 不支持

```
bq27xxx-battery 0-0055: data memory update not supported for chip
```

主线 `bq27xxx` 不认这颗芯片的 data memory 格式。影响设计容量、SOH、循环
次数，**不影响基本电量**（`capacity=99%` 是电压/电流推算的）。

## 结论

**充电和电池的核心功能完整可用，但不是完全移植。** 缺口集中在：

- 两个 i2c 地址的厂商容器占位（日志噪音，无功能影响）
- LED 节点缺失
- 电量计的深度参数读取

用户可感知的功能（电量显示、充电、快充、温度、健康）**全部正常**。

## 如果以后要彻底清理

```text
1. 在 subpmic@34 下补 regulators 子节点
   （buck1/buck2/ldo1..ldo7，名字参照 drivers/regulator/mt6360-regulator.c）
2. 把 mt6893.dtsi 里 3 处引用改成新的 label
3. 删除厂商三个容器 + type_c_port0
4. 补 mt6360-led 节点
5. 刷机验证充电仍正常（重点看 charge_type 和 capacity）
```

风险点：`mt6360-regulator` 在主线上是否完整覆盖 MT6893 的 8 路输出需要验证；
regulator 树一改，`msdc`（UFS/eMMC 供电）和 `spi1`（指纹供电）都可能受影响，
出错会直接影响启动。

## 验证命令

```sh
# 电池与充电状态
for d in /sys/class/power_supply/*/; do
    echo "--- $(basename $d)"
    for f in status capacity voltage_now current_now charge_type health temp; do
        [ -f "$d$f" ] && echo "  $f: $(cat $d$f)"
    done
done

# 驱动绑定
ls /sys/bus/platform/drivers/ | grep -iE "mt6360|bq27|mt6359"

# 相关日志
dmesg | grep -iE "mt6360|bq27|charger|tcpm"
```
