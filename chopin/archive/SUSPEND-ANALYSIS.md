# chopin 挂起（suspend）完整分析

> ## ⛔ 禁止在这台设备上继续测试挂起
>
> 2026-10-30 的实验记录：v127 / v128 两次「第一次挂起」测试，每次都让面板显示
> 错误帧，第二次之后**屏幕边缘出现烙印**。LCD 的图像粘滞靠断电静置消退（数小时），
> 但反复触发会累积成永久性差异。
>
> **`echo mem > /sys/power/state`、`rtcwake -m mem`、`systemctl suspend`
> 一律不要再执行。** logind 抑制锁只挡得住 `systemctl suspend` 与空闲挂起，
> **挡不住 root 直接写 `/sys/power/state`**——那是我测试时用的路径，也正是造成
> 面板损伤的路径。
>
> 结论见 §0：挂起在这台机器上无法安全实现，保持禁用。


## 0. 症状

`systemctl suspend`（或 logind 空闲挂起）之后：

* 屏幕花屏（条纹），只能断电；
* 严重时 SSH 也拒绝连接、`swiotlb buffer is full`，图形栈彻底锁死；
* 内核日志三组关键错误：

```
mtu3 11201000.usb: ip sleep failed!!!
mtu3 11201000.usb: PM: dpm_run_callback(): platform_pm_suspend returns -16
PM: Some devices failed to suspend, or early wake event detected

[DISP][E]mtk_atomic_doze_preparation connector has no crtc
l16_36_02_0b_dsc_vdo,lcm 1400e000.dsi.0: Skipping disable of already disabled panel
[DDP Error][IRQ] DDP_COMPONENT_DSI0:buffer underrun
[DISP][E]mtk_drm_crtc_release_fence no session for CRTC0
[DISP][E]mtk_release_layer_fence:508 layer_info is null     (×12 每次尝试)

Wakeup pending. Abort CPU freeze
```

## 1. 链路 A：USB gadget 拒绝休眠（已验证可修）

**代码**：`drivers/usb/mtu3/mtu3_plat.c`

```c
static int wait_for_ip_sleep(struct ssusb_mtk *ssusb)
{
	if (!ssusb->is_host)
		sleep_check = ssusb_gadget_ip_sleep_check(ssusb);
	if (!sleep_check)
		return 0;
	ret = readl_poll_timeout(..., (value & SSUSB_IP_SLEEP_STS), 100, 100000);
	if (ret) { dev_err(..., "ip sleep failed!!!\n"); ret = -EBUSY; }
	return ret;
}

static int mtu3_suspend_common(...)
{
	...
	ret = wait_for_ip_sleep(ssusb);
	if (ret)
		goto sleep_err;          /* → resume_ip_and_ports(); return -EBUSY */
```

gadget（我们的 `ttyGS0` 串口控制台）处于 configured 状态时 IP 不会进 sleep，
100 ms 超时后**整个系统挂起以 -EBUSY 失败**。

**实测有效的规避**（本轮已验证）：

```sh
echo disconnect > /sys/class/udc/11201000.usb/soft_connect   # 挂起前
echo connect    > /sys/class/udc/11201000.usb/soft_connect   # 恢复后
```

断开后再次尝试，日志中**不再出现 `mtu3 ... -16`**，`Some devices failed to
suspend` 也消失。这也是 Android 的做法（挂起前关掉 USB gadget）。

生产化形态：`/usr/lib/systemd/system-sleep/` 钩子（`pre` disconnect / `post`
connect）。**注意**：串口控制台会在挂起期间消失。

## 2. 链路 B：`Wakeup pending. Abort CPU freeze`（未修）

即使 gadget 已断开、传感器 trigger 已解绑、NetworkManager 已停止，
挂起仍然在 CPU 冻结阶段被中止：

```
[330.332] PM: suspend entry (deep)
[331.619] Wakeup pending. Abort CPU freeze
[331.955] PM: suspend exit
```

`/sys/kernel/debug/wakeup_sources` 显示两个**持续持有**的 wakelock：

```
disp_crtc0_wakelock   active_since 持续增长（10 s+）
cmdq_pm_lock          active_since 持续增长
```

来源：`drivers/misc/mediatek/cmdq/mailbox/mtk-cmdq-mailbox-ext.c`

