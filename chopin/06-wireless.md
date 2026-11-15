# 06 · 无线：Wi-Fi 与蓝牙

Wi-Fi 和蓝牙**共用 conninfra**（同一颗 connac2 芯片），所以先把 conninfra 的上电/
时钟/固件下载跑通，蓝牙就只剩 BTIF 传输和 BT 侧固件。

## Wi-Fi

| 项 | 状态 |
| --- | --- |
| 驱动（`wlan_drv_gen4m` fullmac + conninfra） | ✅ |
| 连接 / 扫描 | ✅ |
| NVRAM（WIFI 校准数据） | ✅ 启动时从 nvdata 拷出 |

**关键坑**：wlan 驱动会在启动后几秒请求 NVRAM，而 conninfra 的预校准只等 ~30 秒；
晚于这个窗口 ⇒ **连接芯片永久卡死**（驱动加载成功但扫不到 AP）。所以 initramfs
必须**最早**把固件写进去（[`01-boot.md`](01-boot.md)）。

不是从 4.14 硬搬：MT6895-Mainline 已经把同一套栈现代化到 7.2，且
`conninfra/platform/mt6893/` 与 chopin 原厂**逐字节相同**，只补了三处平台差异。
细节：[`archive/WIFI-PORT.md`](archive/WIFI-PORT.md)。

## 蓝牙

| 项 | 状态 | 说明 |
| --- | --- | --- |
| 基础（HCI/BTIF/固件） | ✅ | BTIF 时钟 ID、`BT_PLATFORM=6893`、HCI 容错 |
| BD_ADDR | ⚠️ | `btmtk_set_bdaddr_vendor` 失败（327），地址从 nvdata 拷 |
| A2DP（音频） | ⚠️ | 配对成功但连不上 `a2dp-sink` |
| HID（鼠标/键盘） | ❌ | **缺 `CONFIG_UHID`** |

细节：[`archive/BT-HANDOFF.md`](archive/BT-HANDOFF.md)、
[`archive/BT-AUDIO.md`](archive/BT-AUDIO.md)、[`archive/BT-HID.md`](archive/BT-HID.md)。
