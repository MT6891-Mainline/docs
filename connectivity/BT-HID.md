# chopin 蓝牙 HID（鼠标/键盘）：连接失败是缺 `CONFIG_UHID`

> **状态（2026-11-06 上机）**：内核打开 `CONFIG_UHID=y` 后，BT3.0 鼠标
> （`51:23:13:1A:B4:02`）连接成功并出现在 `/dev/input` 下。
> 关联：`connectivity/BT-HANDOFF.md`（控制器/BTIF）、`connectivity/BT-AUDIO.md`
> （A2DP）。

## 症状

蓝牙鼠标已配对，`bluetoothctl connect` 失败或连上后没有输入事件；
`/dev/input/` 下只有内置的 `mtk-pmic-keys`、`mtk-kpd`、触摸屏，没有鼠标。

## 诊断

```sh
ls -l /dev/uhid
#   ls: cannot access '/dev/uhid': No such file or directory

zcat /proc/config.gz | grep -E "CONFIG_UHID|CONFIG_BT_HIDP"
#   CONFIG_BT_HIDP=m
#   # CONFIG_UHID is not set
```

`CONFIG_BT_HIDP`（BR/EDR HID 传输）是有的，但 **`CONFIG_UHID` 没开**。
BlueZ 的 HID 插件并不直接创建输入设备：它把 HID 报告描述符通过
`/dev/uhid` 交给内核，由内核的 `uhid` 驱动注册 `input` 设备。内核没有
uhid 节点时，连接过程拿不到落脚点，鼠标永远出不来。BLE（HOGP）鼠标走同一条
uhid 路径，症状相同。

## 修复

```diff
--- a/kernel/chopin_defconfig
+++ b/kernel/chopin_defconfig
@@
 CONFIG_HID_MULTITOUCH=m
+# BlueZ creates the input device for Bluetooth mice/keyboards through uhid;
+# without it the HID profile has nowhere to hand the report descriptor.
+CONFIG_UHID=y
```

重编内核（`port/tools/build-kernel.sh`，增量约半分钟）并刷新的 boot：

```sh
fastboot flash boot_a out-scp-v181-rootfs/boot-stockdt.img
```

## 验证

```sh
ls -l /dev/uhid
#   crw------- 1 root root 10, 239 /dev/uhid

bluetoothctl connect 51:23:13:1A:B4:02
cat /proc/bus/input/devices | grep -A2 -i mouse
#   N: Name="BT3.0 Mouse"
#   H: Handlers=mouse0 event3
```

## 备注

- `CONFIG_HIDRAW` 仍然没开；仅用鼠标/键盘不需要它，需要给用户态
  hidraw 访问时再说。
- 键盘与此修复无关的部分（布局、Fn 键）另见输入相关文档。