```c
cmdq->wake_lock = wakeup_source_register(dev, "cmdq_pm_lock");
static void cmdq_lock_wake_lock(struct cmdq *cmdq, bool lock) {
	if (!cmdq->wake_locked) { __pm_stay_awake(cmdq->wake_lock); cmdq->wake_locked = true; }
	...
}
```

显示路径的 CMDQ 提交在空闲时也持续占着这个 wakelock（`active_since` 一路增长），
`__pm_stay_awake()` 会累加 `combined_event_count`，于是 `pm_wakeup_pending()` 恒为真，
CPU 冻结永远被中止。

要修这条，需要让 CMDQ 在显示空闲时真正释放 wakelock（检查 ring 是否 drain），
属于 vendor CMDQ 驱动的改动。

## 3. 链路 C：每次挂起尝试都会拆掉显示（最危险）

**这是"花屏"的直接来源**，而且**即使挂起最终失败也会发生**：

```
mtk_atomic_doze_preparation connector has no crtc
Skipping disable of already disabled panel     ← 面板已经是关闭状态
Skipping unprepare of already unprepared panel
DDP_COMPONENT_DSI0: buffer underrun            ← DSI 还在跑但面板已关 → 欠载
mtk_drm_crtc_release_fence no session for CRTC0
mtk_release_layer_fence: layer_info is null ×12
```

含义：挂起路径认为面板"已经关闭"就跳过 disable/unprepare，但 **DSI 仍在送数据**，
于是 underrun；随后 fence 会话被释放，unblank 时状态已经不一致 → 花屏。

为什么面板"已经关闭"：合成器（mutter）在挂起前会先 blank 掉输出，vendor 的
`mtk_atomic_doze_*` 又按 Android HWC 的假设处理，两者叠加导致重复关闭。

v126 已把 `layer_info is null` 从 `-EFAULT` 降级为返回 `fence_fd = -1`，所以
**commit 不再失败**（这是进步：花屏不再恶化成不可恢复），但 DSI underrun 和
面板状态不一致依旧。

## 4. 本轮做过的实验（全部实测）

| # | 实验 | 结果 |
| --- | --- | --- |
| 1 | 直接 `echo mem > /sys/power/state` | `mtu3 -16`，挂起失败，显示被拆，swiotlb 耗尽 |
| 2 | 断开 gadget 后再试 | **`mtu3 -16` 消失**，但仍 `Wakeup pending` 中止；显示仍被拆（layer_null ×12） |
| 3 | 停 `chopin-sensorproxy` + 解绑 trigger + 停 NetworkManager | 仍 `Wakeup pending` → 传感器与 WiFi 不是主因 |
| 4 | `rtcwake -m mem -s 20` | `rtcwake: write error`（挂起中止），桌面存活 |
| 5 | 关 USB/kp 的 `power/wakeup` | 仍 `Wakeup pending` → 唤醒源在显示/CMDQ 侧 |

结论：**B 和 C 未解决**，因此挂起仍然不可用，且每次尝试都会拆一次显示（有花屏风险）。

## 5. 当前策略（已固化）

**禁止挂起**，用 logind 抑制锁（不是 mask target —— 那会让 gnome-session 立刻退出）：

```ini
# /etc/xdg/autostart/chopin-no-suspend.desktop
Exec=gnome-session-inhibit --inhibit suspend --reason "suspend is unsupported on chopin" sleep infinity
```

实测：

```
$ systemctl suspend
Call to Suspend failed: Access denied
$ dmesg | grep -c "suspend entry"
0
```

## 6. 要真正修好挂起，按这个顺序做

1. **链路 C 先修**（安全性前提）：让 blank/unblank 走正规的 disable/enable 路径，
   不要在面板已关时留下运行中的 DSI。落点：
   `mediatek_v2/mtk_drm_drv.c` 的 `mtk_atomic_doze_*`、`mtk_dsi.c` 的
   `mtk_dsi_poweroff`/panel unprepare 顺序，以及 `[DISP] ... buffer underrun`
   的 IRQ 处理。
2. **链路 B**：让 `cmdq_lock_wake_lock(cmdq, false)` 在 ring 空闲时可靠执行
   （`mtk-cmdq-mailbox-ext.c`），或挂起前由 DRM 主动 release 全部 wakelock。
