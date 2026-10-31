# chopin 蓝牙移植（MT66xx / BTIF / connac2）

> 记录把 MT6893（chopin）的蓝牙从厂商 4.14 内核搬到 mainline 7.2.8 的过程。
> **状态（2026-10-24 上机）**：BTIF 传输、ROM patch/固件下载、HCI 初始化全部
> 通过；`hci0` 为 HCI 5.2 / MediaTek（`version 11 manufacturer 70`），
> 已连接蓝牙耳机，LE 扫描正常。

## 结论先放前面

- 蓝牙和 WiFi 共用 **conninfra**（同一颗 connac2 芯片）。WiFi 已经把
  conninfra 的上电/时钟/固件下载跑通，所以蓝牙只需要补 **BTIF 传输**和
  **BT 侧固件**。
- 厂商 BT 栈在 `connectivity/bt/mt66xx` 里本来就有，只是之前没编译。
- 关键适配：BTIF 时钟 ID、`BT_PLATFORM=6893`、HCI init 对
  `READ_SYNC_TRAIN_PARAMS` 的容错、`Read Local Version` 字段无条件刷新。

## 一、硬件与传输

```
AP --BTIF(DMA)-- connac2 (BT/FM/WiFi) -- 天线
```

- `btif@1100c000`：寄存器 + DMA（`0x11000d80`/`0x11000e00`），
  IRQ 0x9e（246，btif）、0x98（247，tx dma）、0x99（248，rx dma）；
- 时钟：`CLK_INFRA_AO_BTIF`、`CLK_INFRA_AO_APDMA`（厂商 DT 里的
  `0x1d`/`0x70` 是厂商编号，mainline 不认）；
- BTIF 驱动把 vFIFO 分配在 `0xd0100000`/`0xd0102000`（CMA 区），
  然后由 BT 栈通过 BTIF 收发 WMT/HCI 包。

## 二、内核改动

| 文件 | 改动 |
| --- | --- |
| `drivers/misc/mediatek/btif/` | 从 MT6895 基线整体导入（7.4k 行） |
| `drivers/misc/mediatek/Kconfig` / `Makefile` | 接上 `CONFIG_MTK_BTIF` |
| `connectivity/bt/mt66xx/btif/Makefile` | `BT_PLATFORM := 6893` |
| `mt6893-xiaomi-chopin-infra-pinctrl.dtsi` | `btif` 时钟改成 mainline ID |
| `net/bluetooth/hci_sync.c` | `READ_SYNC_TRAIN_PARAMS` 返回 Unknown Command 时忽略 |
| `net/bluetooth/hci_event.c` | `Read Local Version` 的 CC 无条件刷新版本字段 |
| `connectivity/bt/mt66xx/btmtk_main.c` | `BT_ADDR_FIRMWARE` 改成 `mediatek/mt6893/BT_Addr` |

配置：`CONFIG_MTK_BTIF=y`、`CONFIG_MTK_COMBO_BT=y`、`CONFIG_BT=y`（BREDR+LE）。

## 三、固件与 BD 地址

厂商 `/vendor/firmware/`（super 分区偏移 `13246464*512`、大小
`4428400*512`，initramfs 的 `fwload` 会挂载）：

| 文件 | 用途 |
| --- | --- |
| `soc3_0_ram_mcu_1a_1_hdr.bin` | connsys MCU patch |
| `soc3_0_ram_bt_1a_1_hdr.bin` | BT patch（主） |
| `soc3_0_ram_bt_1_1_hdr.bin` | BT patch（备） |
| `BT_FW.cfg` | BT 配置（驱动按 `bt.cfg` 找，init 会复制一份） |
| `BT_Addr`（nvdata） | 6 字节 BD 地址，来自 `/nvdata/APCFG/APRDEB/BT_Addr` |

initramfs 的 `fwload`/`mirror_firmware` 已把 BT 文件纳入自动拷贝；
`fwload_nvdata` 会把 nvdata 里的 `BT_Addr`（440 字节 NV item，前 6 字节是
**显示顺序**的地址）反转成 6 字节小端后写入
`/lib/firmware/mediatek/mt6893/BT_Addr`（`install_bt_addr()`）。
直接拷整个 NV item 会得到组播地址（如 `F5:FB:D4:98:E5:04`）；
正确的原厂地址是 `04:E5:98:D4:FB:F5`。

## 四、验证

```sh
dmesg | grep -iE "btmtk|hci0" | head -40   # ROM patch/固件下载/MCU pc
hciconfig hci0 up
btmgmt info | head -3                      # version 11 manufacturer 70
bluetoothctl show | head -3
bluetoothctl --timeout 20 scan on
```

成功标志：`__download_patch_to_emi: load binary [soc3_0_ram_bt_1a_1_hdr.bin]`、
`MCU pc = 0x00100ce2`、`hci0: Opcode 0x0c77 failed: -56` 只出现一次且不中断，
`btmgmt info` 显示 `version 11 manufacturer 70`，`Powered: yes`。

## 五、已知问题 / 备注

- 早期启动（initramfs 阶段，~2.5s）驱动会因找不到 `BT_Addr` 重试若干次，
  switch_root 后能读到，属正常现象。
- `bluetoothctl show` 的版本字段来自 bluetoothd 启动时的缓存：刚换内核或
  刚修好字段后需要 `systemctl restart bluetooth` 刷新一次；`btmgmt info`
  始终是实时值。不是内核问题。

## 六、固件整理（自动）

rootfs 模式启动、镜像完固件后，init 还会做两件收尾（`cleanup_firmware()`）：

1. 若 `mediatek/mt6893/BT_Addr` 不存在而 `mediatek/mt6895/BT_Addr` 存在，
   自动迁移（BT 驱动路径从 xaga 的 mt6895 改成了 mt6893）；
2. 删除早期手工猜名字留下的 `BT_RAM_CODE_MT7922/MT7961_1a_1_hdr.bin`
   （connac 实际加载 `soc3_0_ram_bt_*`），并清掉空的 `mt6895` 目录。

日志里会打印 `init: migrate ...`、`init: removed stale ...`。

## 七、镜像

| 镜像 | md5 |
| --- | --- |
| `out-debian/boot-stockdt.img`（Debian，含 BT） | `21c49885c638c0fa33b8c02d29b07a62` |
| `out-bt-alpine/boot-stockdt.img`（Alpine 调试） | `f90f8d8afb003c260c62a06c59f2cb68` |

内核：`7.2.8-gb8202332cbcc`。
