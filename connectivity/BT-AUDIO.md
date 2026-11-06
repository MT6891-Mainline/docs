# chopin 蓝牙音频（A2DP）：配对好了却连不上

> **状态（2026-11-06 上机）**：装 `libspa-0.2-bluetooth` 后，vivo TWS 3i 可以连接，
> `bluez_output.*` sink 出现，测试音正常。
> 关联：本机的蓝牙控制器固件/BTIF 移植见 `connectivity/BT-HANDOFF.md`。

## 症状

耳机（vivo TWS 3i，`A8:6F:36:16:B7:06`）在配对数据库里是 `Paired: yes`、
`Trusted: yes`，但 `bluetoothctl connect` 之后 `Connected: no`，
`/var/log` 与 `journalctl -u bluetooth` 里反复出现：

```
src/service.c:btd_service_connect() a2dp-sink profile connect failed for A8:6F:36:16:B7:06: Protocol not available
```

## 不是固件问题

排查过的、**正常**的部分：

| 检查 | 结果 |
| --- | --- |
| 控制器固件 / BTIF | `MTK-BTIF` 初始化成功，`hci0` HCI 5.2 / MediaTek（`version 11 manufacturer 70`） |
| 扫描 | 正常，能扫到附近设备 |
| 配对 | 已配对（`/var/lib/bluetooth/04:E5:98:D4:FB:F5/A8:6F:36:16:B7:06/info` 有记录） |
| 适配器 | `UP RUNNING PSCAN` |

所以不是 preloader/LK/蓝牙固件的问题，是 **用户空间缺了 A2DP 的媒体端点**。

## 根因

BlueZ 的 `a2dp` 插件本身只负责 AVDTP/L2CAP 信令；真正的音频端点由
**音频服务器**（这里是 PipeWire + WirePlumber）通过 SPA 的 `bluez5` 模块注册。
Debian 把这个模块单独打包成 `libspa-0.2-bluetooth`，而我们的 rootfs 只装了
`pipewire` / `wireplumber` 本体，没装它：

```
libpipewire-0.3-0t64      1.4.2-1
libpipewire-0.3-modules   1.4.2-1
pipewire                  1.4.2-1
pipewire-bin              1.4.2-1
libspa-0.2-bluetooth        ← 缺失
```

没有它，BlueZ 的 a2dp profile 拿不到传输，连接以 `Protocol not available`
（`ENOPROTOOPT`）结束。

## 修复

```sh
apt-get install -y libspa-0.2-bluetooth

# 让 PipeWire/WirePlumber 重新加载 bluez5 模块
systemctl --user restart wireplumber pipewire pipewire-pulse

# 重新连接耳机
bluetoothctl connect A8:6F:36:16:B7:06
```

验证：

```sh
bluetoothctl info A8:6F:36:16:B7:06 | grep -E "Connected|Audio Sink"
#   Connected: yes
#   UUID: Audio Sink  (0000110b-...)

pactl list short sinks
#   59  bluez_output.A8_6F_36_16_B7_06.1  PipeWire  s16le 2ch 48000Hz

pactl set-default-sink bluez_output.A8_6F_36_16_B7_06.1
speaker-test -c 2 -t sine -f 440 -l 1
```

## 仍缺的两项

### 1. 免提（HFP/HSP）不可用

```
Unable to connect Hands-Free Voice gateway: socket(STREAM, RFCOMM): Protocol not supported (93)
```

内核没有 `CONFIG_BT_RFCOMM`，RFCOMM 套接字建不起来，耳机的麦克风/通话用不了。
音乐播放走 A2DP/L2CAP，不受影响。要修得在内核配置里打开 RFCOMM 并重建 boot。

### 2. AVRCP 媒体键

```
profiles/audio/avctp.c:init_uinput() AVRCP: failed to init uinput for vivo TWS 3i
```

BlueZ 用 `/dev/uinput` 把耳机的播放/暂停键注入成输入事件，普通用户没有权限。
需要一条 udev 规则或把用户加进相应组；不加的话只是媒体按键不生效。

## 交付物

`git/chopin-support` 的 `denial` 分支里 rootfs 的软件清单只列了
`pipewire`/`wireplumber`；`libspa-0.2-bluetooth` 需要补进去，否则每刷一次
新 rootfs 都会重现这个问题。