3. **链路 A**：加 `system-sleep` 钩子断开/重连 gadget（已验证有效）；
   更彻底的方案是改 `mtu3`：IP 睡眠超时不再让整个系统挂起失败。
4. 全部就绪后再解除抑制锁，用 `rtcwake -m mem -s 20` 逐次验证（有 RTC 自动唤醒，
   不会睡死）。

**在此之前，任何一次挂起尝试都可能让屏幕花屏**，所以抑制锁必须保持开启。

---

## 7. 补充：只有第一次挂起会花屏（v127 修复）

### 7.1 现象

用户观察：**同一次开机里，第一次挂起会花屏，之后就不再花**。

这条信息直接指向 `mtk_drm_sys_suspend()` 的一处**不平衡**。

### 7.2 根因

```c
static int mtk_drm_sys_suspend(struct device *dev)
{
	/* 只要有一个 CRTC 的 wakelock 还持有，才算"需要挂起" */
	for (i = 0; i < 3; i++)
		wake_state |= mtk_crtc->wk_lock->active;

	if (wake_state == false)
		goto OUT;                       /* ① 跳过 drm_atomic_helper_suspend() */

	private->suspend_state = drm_atomic_helper_suspend(drm);
OUT:
#ifdef MTK_DRM_FENCE_SUPPORT
	mtk_drm_suspend_release_fence(dev);     /* ② 但 fence 照样释放 */
#endif
}

static int mtk_drm_sys_resume(struct device *dev)
{
	if (!private->suspend_state) {          /* ③ 于是 resume 什么都不做 */
		DDPINFO("suspend_state is null\n");
		return 0;
	}
	drm_atomic_helper_resume(drm, private->suspend_state);
}
```

第一次挂起时合成器已经把屏幕 blank 掉，所以 `wake_state == false`：

1. ① 跳过状态保存 → `suspend_state == NULL`
2. ② 仍然执行 `mtk_drm_suspend_release_fence()`，它对
   `MTK_TIMELINE_OUTPUT_TIMELINE_ID`（**12**）个 timeline 逐个调
   `mtk_release_layer_fence()` —— 正好对上日志里的 `layer_info is null` ×12
3. ③ resume 因为 `suspend_state` 是 NULL 直接返回，**显示没有被恢复**
4. 合成器下一次提交时 fence 会话已经没了，无法正确重新点亮面板
   → `DSI0: buffer underrun`、`no session for CRTC0` → **花屏**

**为什么之后不再花**：`suspend_state` 一直是 NULL、fence 会话已经被清空，
合成器也重新初始化了自己的状态，不再有新的不匹配，所以后续挂起"看起来正常"。

### 7.3 修复（v127）

```diff
-	if (wake_state == false)
-		goto OUT;
+	if (wake_state == false) {
+		/* 没有真正挂起，就不要动 fence，否则 resume 无状态可恢复 */
+		DDPMSG("%s: nothing to suspend, keeping fences\n", __func__);
+		return 0;
+	}
 	...
 	private->suspend_state = drm_atomic_helper_suspend(drm);
-OUT:
 #ifdef MTK_DRM_FENCE_SUPPORT
 	mtk_drm_suspend_release_fence(dev);
 #endif
```

外加防御：`mtk_drm_suspend_release_fence()` 在主 session 未注册时直接返回。

### 7.4 验证方法（v127）

```sh
fastboot flash boot_a out-scp-v127-rootfs/boot-stockdt.img
```

开机后**第一次**主动挂起（有 RTC 自动唤醒，不会睡死）：

```sh
echo disconnect > /sys/class/udc/11201000.usb/soft_connect   # 串口 gadget 会拒绝休眠
rtcwake -m mem -s 20                                          # 20 秒后自动醒
echo connect > /sys/class/udc/11201000.usb/soft_connect
```

判据：

```sh
dmesg | grep -c "layer_info is null"      # 期望 0（原来是 12）
dmesg | grep -c "no session for CRTC"     # 期望 0
dmesg | grep -c "buffer underrun"         # 期望 0
```

即使挂起仍然因为 `Wakeup pending` 中止（见 §2），**这一次不应再花屏**。

