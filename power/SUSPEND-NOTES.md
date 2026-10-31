# chopin 主线 suspend 笔记（2026-10-23）

## 现状

- **suspend 不能真正进入 deep/s2idle**：`mtu3 11201000.usb` 的 gadget 还在跑，
  `wait_for_ip_sleep()` 超时返回 `-EBUSY`，整个 suspend 被 abort。
- abort 后第一次 resume 会让 DSI/panel 停在半挂起状态 → 花屏。
- 第二次 suspend（走完一次 panel 关/开）能恢复显示，所以目前用
  `systemd-sleep` hook 在第一次 resume 后自动再 suspend 一次。
- 关闭 mtu3 wake（`ssusb_wakeup_set(false)`）会让系统真正睡下去，但
  **PMIC 电源键 wake 不生效，会睡死**；不要再用。
- `mtk-pmic-keys` 已加 `device_init_wakeup()`，`power/wakeup=enabled` 能出现，
  但仍不足以从该 sleep 状态唤醒；还需要 pwrap/PMIC IRQ wake 的完整验证。

## 当前可用 workaround

`/usr/lib/systemd/system-sleep/chopin-display-reset`：

```sh
#!/bin/sh
logger -t chopin-display "sleep hook: $1 $2"
[ "$1" = "post" ] || exit 0

if [ ! -e /run/chopin-resume-retry ]; then
  touch /run/chopin-resume-retry
  systemd-run --on-active=3 systemctl suspend
else
  rm -f /run/chopin-resume-retry
fi
```

效果：第一次 suspend 可能短暂花屏，自动第二次 suspend 后恢复；不会睡死。

## 90 Hz

- 面板 90Hz DDIC 寄存器 `0x18=0x21` 实际是 **91.2Hz**，DRM 模式原来是
  90.0Hz，二者不匹配，90Hz 下即使不 suspend 也会 `DSI0 buffer underrun`。
- 已把 90Hz 从 connector 模式列表移除，只保留 60/120。
- 90Hz 花屏和 suspend 花屏是**两个独立问题**；前者是 DSI/DSC 时序，
  后者是 mtu3 abort + panel 半挂起。

## 镜像

- 当前稳定：`out-safe/boot-stockdt.img`
  - md5 `5cb59a4d80cedc05334924aef3326ad4`
  - GPU 内置、触控半屏修复、90Hz 移除、mtu3 原版 abort + retry hook
- 恢复：`out-gpu-touch-half/boot-stockdt.img`
  - md5 `11dd6e94cfcfa3a40555820155902f46`

## 后续要真正修

1. mtu3 gadget suspend：USB console 活跃时不要返回 -16；
2. PMIC 电源键 wake：pwrap/PMIC IRQ wake 路径；
3. DRM/DSI resume：第一次 resume 后强制重新初始化 panel；
4. 90Hz：对照 vendor DSI/DSC 参数（data rate、HFP、DSC timing）。