### 7.5 仍然未解决

这次只修了"挂起被中止时把显示状态搞坏"的问题。§2 的 `Wakeup pending`
（CMDQ wakelock 持续持有）依然存在，所以挂起**仍然不能真正完成**；
抑制锁继续保留。

---

## 8. 实测 v127：fence 修好了，但画面仍花 —— 因为面板从未被重新打开

### 8.1 v127 的实测数据（本次开机的第一次挂起）

```
layer_info is null = 0        ← 从 12 降到 0，fence 破坏已修复 ✅
no session for CRTC = 1
buffer underrun = 2
wakeup_pending = 1            ← 挂起仍被中止
挂起累计时长 = 0.0 s           ← 没真正挂起
桌面：gnome-shell 存活，DSI status=connected / enabled=yes
```

但**用户确认画面仍是花屏**。

### 8.2 真正原因：挂起尝试后，面板再也没被 prepare/enable

统计挂起尝试之后的显示动作：

```
Skipping disable of already disabled panel      (disable=1，被跳过)
Skipping unprepare of already unprepared panel (unprepare=1，被跳过)
prepare = 0
enable  = 0
mi_disp_notifier_call_chain, notify = 2        (blank 通知)
（没有 notify = 1，即 unblank 从未发生）
```

链路：

1. 合成器（mutter）在挂起前先 blank → vendor blank notifier **直接关掉面板**
   （`drm_panel_disable`/`unprepare`，所以后面 `mtk_drm_sys_suspend` 里的操作全被跳过）
2. `mtk_drm_sys_suspend()` 因为 `wake_state == false` **没有保存状态**
   （`private->suspend_state == NULL`）
3. 挂起因 `Wakeup pending` 中止，`mtk_drm_sys_resume()` **直接 return**（没状态可恢复）
4. 于是**面板保持关闭**，DSI 却还在跑 → `buffer underrun`
5. 合成器以为屏幕还开着，不再发 enable → 面板停在半初始化状态 → **花屏**

### 8.3 修复（v128）

`mtk_drm_sys_suspend()` **无条件快照显示状态**，这样 resume 一定会执行
`drm_atomic_helper_resume()`，重新 enable 面板并再发一次 unblank notifier：

```diff
-	if (wake_state == false)
-		return 0;                  /* 没有状态 → resume 什么都不做 */
-
 	private->suspend_state = drm_atomic_helper_suspend(drm);
 	...
 #ifdef MTK_DRM_FENCE_SUPPORT
-	mtk_drm_suspend_release_fence(dev);
+	/* 显示本来就空闲时，fence 仍归合成器所有，不能释放 */
+	if (wake_state)
+		mtk_drm_suspend_release_fence(dev);
 #endif
```

`wake_state == false` 只表示"合成器已经把输出 blank 了"，**不代表什么都没发生**——
面板确实被关掉了，而唯一能把它打开的就是带状态的 resume。

### 8.4 为什么只有第一次花屏（完整解释）

| 第几次 | 面板状态 | 结果 |
| --- | --- | --- |
| 第 1 次 | 由正常显示 → 被 blank 关闭 | 没有状态快照 → resume 不恢复 → **花屏** |
| 第 2 次起 | 已经是关闭/半初始化状态 | blank 操作变成 no-op，状态不再变化 → 看起来"正常" |

### 8.5 镜像

```
out-scp-v128-rootfs/boot-stockdt.img
md5 af13aed195639d346eab564944b573ff   18.0 MB
```

验证方法同 §7.4（`rtcwake -m mem -s 20`，判据：
`dmesg | grep -c "Skipping enable"` 应该 ≥1，说明面板被重新打开过）。

---

## 9. 为什么花屏会「持续」而不是自愈（v129）

### 9.1 问题

以前的经验：花屏 → 断电静置 → 恢复。
但挂起导致的花屏**会在系统运行时一直持续**，直到重启才消失。
持续以错误内容驱动 LCD = 图像粘滞不断累积 = 边缘烙印（已实测出现）。

### 9.2 内核层原因：DSI underrun 中断没有任何恢复动作

`mediatek_v2/mtk_dsi.c` 的 IRQ handler：

```c
if (status & BUFFER_UNDERRUN_INT_FLAG) {
	DDPAEE("[IRQ] %s:buffer underrun\n", ...);      /* 打印 */
	...
	if (mtk_crtc)
		atomic_set(&mtk_crtc->force_high_step, 1);  /* 抬一档带宽步进 */
}
```

**只有日志和带宽步进，没有复位、没有重新初始化、没有重启视频流。**
所以 DSI 一旦进入 underrun（面板被关、DSI 还在送）就一直是错的，
控制器继续以 120 Hz 输出垃圾帧。

### 9.3 驱动里其实有恢复例程，只是没被调用

`mediatek_v2/mtk_disp_recovery.c: mtk_drm_esd_recover()`：

```
CONNECTOR_PANEL_DISABLE
mtk_drm_crtc_disable(crtc, true)
mtk_drm_crtc_enable(crtc)
CONNECTOR_PANEL_ENABLE
mtk_crtc_hw_block_ready()
```

——正是**一次软重启显示**。它只被 ESD 检测线程在"面板读失败"时调用，
而 DSI underrun 不会让 ESD 检测失败，所以永远轮不到它。

### 9.4 修复（v129）：resume 之后强制重初始化

```c
static int mtk_drm_sys_resume(struct device *dev)
{
	drm_atomic_helper_resume(drm, private->suspend_state);
	drm_kms_helper_poll_enable(drm);
	private->suspend_state = NULL;

	/* 失败的挂起会把面板留在"关了但 DSI 还在跑"的状态，
	 * 这里重做一次完整的 disable/enable 循环（= 重启时的动作） */
	mtk_disp_recover_all(drm);
}
```

新增 `mtk_disp_recover_all(struct drm_device *drm)`：
遍历所有 active CRTC，在 `MTK_DRM_OPT_ESD_CHECK_RECOVERY` 打开时调用
`mtk_drm_esd_recover()`。

### 9.5 效果预期

| 场景 | v128 之前 | v129 |
| --- | --- | --- |
| 挂起尝试失败后 | 面板花屏，持续到重启 | resume 时强制重初始化 → 应恢复正常 |
| 面板损伤 | 持续错误驱动，累积烙印 | 仅在失败瞬间有短暂错误帧 |

**这不能阻止挂起失败**（`Wakeup pending` 仍在），但能让**失败不再伤害面板**。

### 9.6 镜像

```
out-scp-v129-rootfs/boot-stockdt.img
md5 9eb8fb3410eda08792fef093afcb3d7c   18.0 MB
```

验证（等面板静置恢复、并且**愿意再冒一次险**时）：

```sh
echo disconnect > /sys/class/udc/11201000.usb/soft_connect
rtcwake -m mem -s 20
dmesg | grep -c "recovering crtc"        # 期望 ≥1，说明 recover 跑了
```

如果 `recovering crtc` 出现且屏幕恢复正常 → 挂起失败不再损伤面板。
如果仍然花屏 → 恢复例程对这条路径无效，**就此停止**，保持禁用挂起。

---

## 20. 最终结论（2026-10-31）：挂起放弃，熄屏已实现

### 20.1 USB 阻塞已修复（v144）

`mtu3_dev_on_off()` 只写硬件 pullup、不更新 `mtu->softconnect`，
导致软断开后 `ssusb_gadget_ip_sleep_check()` 误判"还能睡"，
`wait_for_ip_sleep()` 超时后让**整个系统挂起以 -EBUSY 失败**。

v144：IP 不睡就不再失败，PHY/时钟保持开启并记 `ip_awake_during_suspend`，
resume 时跳过重复使能。实测 `platform_pm_suspend returns -16` 消失。

### 20.2 但撞上 CMDQ wakelock（无法安全修复）

```
[102.51] PM: suspend entry (deep)
[104.14] Wakeup pending. Abort CPU freeze
[106.14] PM: suspend exit
```

`pm_wakeup_clear()` 之后、`suspend_enter()` 检查之前，
**显示挂起路径自己用 CMDQ 提交**激活了 `cmdq_pm_lock`
→ `pm_wakeup_pending()` 为真 → 中止。

要修必须在 DRM 挂起后强制 relax CMDQ wakelock，
而 CMDQ 可能仍有事务在跑 → 截断 → 卡死或 resume 花屏。

### 20.3 实测代价（5 次尝试）

| 尝试 | 结果 |
| --- | --- |
| v125–v128 时代 | USB 拒绝休眠 → **花屏** |
| v144（USB 已修） | CMDQ wakelock → **系统卡死**（ARP 在、SSH 死） |

两次都需要强制断电重启，并伴随面板图像粘滞风险。

### 20.4 最终决定

| 项 | 决策 |
| --- | --- |
| 挂起 | ❌ **永久放弃** |
| 熄屏（DPMS off） | ✅ **启用**，60 秒空闲 |
| 保护 | logind 抑制锁（6 个）、`chopin-no-suspend.desktop` |

**收益对比**：

* 熄屏：省 **0.5–1.5 W**（面板 + 显示管线 + DSI）✅ 已实现
* 挂起：再省 **0.3–0.5 W** ❌ 风险远超收益

### 20.5 ⛔ 禁止命令

抑制锁**挡不住**下面两条，它们是造成卡死的路径：

```sh
echo mem > /sys/power/state     # ⛔ 禁止
rtcwake -m mem -s N             # ⛔ 禁止
```

日常的 `systemctl suspend` / GNOME 关机菜单的"挂起"会被抑制锁拒绝，是安全的。

---

## 21. 2026-10-31 再攻挂起：找到"花屏"的真凶

### 21.1 mtu3 那一关确实通了

v155 修好 `softconnect` 状态跟踪后，挂起完整走过了 USB 阶段：

```
PM: suspend entry (deep) → … → PM: suspend exit
（没有 "ip sleep failed"，没有 returns -16）
```

**mtu 不再是阻塞点。**

### 21.2 wakelock 也不再阻塞（v159）

`/sys/kernel/debug/wakeup_sources` 显示 `disp_crtc0_wakelock` 和 `cmdq_pm_lock`
长期 active，而 `wakeup_source_activate()` **无条件递增全局计数器**：

```c
/* drivers/base/power/wakeup.c */
cec = atomic_inc_return(&combined_event_count);
```

挂起期间显示自己被提交/关闭 → wakelock 重新激活 → 计数器变化 →
`Wakeup pending. Abort CPU freeze`。

v159 在挂起期间跳过 wakelock 操作（`mtk_drm_set_suspend_in_progress()`）。

### 21.3 花屏真凶：**resume 里多做了一次开关**

关键观察：**挂起回来时画面先好了一下，然后才花**。

日志顺序解释了原因：

```
56.24  [DDP Error]mtk_dsi_exit_ulps fail
56.24  Skipping prepare of already prepared panel
56.24  Skipping enable of already enabled panel
56.24  [DISP][E][IRQ] DDP_COMPONENT_OVL0: frame underflow!
```

`drm_atomic_helper_resume()` **已经把画面恢复了**（用户看到的"一瞬间好"）。
紧接着 v129 加的那句无条件 `mtk_disp_recover_all()` 又做了一遍 关→开：

* 多一次 `mtk_dsi_exit_ulps()` —— 挂起后这次会超时失败
* 面板状态机认为"已 enable"→ 恢复调用变空操作
* 结果：DSI 链路停在 ULPS，帧下溢 → **花屏**

**是恢复逻辑自己把好画面搞坏了。**

### 21.4 修复

| 版本 | 改动 |
| --- | --- |
| v160 | `mtk_dsi_exit_ulps()` 失败时重置 PHY 并重试（最多 3 次） |
| **v161** | resume **不再**强制重初始化；交给 underrun 重试路径按需处理 |

```c
/* mtk_drm_sys_resume() */
drm_atomic_helper_resume(drm, private->suspend_state);
/* 不再调用 mtk_disp_recover_all()：它自己就会把画面弄坏 */
```

真正被拆掉面板的情况，由 v143 的 underrun 重试兜底（等 CRTC active 后
重新初始化）。

### 21.5 仍未解决

```
[54.22] [DISP][E] mtk_crtc_cmdq_timeout_cb cmdq timeout, crtc id:0
[54.23] [CMDQ]<0>(0) suspend:0x1
```

挂起时显示还在提交 CMDQ 命令，而 CMDQ 已经开始挂起 → 命令等不到完成 →
超时。这是设备挂起顺序问题，**尚未修复**，可能是后续卡顿/异常的另一来源。

---

## 22. 2026-10-31 v162 实测：**花屏修好了**，但挂起仍被 WiFi 打断

### 22.1 显示侧的证据

挂起尝试后（中止），计数器：

```
ulps_fail = 0      ← 不再出现 "mtk_dsi_exit_ulps fail"
skip_en   = 0      ← 不再出现 "Skipping enable of already enabled panel"
underrun  = 2      ← 熄屏瞬间的正常 underrun
```

**之前导致花屏的两个签名都消失了。** 修复合集：

| 版本 | 改动 |
| --- | --- |
| v160 | `mtk_dsi_exit_ulps()` 超时后重置 PHY 重试 |
| v161 | resume 不再无条件做完整 关→开 |
| **v162** | 跟踪 ULPS 状态；PHY 断电后不再尝试退出 ULPS |

其中 v162 是关键：`mtk_dsi_disable()` 进 ULPS 后立刻 `phy_power_off()`，
ULPS 状态随 PHY 断电已经不存在，再去"退出"必然失败。

### 22.2 但挂起仍中止：WiFi 扫描

```
[38.16] Skipping disable of already disabled panel
[39.20] Wakeup pending. Abort CPU freeze
[39.20] CHOPIN-SCN: kalScanDone bss=1 status=0x0 aborted=0 list=45   ← 同一毫秒
```

WiFi 的周期性扫描在挂起窗口内完成，`wakeup_source_activate()` 递增
`combined_event_count` → 中止。

**要稳定挂起必须处理这个**：挂起前停止 WiFi 扫描，或让 WiFi 驱动在系统挂起
窗口内不产生唤醒事件。

### 22.3 中止后显示停在关闭状态

中止后 `dpms=Off`，`/sys/class/drm/card1-DSI-1/dpms` 又被 DRM master 占用
（`Permission denied`），只能靠重启 gdm（完整 modeset）恢复。

这是一个独立的小问题：中止路径没有把显示状态复位。

### 22.4 当前结论

* **花屏问题已解决**（三次针对性修复后计数器全清）
* 挂起本身仍不可用：WiFi 扫描会打断它
* 中止后显示需要手动恢复（重启 gdm）

**建议**：把"熄屏"作为日常省电手段（已稳定），挂起不再追求。

---

## 23. 最终结论（2026-10-31 晚）：挂起放弃，改用熄屏

### 23.1 这一轮修好的东西（都有价值，与挂起无关）

| # | 问题 | 修复 | 版本 |
| --- | --- | --- | --- |
| 1 | **PMIC 中断接到了 GIC** | 改接 pinctrl EINT（GPIO 222 电平高） | v166 |
| 2 | 电源键 **release 中断**被武装成唤醒源 | 只武装按下中断 | v170 |
| 3 | `mtk_pmic_keys_resume()` 解除未武装的 release | 同步修正 | v175 |
| 4 | USB gadget 的 `softconnect` 状态跟踪 | 如实反映硬件 | v155 |
| 5 | 显示 wakelock 在挂起中激活 | 挂起期间跳过 | v159 |
| 6 | WiFi 唤醒事件上报 | `pm_suspend_target_state` 抑制 | v167 |
| 7 | conninfra + 蓝牙唤醒锁 | 同上 | v168 |
| 8 | 蓝牙 IRQ 武装 | 默认不武装（`bt_wakeup_enable`） | v169 |
| 9 | WiFi 扫描请求撞进挂起窗口 | 挂起期间拒绝扫描 | v171 |
| 10 | 9 个永久持有的唤醒锁否决挂起 | 只检查事件计数 | v173 |
| 11 | timesync / 充电器的 hard 事件 | 挂起期间抑制 | v174 |

**其中 #1 是最有价值的发现**：PMIC 中断接错导致的不止电源键——PMIC 的
RTC 闹钟、电池、耳机检测中断**全都是死的**，只是没人注意到。

### 23.2 仍然失败的原因

最后一轮（v175/v176）的诊断栈：

```
pm_system_wakeup
  pm_system_irq_wakeup
    irq_pm_handle_wakeup
      irq_can_handle_pm
        handle_fasteoi_irq
          gic_handle_irq
```

**是某个硬件 IRQ 在挂起窗口里触发**，`pm_system_irq_wakeup()` 把它记进
`wakeup_irq[]` 并置 `pm_abort_suspend` → 中止。

v176 会打印 IRQ 号和驱动名（`CHOPIN-ABORT: wakeup irq N (name)`），
但**没有继续验证**——见 §23.4。

### 23.3 挂起的真实代价（实测）

| 次数 | 结果 |
| --- | --- |
| 10+ 轮尝试 | 3 次系统卡死、3 次花屏、每次都要重启 |
| 每次卡死 | 屏幕处于未知状态，有图像粘滞风险 |

即使修好，收益也只是相比熄屏多省 **0.3–0.5 W**。

### 23.4 决定：用熄屏（已经成熟）

| 项 | 方案 |
| --- | --- |
| **电源键** | 切换背光（`chopin-powerkey.service` → `lm36273/brightness`） |
| 空闲熄屏 | 60 秒（GNOME idle-delay，DPMS off，已验证不花屏） |
| GNOME 电源键动作 | `power-button-action='nothing'`（不让 gsd 去挂起） |
| logind | `HandlePowerKey=ignore` |
| 挂起 | ❌ 不启用；`mem_sleep` 保持 deep 但不会触发 |

**用背光切换而不是 DRM 熄屏**的理由：不碰 DSI/面板状态机，零花屏风险，
唤醒是瞬时的。真正的 DPMS 熄屏仍由 60 秒空闲触发。

### 23.5 还留下的诊断代码

`pm_wakeup_pending()` / `pm_system_wakeup()` / `pm_system_irq_wakeup()` /
`wakeup_source_activate()` 里的 `CHOPIN-WAKEUP` / `CHOPIN-ABORT` 打印**保留**：
它们只在挂起尝试时触发，正常情况下无声，将来想继续做可以直接用。

### 23.6 如果将来要重做

1. 刷 v176，按电源键，读 `dmesg | grep CHOPIN-ABORT` → 拿到 IRQ 号
2. 对照 `/proc/interrupts` 找到驱动
3. 在驱动的 `suspend` 回调里 `disable_irq_wake()`，`resume` 里恢复
4. 重复直到没有 `CHOPIN-ABORT`

预计还剩 1–3 个 IRQ（蓝牙 BTIF、PMIC EINT、触摸屏最可能）。
但**不建议继续**：代价是反复卡死和花屏风险，收益只有 0.3–0.5 W。

---

## 24. 后续事故：不要把 logind 的崩溃当成挂起改坏了

2026-10-31 晚出现"系统不稳定、进不去桌面"，日志：

```
systemd-logind[364]: Failed to abandon session scope, ignoring:
                     Transport endpoint is not connected
Session 9 logged out
...
gdm3: GdmLocalDisplayFactory: Failed to issue method call:
      GDBus.Error:org.freedesktop.DBus.Error.TimedOut:
      Failed to activate service 'org.freedesktop.login1'
gdm-wayland-session: Unable to register display with display manager
gdm-x-session: (EE) Cannot run in framebuffer mode   ← 回退到 X 后必然失败
```

### 原因

**在有活动用户会话时执行 `systemctl restart systemd-logind`**。logind 重启后
无法接管已有会话的 scope（`Transport endpoint is not connected`），它的 D-Bus
接口从此超时；gdm 拿不到 seat 就创建不了 display，于是：

```
gdm → 没有 seat → 回退到 X 会话 → X 没有驱动 → 失败 → 循环
```

而 X 的报错（`Cannot run in framebuffer mode`）很有误导性，它只是 gdm 回退后的
二次失败，**真正的根因是 logind**。

### 修复

```sh
systemctl restart systemd-logind    # 这次没有活动会话，安全
systemctl restart gdm
# 之后 gnome-shell 立刻回来（shell=2，179 MB）
```

**规则：改 logind 配置（`HandlePowerKey` 等）后不要 `restart` logind，
它会在下次启动时自动生效；必须重启时要先退出所有会话。**

### 附属修复

`systemd-localed.service` 一直是 failed，因为系统 locale 没设置。写入
`/etc/locale.conf`（`LANG=zh_CN.UTF-8`）后正常，`systemctl --failed` 为 0。
