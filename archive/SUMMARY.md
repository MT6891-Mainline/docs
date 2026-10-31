# chopin（Redmi Note 10 Pro / MT6893）主线移植 — 成果总结

> 内核基线：`linux-7.2.8`；参考实现：`MT6895-Mainline/linux`（已跑通）、
> `MT6893-Mainline/linux`（同 SoC）。
> **本轮（2026-10-01）推翻了上一轮的核心结论，见第一节。**

---

## 第三十轮（2026-10-18）：Wi-Fi 驱动移植 —— 构建通过，上机待做

完整记录见 `port/WIFI-PORT.md`，这里只放结论。

**做了什么**：把 MT6895-Mainline 已经 7.2 化的整套 downstream connectivity
（`conninfra` + `connadp` + `connfem` + `wlan_adaptor` + `wlan_drv_gen4m`，
789 个文件）搬进 `git/chopin-linux-7.2.8/drivers/misc/mediatek/connectivity/`，
按 MT6893 改成 `CONNAC2X2_SOC3_0` / `WLAN_CHIP_ID=6893` / `CONFIG_MTK_COMBO_CHIP_CONSYS_6893`，
接进 Kconfig/Makefile/defconfig，补 DT，`Image` + `dtb` 编译通过
（`build/net-build7.log`、`build/net-build9.log` 均 `EXIT=0`）。

**关键事实**：

| 事实 | 证据 |
| --- | --- |
| 基线里的 `conninfra/platform/mt6893/` 与 chopin 原厂 4.14 内核**逐字节相同** | `diff -rq` 无输出 |
| 但 `soc3_0.c`/`hal_dmashdl_soc3_0.h` 是更新 release，ring 映射与 DMASHDL quota 和原厂不同 | 已换回原厂版本，只改 `kalGetFwFlavor()` 签名 |
| 7.2 上不能走 `pm_runtime`（没有 CONN genpd），改用 MT6893 自带的 SPM 直写序列 | `MTK_CONNINFRA_CLOCK_BUFFER_API_AVAILABLE=0` |
| `mt6893_pmic.c` 会经 `g_regmap` 写 PMIC；该 regmap 来自 consys 的 `pmic` phandle | DT 已补 `pmic = <&{/pwrap@10026000/mt6359-pmic}>` |
| 固件在 `/vendor`（super 内的逻辑分区），SELinux 挡住 adb shell | initramfs 新增 `fwload`，loop+offset 只读挂载后拷到 `/lib/firmware` |
| `make dtbs` 不重建内核里真正用的那棵树 | 改 dts 后必须 `make Image`（或 touch `arch/arm64/kernel/chopin-board.dts`） |

**还没做的**：上机。需要 `fastboot flash boot_a port/out-wifi/boot-stockdt.img`
后按 `port/WIFI-PORT.md` 第六节取日志。

---

## ★★★★★★★★★★★★★★★★★★★★★★★★★★★ 第二十九轮（2026-10-03）：DRM 首次上机 —— **驱动侧全绿，屏幕仍未点亮**

本轮第一次把真 DRM 显示栈刷进设备。结论分两半：**软件栈整条 bind 成功**
（这是过去二十八轮里从未有过的），但**没有出图**，且缺一项关键证据来判断
卡在"面板不接受非压缩流"还是"根本没在扫描"。

### ① 已实测的（设备输出，`chopin# drm`）

| 证据 | 值 | 含义 |
| --- | --- | --- |
| `/sys/class/drm/card0-DSI-1/status` | `connected` | panel 驱动 probe 成功、`mipi_dsi_attach` 完成、DSI bridge/connector 建起来了 |
| `/sys/class/drm/card0-DSI-1/enabled` | `enabled` | CRTC 真的 enable 过（不是只注册） |
| `/sys/class/drm/card0-DSI-1/dpms` | `On` | 没被 dpms off |
| `/sys/class/drm/card0-DSI-1/modes` | `1080x2400` **一行** | 刷的是**单模式**构建（两模式构建尚未上机）；模式名来自我们的 panel 驱动，不是 LK 的留给值 |
| `card0/device/graphics/fb0` | 存在 | DRM fbdev（DRM_CLIENT_DEFAULT_FBDEV）注册了 fb0 |
| `card0/device/device/uevent` | `DRIVER=mediatek-drm` | `mediatek-drm` 平台驱动绑上了 |
| `card0/device/power/runtime_status` | `active` | DRM 设备 runtime-PM 处于活动 |
| 屏幕 | 仍黑 | —— |

**这一条链能全绿，说明下面这些全对**：`mtk-mmsys` 的 `clk-mt6893-mm`
实例化 → 显示时钟 provider 存在（否则 DSI/OVL 都拿不到 clock）→ mt6893 DDP
主路径 7 个组件全部 probe → `mt6893-dsi` 注册 host → DSI 与 mipi_tx PHY 绑定
→ 面板 53 条初始化命令发完（`prepare` 无报错，否则 connector 不会是
`connected`）→ `devm_drm_of_get_bridge()` 找到了 panel 并包成 bridge →
`drm_bridge_connector_init()` 建出 connector。

### ② 本轮**未能**确定的（缺证据）

判"面板不接受非压缩"还是"根本没扫描"，需要**时钟门 + 中断计数 + 完整 dmesg**：

- `mm_dsi0_mm_clk` / `mm_dsi0_intf_clk` / `mipi_tx0_pll` 是否 enabled；
- disp / dsi / mutex 三类中断计数是否在涨（**在涨 = DSI 真的在发**，恒 0 = 只配置没跑）；
- `/sys/kernel/debug/dri/0/state` 里 CRTC 的 active/mode/clock；
- 用户贴出的输出在 `=== /proc/fb ===` 处截断（且该处看起来为空，需复查
  `/proc/fb` 是否真的有 `0 mediatek-drmdrmfb`）。

这三项本来就在镜像里的 `drm` 命令里（13:52 之后的构建），所以**不需要重刷就能拿到**。

### ③ 第三次上机（`drm` 的 DRM state）—— **DRM 侧已完全就绪，问题在硬件下游**

`/sys/kernel/debug/dri/0/state`（节选）：

```
crtc[61]: crtc-0   enable=1 active=1
  mode: "1080x2400": 36 139000 1080 1564 1574 1582 2400 2454 2464 2474 0x48 0x0
plane[37]: plane-0 crtc=crtc-0 fb=62  allocated by = [fbcon]
  format=XR24 size=1080x2400 pitch[0]=4320
  obj[0]: dma_addr=0x00000000f8b00000 size=10371072
encoder-0/bridges: mtk_dsi_bridge_funcs + panel_bridge_bridge_funcs
  OF: /dsi@1400e000/panel@0:xiaomi,k10a-36-02-0a
/proc/fb: 0 mediatek-drmdrmfb
```

⇒ **首选 834 Mbps 模式被选中并在跑、fbcon 的 framebuffer 挂在 plane-0 上（物理
0xf8b00000）、DSI 与面板桥都接上、fbcon 也绑上了 DRM 的 fb**；但 `fbpaint` 仍无彩条。
所以不是"没人画"，是**硬件没把画面送到面板**。

两个附带发现：`/sys/class/drm/card0-DSI-1/dpms` **写不了**（EPERM：连接器 sysfs 只允许
DRM master 写，initramfs shell 不是 master），A/B 要改用 `unbind`/`bind`；
`dmesg panel` 里没看到 0x0a 回读那行（可能被 `drm.debug=0x06` 刷掉），要用 `dmesg readback`。

### ④ 为"看不见硬件寄存器"补的工具

arm64 没有可用的 `/dev/mem`，所以这轮给 `mediatek-drm` 加了
`/sys/kernel/debug/mediatek-drm/disp-regs`（bind 时建立、unbind 时移除）：
ioremap 并打印 **MMSYS 路由/时钟门寄存器**（0x650/0x658/0xeb4、RDMA0 mux 链
0xf34–0xf40、0xf48–0xf54、0xf68–0xf74、0xf80/0xff8、CG_CON0、SW0_RST_B）、
**mutex**（EN/MUTEX0/RST/SOF/MOD）、以及路径上每个组件的头 0x40 字节 + OVL 层地址、
RDMA `MEM_START_ADDR`、DSI `SIZE_CON` 与水平时序字数。以后"路由 mask 写错"这类
只在硬件上才暴露的问题，可以直接看到。

### ⑤ 下一步（两条并行）

1. **补证据**：重跑 `chopin# drm`（完整输出）+ `irq` + `dmesg | grep -iE 'drm|dsi|panel|rdma|mutex'`。
2. **换首选 DSI 速率再试一次**：新构建（`ringfix62` 更新版，sha 见 `DELIVERY.md`）
   把首选模式的链路速率**钉在厂商原样的 834 Mbps/lane**
   （`clock = 139000 kHz` → 24bpp/4lane = 834 Mbps，刷新 35.5 Hz），
   704.5 Mbps 的 30 Hz 模式作为第二个模式并列。这样"压缩/不压缩"成为与厂商
   配置唯一的差异；若面板 DDI 的 PLL 只认标称速率附近，旧构建的 704 也锁不上。

3. 若证据表明"DSI 在发、面板不亮"，下一轮就是 **DSC**：backport 上游 v9 的
   `mtk_disp_dsc.c` + DSI 侧 DSC（`PS_SEL=5`、`SIZE_CON=(vactive<<16)|(ps_wc/3)`、
   bpp=8 算 data_rate），面板侧 PPS 参数已从厂商驱动完整解出。

---

## ★★★★★★★★★★★★★★★★★★★★★★★★★★★ 第二十八轮（2026-10-02）：★★★ 充电 / 电池 / 屏幕 / 拔插 / USB host 全部落地 ★★★

这一轮把 chopin 从"能启动"推进到"**能当设备用**"：真充电、看得见电池、屏幕可读、拔插自愈、**能当 USB 主机**。
全部以设备实测为准，每条都验证通过后才提交。

### ① 真充电 + 电池状态可见（ringfix47 / 50 / 51，已提交 `ec95dff`、`537ad7c`、`76b902d`）

MT6360 按主线绑定接进内嵌 board 树（**在 i2c5 / 0x11F00000 上**，PMU 0x34 ✓、PMIC 0x1A ✓、LDO 0x64 ✓、TCPC 0x4E ✓；
PMU IRQB = **GPIO 24** ✓、TCPC PD IRQB = **GPIO 25** ✓），MFD 下带 `charger`/`adc`/`tcpc` 子节点 ✓，
配置全部 `=y` ✓（initramfs 不加载模块 ✓）。实测：

```
charger [mt6360-chg.2.auto]: status = Charging   online = 1   input_limit = 2000000
MT6360 PMU ADC: VBAT raw=3175 x1250 = 3.968 V     ← 从 3.83 V 涨上来 ✓
                IBAT raw=705  x2500 = +1.762 A    ← ★ 真在充电（开机自动 2 A 生效）
                VSYS 4.005 V   IBUS 1.825 A
```

**踩过的坑**：`CONFIG_CHARGER_MT6360=m` / `TYPEC_MT6360=m` 会被依赖链静默降级 ⇒ 全部强制 `=y` ✓；
0x34 在 i2c6 上被 `speaker_amp` 占用 ⇒ `-EBUSY` ✓（MT6360 在 **i2c5** ✓，以 base 树为准 ✓）；
`PLATFORM_DEVID_AUTO` 的设备名带 `.N.auto` ✓（`mt6360-chg.2.auto` ✓），按名字猜会误判成"没绑上" ✓。

### ② 热重启后屏幕可读（ringfix55–56，已提交 `62119d0`）

**机制**：热重启后 LK 把活动缓冲留在**另一块**（CAND1 0x77a20000 ✓），而 fbcon 只往 CAND0（0x78e0c000 ✓）写
⇒ 屏幕上**只有条纹没有文字** ✓，看起来像变砖 ✗（而 USB 串口当时也可能不在 ✓ ⇒ 两条通道同时归零 ✗）。

**修法**：驱动本来就有整个保留区的 **NonCacheable 映射** ✓（`ioremap_wc(FB_REGION_BASE,…)` ✓，两块候选都在里面 ✓），
于是**每 2 秒把 CAND0 镜像到 CAND1** ✓（源与目的都走该映射 ✓，写完只需 `wmb()` ✓）。
实测：热重启后**能进入屏幕控制台、串口可连、不再卡第一屏** ✓✓。

**失败记录（重要）**：第一次实现用 `__va(FB_CAND0)` 读源 ✗ —— **mblock-19 是 `nomap`** ✓，
根本不在线性映射里 ✓，而故障发生在 **kworker** 中 ✓ ⇒ workqueue 机制被搞坏 ⇒ 看门狗复位 ✓，
**只能重刷** ✗（屏幕日志正好停在 0.76 s = workqueue 线程起来的时刻 ✓）。

### ③ 热拔插后串口自愈（ringfix57–59，已提交 `f0ab97e`）

**PC 侧 dmesg 定性**：每次插回**都**重新枚举（`0525:a4a7 Gadget Serial` ✓ + `cdc_acm ... ttyACM0` ✓）✓
⇒ 枚举一直是好的 ✓；缺的是**里面读的人** ✗ —— `shell_loop()` 在拔线时的读错误（`EIO`/`EOF`）上 `return` 退出了 ✗，
之后没人读 `/dev/ttyGS0` ✓ ⇒ minicom 连得上却没提示符 ✓（**内核全程正常** ✓，日志时间戳继续走到 73.9 s ✓）。

**修法**：读失败不再退出 ✓，改为"等 `/dev/ttyGS0` 回来 → 重新 `open` + `dup2` 接管 → 继续" ✓。
另外看护改成**有证据才动手** ✓：插回后读 `/sys/class/udc/11201000.usb/state` ✓，
`configured` ⇒ 什么都不做 ✓（`g_serial` 的 tty 绑定时注册一次、插拔不重建 ✓ ⇒ 重绑反而破坏 ✗）。

### ④ USB host 模式（ringfix60，已提交 `ec88d94`）

charger 下加 `usb-otg-vbus-regulator` ✓（4.425–5.825 V）✓；mtu3 改 `dr_mode = "otg"` ✓ + `usb-role-switch` ✓
+ **`role-switch-default-mode = "peripheral"`** ✓ + `vbus-supply = <&otg_vbus>` ✓。
最后那条是**唯一**防止"开机直接进 host ⇒ 串口在用户态之前就消失"的东西 ✓
（驱动经 `usb_get_role_switch_default_mode()` 读它 ✓）。initramfs 启动时再把角色钉一次 `device` ✓，
`usbhost <秒数>` 开**限时 host 窗口** ✓、到点自动切回 ✓。

**实测（屏幕日志）**：xhci 起来 ✓、枚举到 **USB Hub** ✓、**无线键鼠作为 HID 输入设备被识别** ✓✓；
窗口结束时 `pullup D+` 切回 device ✓，shell 自愈 ✓。

**一个容易误判的现象**：host 窗口里手机是主机 ✓，对面接的是 **Hub 的上行口** ✓（Hub 与其设备正是这样枚举出来的 ✓）；
窗口结束切回 device 后，手机对面仍是 Hub 的上行口 ✗ ⇒ Hub 不会反向枚举设备 ✗
⇒ **PC 侧读不到 ttyACM0 是拓扑决定的，不是 bug** ✓ —— **把手机直插 PC 的 USB 口**即可恢复串口 ✓。

**ringfix61 修掉"切回 device 不可靠"** ✓（已上机验证：切 host 后能切回 device 并重新连上串口 ✓）：
切回后**轮询等主机真的来配置我们（≤5 秒）** ✓，等不到才重绑 mtu3 ✓（重跑 `phy_init` + 重启 gadget ✓，
顺带清掉 PHY 里 host 遗留状态、关掉可能残留的 OTG 升压 ✓）。
**不能用单次读 UDC 状态**：那可能是上一个 host 留下的**陈旧 `configured`** ✗（ringfix58 正是这样漏判的 ✓）。
另把 `usb-otg-vbus` 状态打到屏幕 ✓ 以便确认 VBUS 关闭 ✓。

**安全提示**：手机当主机会**输出 5 V** ✗，对面若是 PC 的口（也供 5 V ✗）就是两边对灌 ✓ ⇒ PC 口可能进保护 ✓；
host 测试建议用 **OTG 转接头 + U 盘/键鼠** ✓，测完**直插 PC** 恢复串口 ✓。

### 本轮的三条通用教训

1. **动唯一调试通道之前，必须先有能替代它的通道** ✓（ringfix53 与 r55 两次都栽在这里 ✗）。
2. **"代码里读得到该属性" ≠ "设备行为符合预期"** ✓ —— 没上机验证的机制不能给"切断观测"的设计背书 ✓。
3. **`nomap` 保留区不能用 `__va`** ✓（只能用 `ioremap` ✓）；而"屏幕镜像"这类改动要**从 late initcall 开始** ✓、
周期取**秒级** ✓，别在 early initcall 就每 250 ms 搬 10 MB ✗。

**当前可用镜像**：`chopin-ringfix60-usbhost-stockdt.img` ▶ `2e3057f1986da873c8e0bc913307acadada118a19359462da0b7aa6d92c5f98c`
（回退良品：ringfix59 ▶ `3279b638a026d067d0363ddfb2596996e59f646012e824b3aeb29033bc4eaf97`）


---

## ★★★★★★★★★★★★★★★★★★★★★★★★★★★ 第二十七轮（2026-10-01）：★★★ UFS 跑通了 ★★★

ringfix19 之后，**存储子系统在主线上完全工作**：

```
ufshcd-mtk 11270000.ufshci: caps: 0x108
ufshcd-mtk 11270000.ufshci: scsi host0: ufshcd
scsi 0:0:0:49488: Well-known LUN   SAMSUNG  KLUDG4UHDC-B0E1  0200 PQ: 0 ANSI: 6
sd 0:0:0:0: [sda] 1024 4096-byte logical blocks: (4.19 MB/4.00 MiB)
sd 0:0:0:1: [sdb] 1024 4096-byte logical blocks: (4.19 MB/4.00 MiB)
sd 0:0:0:2: [sdc] 31240192 4096-byte logical blocks: (128 GB/119 GiB)
sd 0:0:0:0/1/2: Attached SCSI disk
30:  284  0 0 0 0 0 0 0   GICv3  137 Level  ufshcd          ← 中断也通了
```

`[sdc]` = 31240192 × 4096 = **128 GB**，就是用户数据 LUN。同时
`arm64_serror_panic` = 0、userspace 正常到达。

**触发它的全部改动只有四条 DT 属性/引用**（都写在 `chopin-board.dts` 的覆盖里）：

| 改动 | 作用 |
| --- | --- |
| `compatible = "mediatek,mt6893-ufshci", "mediatek,mt8183-ufshci"` | 命中主线驱动 |
| `clocks`/`clock-names = "ufs","ufs_sel"` | 换掉厂商时钟编号 |
| `freq-table-hz = <0 0>, <0 0>` | 表长与时钟数对齐 |
| `phys = <&{/ufs_mphy@11fa0000}>` + 该节点 `#phy-cells = <0>` | 让 PHY 成为可用 provider |
| `vcc-supply = <&ufs_fixed_vcc>`（always-on regulator-fixed） | 绕开没注册出来的 `ldo_vemc` |
| **`mediatek,ufs-disable-mcq`** | **触发 `UFSHCD_QUIRK_BROKEN_LSDBS_CAP`**，这是最后一把钥匙 |

配合前面已经完成的 **reserved-memory 迁移（29 个固定区）**——正是它把最初的 SError 消灭掉。

### 完整报错链（八层，逐层剥开）

| 轮次 | 报错 | 修的是 |
| --- | --- | --- |
| 12 | `arm64_serror_panic` | reserved-memory 缺 14 区（根因） |
| 13 | `ufs-clk clk get failed` | 保留区迁移 → SError 消失 |
| 14 | `freq-table-hz len mismatch` / `phys node not found` | 时钟改主线 specifier |
| 15 | `vcc get failed, err=-517` | 时钟表长 + phys |
| 16 | `probe failed -517`（等 PHY） | vcc 改固定稳压器 |
| 17 | `could not get #phy-cells` | PHY probe 加逐级报错（定位手段） |
| 18 | `caps: 0x0` → `legacy doorbell mode not supported` | `#phy-cells = <0>` |
| **19** | **无** | **`mediatek,ufs-disable-mcq`** |

---

## ★★★★★★★★★★★★★★★★★★★★★★★★★★ 第二十六轮（2026-10-01）：★ 最后一层是**一个 DT 属性** ★

第二十五轮那句 "legacy doorbell mode not supported" 追下去，根因是**一个 DT 属性没写**：

```c
/* ufs-mediatek.c:713 */
	if (of_property_read_bool(np, "mediatek,ufs-disable-mcq"))
		host->caps |= UFS_MTK_CAP_DISABLE_MCQ;
...
/* ufs-mediatek.c:1286 */
	if (host->caps & UFS_MTK_CAP_DISABLE_MCQ)
		hba->quirks |= UFSHCD_QUIRK_BROKEN_LSDBS_CAP;
```

而 `UFSHCD_QUIRK_BROKEN_LSDBS_CAP` 正是 `ufshcd_hba_capabilities()`（`ufshcd.c:2529`）里
决定**是否相信控制器 LSDB_SUPPORT 位（bit29）**的那个开关：

```c
	if (!(hba->quirks & UFSHCD_QUIRK_BROKEN_LSDBS_CAP))
		hba->lsdb_sup = !FIELD_GET(MASK_LSDB_SUPPORT, hba->capabilities);
	else
		hba->lsdb_sup = true;          /* ← 这块 legacy IP 需要走这里 */
```

这块 legacy IP（`0x20160706`，`is legacy : 1`）把 bit29 报成 1（"legacy single doorbell
removed"），但**其实只支持 legacy doorbell** → `lsdb_sup` 为 false → `ufshcd_add_scsi_host`
直接 `-EINVAL`。

**我们自己的 `mt6893.dtsi` 里本来就有 `mediatek,ufs-disable-mcq;`，而 Android 基树的
`ufshci@11270000` 没有** —— 这就是全部差别。顺带解释了 `caps: 0x0`（`host->caps` 的那些
`UFS_MTK_CAP_*` 位同样靠这族属性设置；我们 dtsi 里 `caps` 是 `0x108`）。

ringfix19 补上两条属性：

```dts
mediatek,ufs-disable-mcq;   /* → UFSHCD_QUIRK_BROKEN_LSDBS_CAP，关键 */
mediatek,ufs-broken-vcc;    /* 让驱动别管已经上好电的 VCC */
```

反编译验证属性都在 ✓。

### UFS 报错链（第八层，也是最后一层）

| 轮次 | 报错 | 修的是 |
| --- | --- | --- |
| 12 | `arm64_serror_panic` | reserved-memory 缺 14 区 |
| 13 | `ufs-clk clk get failed` | 保留区迁移 → SError 消失 |
| 14 | `freq-table-hz len mismatch` / `phys node not found` | 时钟改主线 specifier |
| 15 | `vcc get failed, err=-517` | 时钟表长 + phys |
| 16 | `probe failed -517`（等 PHY） | vcc 改固定稳压器 |
| 17 | `could not get #phy-cells` | PHY probe 加逐级报错（定位手段） |
| 18 | `caps: 0x0` → `legacy doorbell mode not supported` | `#phy-cells = <0>` |
| **19** | — | **`mediatek,ufs-disable-mcq`（触发 LSDBS quirk）** |

**一个反复出现的模式**：基树（厂商 live FDT）用的是厂商 compatible、厂商时钟编号、
厂商省略掉的属性；每补一个，UFS 就往里走一层。

---

## ★★★★★★★★★★★★★★★★★★★★★★★★★ 第二十五轮（2026-10-01）：`#phy-cells` 修好，暴露最后一层

ringfix18 之后 PHY 那条路彻底通了，报错换成一句非常具体的：

```
ufshcd-mtk 11270000.ufshci: caps: 0x0
ufshcd-mtk 11270000.ufshci: legacy IP version - 0x20160706, is legacy : 1
ufshcd-mtk 11270000.ufshci: device reset done
ufshcd-mtk 11270000.ufshci: ufshcd_add_scsi_host: failed to initialize
                            (legacy doorbell mode not supported)
ufshcd-mtk 11270000.ufshci: error -EINVAL: Initialization failed with error -22
```

源码位置（`drivers/ufs/core/ufshcd.c:11018`）：

```c
	if (!is_mcq_supported(hba) && !hba->lsdb_sup) {
		dev_err(hba->dev,
			"%s: failed to initialize (legacy doorbell mode not supported)\n",
			__func__);
		return -EINVAL;
	}
```

而 `hba->lsdb_sup` 在 `ufshcd_hba_capabilities()`（`:2529`）里算出来：

```c
	if (!(hba->quirks & UFSHCD_QUIRK_BROKEN_LSDBS_CAP))
		hba->lsdb_sup = !FIELD_GET(MASK_LSDB_SUPPORT, hba->capabilities);  /* bit29 of 0x20000000 */
	else
		hba->lsdb_sup = true;
```

**`caps: 0x0` 是这里的核心异常**：`hba->capabilities`（`REG_CONTROLLER_CAPABILITIES`）读出来是 0，
而 ringfix15/16 时它还是 **`0x108`**。也就是说 **UFS 控制器的寄存器现在读不出东西了** ——
很可能 `ufshcd_hba_capabilities()` 提前返回，于是 `lsdb_sup` 停在 `false`（结构体零初始化），
最后就报成"legacy doorbell mode not supported"。

### 下一轮要查的（UFS 的最后一段）

1. **为什么 `caps` 从 0x108 变成 0x0**：对比 ringfix15/16 与 17/18 的差别 —— 唯一变化是
   PHY 现在真的被 enable/上电了。怀疑 PHY 的 `power_on`/`phy_init` 与控制器寄存器窗口
   有顺序或时钟域冲突（例如 PHY 复位把 UFS 时钟门掉，或 `ufs_sel` 被切走）。
2. 确认 `ufshcd_hba_capabilities()` 是否提前 return（在它前后加一行打印即可）。
3. 备选：给 mt6893 加上 `UFSHCD_QUIRK_BROKEN_LSDBS_CAP`（强制 `lsdb_sup = true`），
   作为绕过手段 —— 但**先查明 caps 为何为 0**，否则只是掩盖问题。

---

## ★★★★★★★★★★★★★★★★★★★★★★★★ 第二十四轮（2026-10-01）：PHY 通了，UFS 就差一个 `#phy-cells`

ringfix17 的逐级报错立刻给出答案 —— **PHY 侧完全正常**：

```
MTK-UFSPHY: mmio ok
MTK-UFSPHY: clocks ok (unipro, mp)
MTK-UFSPHY: phy_create ok
MTK-UFSPHY: provider registered, probe done      ← probe 完整成功
```

而且 `probe failed -517`（deferred probe）**从 30 次降到 0 次** ✓ —— UFS 不再 defer 了。
同时新增的报错链把最后两个问题点得清清楚楚：

```
OF: /ufshci@11270000: could not get #phy-cells for /ufs_mphy@11fa0000   ← 真凶
ufshcd-mtk: error -ENXIO: IRQ index 1 not found                          ← 次要
ufshcd-mtk: caps: 0x0 → Initialization failed with error -22
```

**基树的 `ufs_mphy@11fa0000` 没有 `#phy-cells`**（我们自己的 dtsi 第 1096 行是
`#phy-cells = <0>`）。没有它，`of_phy_provider_register()` 注册的 provider 无法被
`devm_of_phy_get_by_index()` 使用 → 主机以 -EINVAL 失败。

ringfix18 补上：

```dts
&{/ufs_mphy@11fa0000} { ...; #phy-cells = <0>; };
```

反编译验证 `#phy-cells = <0>`、`clock-names = "unipro","mp"` ✓。

### UFS 报错链（第七层了）

| 轮次 | 报错 | 修的是 |
| --- | --- | --- |
| 12 | `arm64_serror_panic` | reserved-memory 缺 14 区（根因） |
| 13 | `ufs-clk clk get failed` | 保留区迁移 → SError 消失 |
| 14 | `freq-table-hz len mismatch` / `phys node not found` | 时钟改主线 specifier |
| 15 | `vcc get failed, err=-517` | 时钟表长 + phys |
| 16 | `probe failed -517`（等 PHY） | vcc 改固定稳压器 |
| 17 | `could not get #phy-cells` | 给 PHY probe 加逐级报错（定位手段） |
| **18** | — | **`#phy-cells = <0>`** |

---

## ★★★★★★★★★★★★★★★★★★★★★★★ 第二十三轮（2026-10-01）：给 UFS PHY 加逐级报错

第二十二轮定位到 UFS 在等它的 PHY provider，而 `ufs_mtk_phy_probe()` 里**每一步失败都是静默的**
（全是 `return PTR_ERR(...)`，没有一句日志）。先排除两个"看起来可疑但其实没问题"的点：

| 检查 | 结果 |
| --- | --- |
| PHY 的 compatible 是否匹配 | ✓ `ufs_mtk_phy_of_match` 只认 `mediatek,mt8183-ufsphy`，我们的回退项里有它 |
| UFS 怎么取 PHY | `ufs-mediatek.c:309 host->mphy = devm_of_phy_get_by_index(dev, np, 0)` —— **按 index 取，不需要 phy-names** |
| PHY 要的时钟是否存在于我们的表里 | ✓ `clk-mt6893-infra_ao.c:132` `CLK_INFRA_AO_UNIPRO_SYSCLK` → `"infra_ao_unipro_sysclk"`；`:133` `CLK_INFRA_AO_UFS_MP_SAP_BCLK` → `"infra_ao_ufs_bclk"` |

于是给 `phy-mtk-ufs.c` 的 probe 每级加 `dev_info`/`dev_err`（ringfix17）：

```
MTK-UFSPHY: mmio ok
MTK-UFSPHY: clocks ok (unipro, mp)
MTK-UFSPHY: phy_create ok
MTK-UFSPHY: provider registered, probe done
```

—— 下一次启动的日志会直接指出卡在哪一步（ioremap / 时钟 / phy_create / provider 注册），
不用再靠猜。注意 `devm_platform_ioremap_resource()` 内部会 `request_mem_region`，
所以"某块 region 已被别的驱动占走"也是候选（那会返回 -EBUSY）。

---

## ★★★★★★★★★★★★★★★★★★★★★★ 第二十二轮（2026-10-01）：vcc 解决，UFS 改为等 PHY

ringfix16 之后，**供电那层彻底过了**：

| 报错 | ringfix15 | ringfix16 |
| --- | --- | --- |
| `ufshcd_get_vreg: vcc get failed, err=-517` | 出现 | **消失** ✓ |
| `probe failed -517` | 有 | 仍有，但**不再是 vreg** |

现在的原文是：

```
ufshcd-mtk 11270000.ufshci: find ti,syscon-reset fail
ufshcd-mtk 11270000.ufshci: phys node found
ufshcd-mtk 11270000.ufshci: probe failed -517
platform 11270000.ufshci: deferred probe pending: ufshcd-mtk
ufs_mtk_phy 11fa0000.ufs_mphy: Unbalanced pm_runtime_enable!
```

即 **UFS 在等它的 UFS PHY**（`phys` provider 没注册出来）。PHY 驱动本身已绑定
（`ufs_mtk_phy 11fa0000.ufs_mphy` 有输出），但 `Unbalanced pm_runtime_enable!`
说明 **probe 走了一段又失败、然后被重试**。

PHY 驱动侧确认（`drivers/phy/mediatek/phy-mtk-ufs.c:186`）：

```c
static const struct of_device_id ufs_mtk_phy_of_match[] = {
	{.compatible = "mediatek,mt8183-ufsphy"},
	{},
};
```

我们的覆盖写的是 `compatible = "mediatek,mt6893-ufsphy", "mediatek,mt8183-ufsphy";`
—— 回退项在里面 ✓，匹配没问题。probe 顺序是：

```
devm_platform_ioremap_resource → ufs_mtk_phy_clk_init → devm_phy_create
  → devm_of_phy_provider_register      ← 失败点应该在这之后/附近
```

### 下一轮要查的

1. `devm_of_phy_provider_register()` 为什么没成功（`dev->of_node`、provider 重名、
   或 probe 在其之前 return）；
2. `Unbalanced pm_runtime_enable!` 的真正来源（`phy_create()` 里的 `pm_runtime_enable`
   在第二次尝试时重入）→ 借此反推第一次 probe 失败的确切位置；
3. 顺带处理 `find ti,syscon-reset fail`（厂商 `resets` 属性我们没有，驱动已容忍）。

### 正面信号

- `infra_ao_ufs_bclk ... 11fa0000.ufs_ ... mphy ... mp` → **PHY 的 `mp` 时钟已消费** ✓
- `arm64_serror_panic` 持续为 **0** ✓，userspace 每次都能到 ✓

---

## ★★★★★★★★★★★★★★★★★★★★★ 第二十一轮（2026-10-01）：UFS 供电改为固定稳压器

第二十轮查明 UFS 卡在 `ufshcd_get_vreg: vcc get failed, err=-517`（EPROBE_DEFER），
即基树把 `vcc-supply` 接到 MT6359 的 `ldo_vemc`，而那个稳压器始终没注册出来。

驱动侧确认（`drivers/ufs/core/ufshcd.c:9707`）：

```c
int ufshcd_get_vreg(struct device *dev, struct ufs_vreg *vreg)
{
	if (!vreg) goto out;
	vreg->reg = devm_regulator_get(dev, vreg->name);
	if (IS_ERR(vreg->reg)) { ret = PTR_ERR(vreg->reg); dev_err(...); }
}
```

**既然 LK 已经替我们把 UFS 上电并初始化好了**（`[UFS] info: ufs init OK`、`VUFS12=1200000`），
内核这边不需要自己管那条轨道。于是照参考板 x6710 的做法（它的 `ufs_dummy_vcc`）加一个
always-on 的 `regulator-fixed`：

```dts
/ {
	ufs_fixed_vcc: regulator-ufs-vcc {
		compatible = "regulator-fixed";
		regulator-name = "vreg_ufs_vcc";
		regulator-min-microvolt = <3000000>;
		regulator-max-microvolt = <3000000>;
		regulator-always-on;
		regulator-boot-on;
	};
};
&{/ufshci@11270000} { vcc-supply = <&ufs_fixed_vcc>; ... };
```

反编译验证 phandle 归属：

| 属性 | phandle | 实际属主 |
| --- | --- | --- |
| `vcc-supply` | `0x52` | **`regulator-ufs-vcc`**（regulator-fixed, always-on）✓ |
| `phys` | `0xf2` | **`ufs_mphy@11fa0000`** ✓ |

其余三项仍是主线值：`clock-names = "ufs","ufs_sel"`、`freq-table-hz = <0 0 0 0>` ✓。

### UFS 报错链回顾（每一轮剥掉一层）

| 轮次 | 报错 | 修的是 |
| --- | --- | --- |
| ringfix12 | `arm64_serror_panic`（DMA 时外部中止） | ← 根因是 reserved-memory 缺 14 个区 |
| ringfix13 | `ufshcd_init_clocks: ufs-clk clk get failed` | 保留区迁移（SError 消失） |
| ringfix14 | `freq-table-hz len mismatch` / `phys node not found` | 时钟改主线 specifier |
| ringfix15 | `vcc get failed, err=-517` | freq-table-hz 表长 + phys |
| **ringfix16** | — | **vcc 改固定稳压器，应该能走完 probe** |

---

## ★★★★★★★★★★★★★★★★★★★★ 第二十轮（2026-10-01）：UFS 四层全过，现在**在等电**

ringfix15 之后，UFS 的报错链彻底走完：

```
phys node found                                                   ← phys 修好 ✓
（freq-table-hz len mismatch 消失）                                ← 表长修好 ✓
ufshcd_get_vreg: vcc get failed, err=-517                          ← -517 = -EPROBE_DEFER
platform 11270000.ufshci: deferred probe pending: ufshcd-mtk
ufs_mtk_phy 11fa0000.ufs_mphy: Unbalanced pm_runtime_enable!        ← 次要警告
```

**UFS 已经不再报错，而是在 defer 等它的 `vcc-supply`。** 而且 `clk_summary` 里能看到
`infra_ao_ufs_bclk ... 11fa0000.ufs_ ... mphy ... mp` —— **UFS PHY 已经吃上自己的 `mp` 时钟** ✓。

基树里 UFS 的 `vcc-supply = <0x4a>`，追踪 phandle 得到的是 MT6359 的
**`ldo_vemc`**（`regulator-name = "vemc"`，`mt6359regulator` 的子节点）。

它在 t=1.3s defer 一次、t=11.5s 再 defer 一次（仍是 -517）→ **说明 `mt6359-regulator`
始终没把自己的 regulator 注册出来**。

### 下一步（UFS 的最后一层）

1. 查 `mt6359-regulator.c` 是如何定位子节点的（`of_get_child_by_name` / 名字匹配），
   以及基树里 `mt6359regulator` 的子节点名（`ldo_vemc` 等）是否与主线描述符表一致；
2. 不一致的话，用 `mt6359.dtsi` 里的主线稳压器子树覆盖基树那一支；
3. 或者（更省事、先验证链路）把 `vcc-supply` 指向一个 `regulator-fixed` 或直接删掉该属性，
   看 UFS 能否走完 probe 出块设备 —— 因为 **LK 已经给 UFS 上电并初始化好了**
   （`[UFS] info: ufs init OK`，`VUFS12=1200000`），内核这边未必需要自己管电。

### 当前整体状态

| 子系统 | 状态 |
| --- | --- |
| 内核启动 / 可观测性 | ✅ |
| reserved-memory | ✅ 29 个固定区，SError 归零 |
| PMIC | ✅ RTC 注册，寄存器通路打通 |
| **UFS** | 🟡 **四层报错全过（compatible → 时钟 → 时钟表 → phys），卡在 vcc 稳压器未就绪** |
| 显示 | ⏸ 需 framebuffer 标 `no-map` + 收敛 reg + 改条纹绘制 |
| USB | ⏸ 已隔离 |

---

## ★★★★★★★★★★★★★★★★★★★ 第十九轮（2026-10-01）：UFS 逐层剥开 —— 时钟 → freq-table-hz → phys

ringfix14（时钟改主线 specifier）之后，UFS 的报错**逐层后退**，每一层都精确命中一个属性差异：

```
ringfix13: ufshcd_init_clocks: ufs-clk clk get failed, -2          ← 厂商时钟编号
ringfix14: freq-table-hz len mismatch → clock parse failed -22      ← 表长与时钟数不符
           phys node not found                                      ← 基树没有 phys
```

对照我们自己的 `mt6893.dtsi`：

| 属性 | 基树（live） | 主线驱动要的 |
| --- | --- | --- |
| `clock-names` | `"ufs-clk","ufs-unipro-clk","ufs-mp-clk","ufs-crypto-clk"`（4 个厂商时钟） | `"ufs","ufs_sel"`（2 个） |
| `freq-table-hz` | 8 个值（4 时钟 × min/max） | 4 个值（`<0 0>, <0 0>`） |
| `phys` | **无** | `<&ufsphy>` |
| `ufs_mphy` 的 `clocks` | **完全无** | `"unipro","mp"` |

ringfix15 全部补齐，DTB 已反编译验证：`clock-names = "ufs","ufs_sel"`、
`freq-table-hz = <0 0 0 0>`、`phys` phandle 解析成功 ✓。保留区仍是 29 个 ✓。

### 当前状态

| 子系统 | 状态 |
| --- | --- |
| 内核启动 / 可观测性 | ✅ |
| **reserved-memory** | ✅ **29 个固定区，SError 归零** |
| PMIC | ✅ RTC 注册，寄存器通路打通 |
| **UFS** | 🟡 属性差异逐层剥开中，最新一版补齐时钟表与 phys |
| 显示 | ⏸ 需把 framebuffer 标 `no-map` + 收敛 reg + 改条纹绘制方式 |
| USB | ⏸ 已隔离 |

---

## ★★★★★★★★★★★★★★★★★★ 第十八轮（2026-10-01）：★★★ 保留区迁移**确认修好 SError**，UFS 转为干净报错 ★★★

刷 ringfix13 后：

| 指标 | 结果 |
| --- | --- |
| **`arm64_serror_panic`** | **0**（到达 userspace 的那两次启动） |
| `init: step 1..7` | 全部完成 |
| `Freeing unused` / `CHOPIN MAINLINE` | 各 8 次 |
| `OF: reserved mem` | 170 行 |

**结论：`reserved-memory` 就是 UFS SError 的根因，迁移后消失。**（MT6895 注释里那句
"the kernel MUST NOT allocate from them or early boot crashes on memory corruption" 完全应验。）

### UFS：从 SError 变成两条明确的缺时钟

```
ufshcd-mtk 11270000.ufshci: ufshcd_init_clocks: ufs-clk clk get failed, -2
ufshcd-mtk 11270000.ufshci: error -ENOENT: Initialization failed with error -2
ufs_mtk_phy 11fa0000.ufs_mphy: error -ENOENT: Failed to get clk 'unipro'
```

原因：**基树的 clock specifier 是厂商时钟编号**（`clock-names = "ufs-clk",
"ufs-unipro-clk", "ufs-mp-clk", "ufs-crypto-clk"`，indices 0x50/0x3f/0x41/0x51），
和我们的主线 clock provider 对不上；`ufs_mphy` 节点甚至**完全没有 clocks**。

修法（ringfix14，取自 `mt6893.dtsi` 的主线定义）：

```dts
&{/ufshci@11270000} {
	clocks = <&{/infracfg_ao@10001000} CLK_INFRA_AO_UFS>,
	         <&{/topckgen@10000000} CLK_TOP_UFS_SEL>;
	clock-names = "ufs", "ufs_sel";
};
&{/ufs_mphy@11fa0000} {
	clocks = <&{/infracfg_ao@10001000} CLK_INFRA_AO_UNIPRO_SYSCLK>,
	         <&{/infracfg_ao@10001000} CLK_INFRA_AO_UFS_MP_SAP_BCLK>;
	clock-names = "unipro", "mp";
};
```

验证：DTB 里 `ufshci` 的 `clock-names = "ufs","ufs_sel"`、`ufs_mphy` 的
`clock-names = "unipro","mp"` ✓。

### 显示：仍差 `no-map`

```
simple-framebuffer 78e0c000.framebuffer: simplefb: cannot reserve video memory at [mem 0x78e0c000-0x7b5ebfff]
simple-framebuffer 78e0c000.framebuffer: probe with driver simple-framebuffer failed with error -12
init: /proc/fb : (empty)
```

`mblock-19-framebuffer`（0x77a20000+0x27e0000）在基树里是 **`no-map` 缺失**（`map non-reusable`），
所以 `request_mem_region` 与 System RAM 冲突、`ioremap` 拒绝 RAM。
下一步：把它改成 `no-map` **并把 simplefb 的 reg 收敛到保留区内**
（0x78e0c000+0x9f6000 = 一帧 1080×2400 stride 4352）。注意 `head.S` 的 K00–K03 条纹
用 `__va()` 写这块内存，改成 `no-map` 后必须同时改条纹的绘制方式。

---

## ★★★★★★★★★★★★★★★★★ 第十七轮（2026-10-01）：★★★ 按 MT6895 的做法迁移 reserved-memory —— 用 live FDT 做基树 ★★★

### 根因：我们的树少了 14 个固定保留区

厂商 boot.img 的 `/reserved-memory` **没有任何固定 `reg` 保留区**，只有 11 个动态分配节点。
那些带固定地址的 `mblock-N-*` 是 **LK 在运行时注入**的（LK 日志里的 `mblock_reserve-R` /
`mblock_create`）。而我们的内核把 LK 给的树**整个换掉**了 —— 等于把这些注入全丢了。

对照 live 树，我们缺 14 个区，其中包含 modem 的巨块：

| 缺失 | 大小 |
| --- | --- |
| `0x6e000000` ap_md_c_smem | **87 MB** |
| `0xfa850000` md_mem_usage | **57 MB** |
| `0xe4000000` md_mem_usage | 126 MB |
| `0x6c000000` adsp / `0x74000000` ap_md_nc_smem / `0xbe1c0000` vpu_binary | 14 / 24 / 28 MB |
| `0xe0000000`/`0xe3000000`/`0xbff70000`/`0x5ff00000`/`0x48170000`/`0x48080000`/`0x7ec00000` | 小 |

**内核被告知这些内存是自由的** —— 包括 modem 的 87 MB + 57 MB —— 于是把 DMA 缓冲/结构体分配进去，
EMI MPU 一挡就是外部中止。**这几乎肯定就是 UFS 那次 SError 的根因。**

### 做法：与 MT6895 完全一致

MT6895 的板级 DTS 注释写得很直白：

> Full MT6895 reserved-memory layout, taken **verbatim from the live device tree
> (/sys/firmware/fdt) of the running stock Android build**. These regions are owned by
> ATF/TEE/SCP/SSPM, modem, GPU and the display; **the kernel MUST NOT allocate from them
> or early boot crashes on memory corruption.**

于是改用 `Andorid-chopin.dts`（从跑着的 Android dump 出来的 live FDT，10513 行，
29 个固定保留区 + 14 个动态分配）：

- 新增 `arch/arm64/boot/dts/mediatek/mt6893-xiaomi-chopin-androidbase.dts`（基树，原样）
- `arch/arm64/kernel/chopin-board.dts` 改为 `/dts-v1/;` + include 基树 + 定向覆盖

覆盖内容（只改必须改的）：

```dts
&{/pwrap@10026000}          { compatible = "mediatek,mt6893-pwrap", "syscon"; ... };
&{/pwrap@10026000/mt6359-pmic}          { compatible = "mediatek,mt6359"; ... };
&{/pwrap@10026000/mt6359-pmic/mt6359regulator} { compatible = "mediatek,mt6359-regulator"; };
&{/ufshci@11270000}         { compatible = "mediatek,mt6893-ufshci", "mediatek,mt8183-ufshci"; };
&{/ufs_mphy@11fa0000}       { compatible = "mediatek,mt6893-ufsphy", "mediatek,mt8183-ufsphy"; };
&{/usb3@11200000} / &{/usb_xhci@11200000} / &{/usb0phy@11e40000} { status = "disabled"; };
&{/chosen}                  { framebuffer@78e0c000 { compatible = "simple-framebuffer"; ... }; };
```

**注意**基树的 GIC 是 `#interrupt-cells = <3>`（我们自己的 dtsi 是 4），所以 pwrap/pmic 的
`interrupts` 按 3 cell 写。

### 验证

| 检查 | 结果 |
| --- | --- |
| 内嵌 DTB 大小 | 29 KB → **277175 字节** |
| `mblock` 保留区 | **32 个** |
| reserved-memory 固定 `reg` 区 | **29 个**（与 live 树一致） |
| 内嵌进 Image | `Image + 0x113e000` 处按字节找到 |
| 覆盖后的 compatible | `mt6893-pwrap` / `mt6359` / `mt6359-regulator` / `mt6893-ufshci` / `mt6893-ufsphy` / `simple-framebuffer` 全部生效 |

### 踩过的坑（值得记下）

`chopin-board.dts` 一度被写到 `boot/dts/mediatek/` 而不是 `arch/arm64/kernel/`，
于是内核编的还是旧文件（DTB 停在 28993 字节、没有 `mblock`）。定位方法是看
`.chopin-board.dtb.cmd` 里的依赖列表 —— 它列的是旧的 `../boot/dts/mediatek/mt6893-xiaomi-chopin.dts`。

---

## ★★★★★★★★★★★★★★★★ 第十六轮（2026-10-01）：★★ **PMIC 起来了**，UFS 走到 DMA 再崩 ★★

cell 数修正（ringfix11）之后：

```
mt6397-rtc mt6359-rtc: registered as rtc0                                  ← PMIC 通了！
mt6397-rtc mt6359-rtc: setting system clock to 2086-09-30T04:43:42 UTC
ufshcd-mtk 11270000.ufs: phys node found
ufshcd-mtk 11270000.ufs: clk "ufs" present / clk "ufs_sel" present
ufshcd-mtk 11270000.ufs: device reset done                                 ← UFS 链路复位成功
scsi host0: ufshcd                                                         ← SCSI host 注册
```

`IRQ index 0 not found` 归零、`mt6359` 从 0 次变 24 次。**MT6359 的 MFD + RTC 注册成功，
说明 pwrap ↔ PMIC 的寄存器通路真正打通了** —— 这是后半段的第一块多米诺。

### 但 UFS 在第一次 DMA 时 SError panic

```
arm64_serror_panic ← arm64_is_fatal_ras_serror ← do_serror ← el1h_64_error_handler
swiotlb_map+0x1cc/0x22c (P)
dma_direct_map_sg ← __dma_map_sg_attrs ← scsi_dma_map ← ufshcd_queuecommand
scsi_queue_rq ← … ← scsi_vpd_inquiry ← scsi_attach_vpd ← ufshcd_async_scan
pstore: backend (ramoops) writing error (-28)
Rebooting in 15 seconds..
```

即 UFS **已完成链路复位并注册了 SCSI host**，在扫描 LUN / 发 VPD inquiry 的**第一次 DMA** 时
收到外部异步中止（SError / RAS fatal）→ panic。

排查结论：
- **不是 IOMMU 问题**：厂商 UFS 节点里**没有 `iommus`** 属性，走的也是裸 DMA。
- **供电是我方伪造的**：我们原来用 `ufs_dummy_vcc`（`regulator-fixed` 占位），而
  `mt6359.dtsi` 里其实早就有真家伙 —— `mt6359_vufs_ldo_reg`（`"vufs"`，1.7–1.9 V），
  厂商树正是把 UFS 的 `vcc-supply` 接到这条轨道。
- 厂商 UFS 还带 `vcc-fixed-regulator`，我们缺。

**ringfix12 改动**：`&ufs { vcc-supply = <&mt6359_vufs_ldo_reg>; }`（换掉伪造稳压器）。

### 待办
- UFS 若仍 SError：查 M-PHY（`ufs-phy@11fa0000`）的寄存器访问，以及 `vcc-fixed-regulator`。
- 另外 UFS 若需要 1.2 V（`ext_buck_vufs12`），厂商是靠 **I2C 上的外部 buck**（厂商专有驱动）——
  那就需要先起 I2C。

---

## ★★★★★★★★★★★★★★★ 第十五轮（2026-10-01）：关闭 USB 后日志变长，并抓到 PMIC 中断的真正原因

ringfix10 的 dump 里，最新那次内核启动不再是"停在 mtu3"，而是**多跑了 1099 行**：

| 簇 | mtu3 | `IRQ index 0` | mt6397 | ufshcd | simplefb | 行数 |
| --- | --- | --- | --- | --- | --- | --- |
| #0 | 18 | 2 | 2 | 95 | 6 | 298 |
| **#1（ringfix10）** | **0** ✓ | 8 | 8 | 186 | 24 | **1099** |
| #2 | 38 | 6 | 6 | 95 | 18 | 795 |

**簇 #1 = ringfix10**：`mtu3` = **0** → USB 确实被关掉了 ✓，日志从 795 行涨到 1099 行，
UFS/PMIC/显示继续 probe。

### 但 PMIC 仍报 `IRQ index 0 not found` —— 一个 cell 数错误

第十四轮加的 `&pmic { interrupts = <GIC_SPI 222 IRQ_TYPE_LEVEL_HIGH>; }` **只有 3 个 cell**，
而这块 SoC 的 GIC 是 **`#interrupt-cells = <4>`**（MTK 形式，`<type number flags 0>`），
dtsi 里所有其它节点都写成 `<GIC_SPI N IRQ_TYPE_LEVEL_HIGH 0>`。
cell 数不匹配 → `platform_get_irq()` 解析失败 → 报出误导性的 `IRQ index 0 not found`。

修正（ringfix11）：补上末尾的 `0`，已反编译确认 `pmic interrupts = <0x00 0xde 0x04 0x00>`、
`interrupt-parent` 解析到 GIC。

### 仍然待办

- **显示**：`simplefb: cannot reserve video memory` + `ioremap attempted on RAM pfn` —— 帧缓冲保留区
  需要 `no-map`，但那会让 `head.S` 里用 `__va()` 画的 K00–K03 条纹缺页，必须同时改条纹绘制方式。
- **USB**：mtu3 卡死的原因待查（PHY/时钟属性与厂商树的差异）。
- **UFS**：日志里已大量出现 `ufshcd`，需要在 PMIC 修好后确认它到底停在哪一步。

---

## ★★★★★★★★★★★★★★ 第十四轮（2026-10-01）：**内嵌 DTB 生效，外设开始 probe** ★★★

刷 `ringfix9-stockdt` 后，日志证实**内核用上了我们自己的树**：

```
Machine model: MT6891Z/CZA                                  ← LK 的树（setup_machine_fdt 读的）
CHOPIN-DTB: overriding LK FDT with embedded board tree (29007 bytes)   ← 替换成功
CHOPIN-CMDLINE: ... chopinport=ringfix8 ...                 ← 来自 CONFIG_CMDLINE（CMDLINE_FORCE=y）
OF: reserved mem: ... log-store@7ffbf000 / live-framebuffer@77a20000 /
                   atf@48c00000 / tee-shm@7a200000 / spm@76010000 /
                   mcupm@7ed00000 / gz@7ee00000 / sspm@7feb0000 ...
```

reserved-memory 列表全是**我们 DTS 的节点名**（厂商树叫 `mblock-N-xxx`）→ 铁证。

### 外设 probe 结果（首次真实读数）

| 驱动 | 日志 | 结论 |
| --- | --- | --- |
| **PMIC** | `mt6397 10026000.pwrap:pmic: error -ENXIO: IRQ index 0 not found` | **pwrap 已 bind 成功** ✓；但 `pmic` 子节点缺 `interrupts` → MFD 探测退出。修：加 `GIC_SPI 222 level-high`（厂商是 `0xde 4`） |
| **USB** | `mtu3 11201000.usb: IP version 0x1001(U3 IP)` → `max_speed: high-speed` **然后日志终止** | **mtu3 probe 卡死**（t=0.36s；之前用厂商树能到 4.5s + userspace）。它在 device initcall 里，挡住后面所有驱动 → 先 `disabled` |
| **显示** | `simplefb: cannot reserve video memory at [mem 0x78e0c000-0x7b5ebfff]` + `WARNING: ioremap attempted on RAM pfn` | 帧缓冲**不是 `no-map`** → `request_mem_region` 与 System RAM 冲突、`ioremap` 拒绝 RAM（与第二轮 logstore 那个坑同源）。参考板的 framebuffer 保留区是 **`no-map`** |
| **UFS** | 未出现 | 被 mtu3 的卡死挡住了，放开后才会看到 |

### 本轮改动（ringfix10）

1. `&usb_phy`/`&usb_host`/`&ssusb` → `disabled`（单变量：让被 USB 挡住的驱动露出来）
2. `&pmic { interrupt-parent = <&gic>; interrupts = <GIC_SPI 222 IRQ_TYPE_LEVEL_HIGH>; }`

### 已知待办

- **显示**：把 framebuffer 保留区改成 `no-map`（参考板做法）。**注意**：这会让线性映射不再覆盖它，
  于是 `head.S` 里用 `__va()` 画的 K00–K03 条纹会立刻缺页 —— 必须同时把条纹绘制改成
  `early_ioremap`/`ioremap`（或改用内核日志通道），所以要单独一轮小心做。
- **USB**：mtu3 卡死需要单独查（PHY/时钟属性与厂商树差异）。

---

## ★★★★★★★★★★★★★ 第十三轮（2026-10-01）：**照 MT6895 的做法把 DTB 塞进内核，绕开 LK/dtbo** ★★★

第十二轮查明"自建 DTB 必须提供覆盖层要的 60 个符号，否则 LK 不引导"。
用户指出 MT6895 分支早就解决了这个问题：**DTB 编进内核镜像，内核启动后自己替换掉 LK 给的树**
（因为动 dtbo 会让 LK 炸）。参考实现在本地 `work/ref-mt6895/raw/6895-branch/arch/arm64/kernel/`：

| 文件 | 内容 |
| --- | --- |
| `xaga-board.dts`（3 行） | `#include "../boot/dts/mediatek/mt6895-xiaomi-xaga.dts"` |
| `xaga-dtb.S` | `.section .rodata` + `.incbin "arch/arm64/kernel/xaga-board.dtb"`，导出 `_binary_..._start/_end` |
| `Makefile` | `obj-y += xaga-dtb.o` / `targets += xaga-board.dtb` / `$(obj)/xaga-dtb.o: $(obj)/xaga-board.dtb` |
| `setup.c` | `setup_arch()` 里紧接 `setup_machine_fdt()` 之后：`initial_boot_params = 内嵌树` + `early_init_dt_scan_chosen(boot_command_line)` |

参考注释解释了这个位置为什么正确：`setup_machine_fdt()` 已经把 LK 的
`/chosen bootargs` 和 `/memory` 吃进 memblock，之后所有环节
（reserved-memory 扫描、`paging_init()`、`unflatten_device_tree()`）都用**我们的树**。

### chopin 的实现

新增/修改：

- `linux-7.2.8/arch/arm64/kernel/chopin-board.dts`（3 行，include 我们的板级 DTS）
- `linux-7.2.8/arch/arm64/kernel/chopin-dtb.S`（`.rodata` + `.incbin`）
- `linux-7.2.8/arch/arm64/kernel/Makefile`（三条规则）
- `linux-7.2.8/arch/arm64/kernel/setup.c`（`setup_machine_fdt()` 之后做替换 + 重读 bootargs）
- 顺手修掉一个真 bug：我们的 `/chosen` **没有** `#address-cells/#size-cells`，
  导致 `framebuffer@78e0c000` 的 `reg` 按错误的 cell 数解析
  （dtc 警告 `invalid length (16 bytes) (#address-cells == 2, #size-cells == 1)`）。
  参考板子设了，我们照做。
- 板级 DTS 的 `bootargs` 现在**真正生效**（内核重读），标签由过期的 `ringfix1` 更新为 `ringfix9`。

验证：

| 检查 | 结果 |
| --- | --- |
| `chopin-board.dtb` 被编译 | ✓ 29007 字节 |
| 内嵌进最终 `Image` | ✓ 在 `Image + 0x113e000` 处按字节找到 |
| `model` = `Xiaomi Redmi Note 10 Pro (chopin)` | ✓ |
| `/chosen` 带 `#address-cells=<2> #size-cells=<2> ranges` | ✓ |
| `framebuffer@78e0c000` = `reg<0 0x78e0c000 0 0x27e0000>` 1080×2400 stride 4352 | ✓ |
| `mt6893-ufshci`/`-xhci`/`-mtu3`/`-tphy`/`mt6359` 节点 | ✓ |

注意：内嵌树**不需要** `__symbols__`（LK 看不到它，内核也不用），所以
`chopin-board.dtb` 不带 `-@`；`mt6893-xiaomi-chopin.dtb` 上的 `-@` 保留（无害）。

**因此 boot.img 仍然用厂商 dtb（stockdt）** —— LK 覆盖层照常成功、肯引导内核，
内核随后换成我们的树。这才是同时满足两边的唯一路线。

---

## ★★★★★★★★★★★★ 第十二轮（2026-10-01）：**为什么自建 DTB 永远起不来 —— 60 个符号**

第十一轮加了 `-@`（`__symbols__` 有了），错误**变了但依然失败**：

```
第十一轮之前: ERROR: ufdt_overlay_do_fixups():No node __symbols__ in main dtb.
第十二轮之后: ERROR: ufdt_overlay_do_fixups():Couldn't find 'mmc_fixed_3v3' symbol in main dtb
```

于是把设备上的 `dtbo_a` 分区抓下来分析（33554432 B，DTBO 容器，1 个 entry，72879 B）：

```
adb exec-out 'su -c "cat /dev/block/by-name/dtbo_a"' > work/dtbo_a.img
dtc -I dtb -O dts -o work/dtbo.dts work/dtbo_entry0.dtbo
```

**这个覆盖层需要基树提供 60 个符号**（`work/dtbo-required-symbols.txt` 已保存）：

```
mmc_fixed_3v3, mmc_fixed_1v8_io, reserved_memory, chosen, mmc0, odm, pio, i2c9,
accdet, irtx_pwm, spi1, mt6360_ldo, goodix_fp, mt6360_pmu, mt6360_pmic,
mt6360_typec, gps, i2c6, i2c7, nfc_*, mt_pmic_vcamio_ldo_reg, kd_camera_hw1,
i2c2/4/8, camera_af_hw_node, auxadc, md_auxadc, pmic_clock_buffer_ctrl,
i2c0/1/3/5, gpio_usage_mapping, gpio, mrdump_ext_rst, dsi_te, irq_nfc, touch,
subpmic_pmu_eint, tcpc_pd, smart_pa, md1_sim1_hot_plug_eint, mt_pmic_vtp_ldo_reg,
keypad, lk_charger, charger, cp_qc30, usbpd_pm, bat_gm30, usb0phy, mtkfb,
dispsys_config, dsi0, spi0, tboard_thermistor4
```

**我们那份精简 DTB 不可能提供这些。** 而且 LK 的日志显示：覆盖层失败后
`boot state: orange`、logo 照画（`mt_disp_update: 0x78e0c000`），但**此后再也没有任何
"jump/boot linux" 记录** —— 也就是说 **LK 在覆盖层失败后放弃了引导**。
（AVB 失败不是原因：stockdt 同样报 `boot_a: Hash of data does not match digest`，但照常启动。）

### 还顺带查到：内核 DTB 的真实来源

LK 日志里固定出现：

```
[118] [SBC] image lk_main_dtb auth init pass
[120] [LK_BOOT] lk_a name         : lk_main_dtb
[121] load_lk_dtb(): load lk_main_dtb dtb_img_size=201784
[1583] mblock_reserve-R start: 0x47c80000, sz: 0x80000 map:1 name:dtb_kernel_addr_mb
```

即 **LK 自己的 `lk_a` 镜像里带一份 201784 字节的 `lk_main_dtb`（过 SBC 校验）**，
另外把 boot.img 的 dtb 放到 `0x47c80000`（512 KB 预留区），**覆盖层就作用在这份 boot.img DTB 上**。

### 结论：自建 DTB 必须**以厂商树为底**

正确做法（下一轮执行）：

1. 用 `work/vendor.dts`（`port/out/stock-base.dtb` 反编译，8071 行，含全部 60 个符号）作为基树；
2. 在上面改我们需要的节点 —— **注意覆盖层会覆盖它自己 fragment 命中的节点**：

| 我们要改的 | 覆盖层是否也改它 | 能否生效 |
| --- | --- | --- |
| `pwrap@10026000`（→`mt6893-pwrap`） | **否**（无 fragment） | ✅ 能生效 |
| `ufshci@11270000`（→`mt6893-ufshci`） | **否** | ✅ |
| `ufs_mphy@11fa0000`（→`mt6893-ufsphy`） | **否** | ✅ |
| `usb_xhci@11200000` / `usb3@11200000` | **否** | ✅ |
| `usb0phy@11e40000` | **是**（fragment@65） | ❌ 会被覆盖回去 |
| `dsi0` / `mtkfb` / `dispsys_config` | **是**（73/74/75） | ❌ |
| `/chosen`（加 simple-framebuffer） | **是**（fragment@3） | ⚠️ 需谨慎（可能只加 bootargs，子节点或可保留） |
| `mmc0` / `auxadc` / `gpio` / `i2c*` | 是 | ❌（这些我们也不需要） |

3. 仍然要带 `-@` 编译并包成 DTBO 容器。

---

## ★★★★★★★★★★★ 第十一轮（2026-10-01）：**`owndtb` 失败的真正原因 —— 基树缺 `__symbols__`** ★★★

第十轮试了容器包装后的自建 DTB，**仍然是刷完立刻重启** —— 说明容器格式不是原因。
于是去翻 **LK 自己的日志**（它同样被转存进 expdb）。厂商树那次启动的日志里赫然写着：

```
[122] load_lk_dtb(): load lk_main_dtb dtb_img_size=201784
[133] dtbo_total_len: 0x11cef
[135] dtbo_size : 72879
[148] [PROFILE] ::: lvl(3) dtb_overlay takes 13 ms
ERROR: ufdt_overlay_do_fixups():No node __symbols__ in main dtb.
ERROR: ufdt_overlay_apply():failed to perform fixups in overlay
[1611] ufdt_apply_overlay() failed!
[1611] boot image dtb_overlay fail
```

**LK 在把 dtbo 分区的内容覆盖到 boot.img 里的基树之前，会按名字走 `__symbols__` 做
fixup。我们的 DTB 没有这个节点（编译时没带 `-@`），fixup 失败 → LK 直接不跳内核。**
厂商的 `stock-base.dtb` 里有 `__symbols__`，所以能启动 —— 与"stockdt 能跑、owndtb 立刻重启"
的观测完全吻合。

验证：

```bash
python3 -c "print(b'__symbols__' in open('port/out/stock-base.dtb','rb').read())"   # True
python3 -c "print(b'__symbols__' in open('build/chopin/.../mt6893-xiaomi-chopin.dtb','rb').read())"  # 修前 False
```

**修法**（内核自带的每目标 dtc 开关，`scripts/Makefile.dtbs:120`
`DTC_FLAGS += $(DTC_FLAGS_$(target-stem))`）：

```makefile
# arch/arm64/boot/dts/mediatek/Makefile
DTC_FLAGS_mt6893-xiaomi-chopin += -@
```

修后 DTB 由 28931 → **38696 字节**，`__symbols__` 存在，且节点无损失（`framebuffer` 仍在
`/chosen` depth 2；`ufs`/`ufs-phy`/`usb`/`phy`/`pwrap`/`pmic` 全部 `okay`）。

### 附：顺手修掉两个自造的可观测性障碍

1. **去掉 `initcall_debug`** —— 它的用途（定位卡住的 initcall）已经完成，留着只会用几千行
   `calling X`／`initcall X returned` 淹没日志窗口和回收出来的日志。
2. **initramfs 的日志过滤器屏蔽 initcall 噪音** —— 否则按 "regulator" 之类的关键词过滤时，
   命中的全是无关的 initcall 函数名（实测：回收出来的那段几乎全是噪音）。

---

## ★★★★★★★★★★ 第十轮（2026-10-01）：**外设阶段的前置阻塞点找到了**

### 0. 网络：GitHub 直连不通，**加速镜像可用**

`github.com` 直连超时（curl `000`），但 `ghfast.top` / `gh-proxy.com` / `ghproxy.net` /
`gitclone.com` 都返回 200，`git ls-remote` 走镜像也通，分支哈希与旧报告完全一致
（`dirty-x6710` = `280348eb…`、`master` = `c53f467…`）。于是两个参考仓库的完整源码都能取：

```bash
git clone --depth 1 -b dirty-x6710 https://ghfast.top/https://github.com/MT6893-Mainline/linux <dir>
curl -o x.c https://ghfast.top/https://raw.githubusercontent.com/<owner>/<repo>/<branch>/<path>
```

### 1. 关键事实：**厂商 DTB 的 compatible 与主线驱动几乎全不匹配**

内核一直用的是**厂商树**（LK 给的），把它反编译出来对账：

```bash
dtc -I dtb -O dts -o work/vendor.dts port/out/stock-base.dtb     # 8071 行
```

| 子系统 | 厂商节点 | 厂商 compatible | 我们的 match 表 | bind? |
| --- | --- | --- | --- | --- |
| PMIC | `pwrap@10026000` | `mediatek,mt6885-pwrap` | **没有**（那串只在注释里） | ❌ |
| PMIC 子节点 | `mt6359-pmic` | `mediatek,mt6359-pmic` | 表里要 `mediatek,mt6359` | ❌ |
| SPMI | `spmi@10027000` | `mediatek,mt6885-pmif` | 只有 mt6873/mt8195/mt8196 | ❌ |
| UFS | `ufshci@11270000` | `mediatek,ufshci` | 要 `mt8183/6893-ufshci` | ❌ |
| UFS PHY | `ufs_mphy@11fa0000` | `mediatek,ufs_mphy` | 要 `mt8183/6893-ufsphy` | ❌ |
| MMC | `mmc@11230000` | `mediatek,mt6885-mmc` | `mt8183-mmc` | ❌ 且 status=disabled |
| USB | `usb_xhci@11200000` | `mediatek,mt67xx-xhci` | 要 `mtk-xhci` | ❌ |
| USB PHY | `usb0phy@11e40000` | `mediatek,mt6885-phy` | 要 `mt6893-tphy` | ❌ |
| DSI | `dsi@1400e000` | `mediatek,dsi0` | 要 `mt6893-dsi` | ❌ |

**结论：要从厂商树（stockdt）切换到我们自己的 DTB，否则后半段一个都起不来。**
反过来，我们自己的树全部对得上：`mt6893-pwrap` + `mediatek,mt6359` 子节点 +
`mt8183-ufshci`/`mt8183-ufsphy` + `mt6893-mtu3`/`mt6893-xhci`/`mt6893-tphy`。

> **更正（第十一轮）**：先前说"pwrap 已经补过 `mt6885-pwrap`"是**错的** ——
> `grep -oE` 匹配到了注释里带引号的字符串。核对 `mtk-pmic-wrap.c:2455-2464`，
> 那串只存在于注释，match 表只有 `mt6893-pwrap`。所以厂商树下 pwrap 从来没 bind 过。

### 2. `owndtb` 失败的机理：**原厂 dtb 区是 Android DTBO 容器**

原厂 boot.img 的 `dtb` 区不是裸设备树：

```
magic=0xd7b7ab1e total=201848 header_size=32 entry_size=32
count=1 entries_offset=32 page_size=2048 version=0
entry[0]: dt_size=201784 dt_offset=64
```

新增 `port/tools/wrap-dtb.py` 复现它，**与原厂容器逐字节相同**（`orig == mine` → True）。
`build-images.sh` 现产出三种变体：`owndtb`（容器包装）、`owndtb-raw`（旧行为/对照）、
`stockdt`（厂商树）。

### 3. 板级 DTS：按唯一已知可用的同 SoC 板子（Infinix x6710）改写

- `#include "mt6359.dtsi"` —— 它写成 `&pwrap { pmic { ... } }`，正好挂到我们的 pwrap 下
  （**参考板与我们的最大 DTS 差异就是这一行**）
- `simple-framebuffer` 从根节点**移到 `/chosen`**（x6710 的写法；`drivers/of/platform.c:581`
  对 `/chosen` 有专门处理并会 `sysfb_disable`）
- UFS + `regulator-fixed` 的 `ufs_dummy_vcc`
- USB：`usb_phy` / `usb_host` / `ssusb`（peripheral, high-speed）
- `&mmc0` 保持 disabled（本机从 UFS 启动，厂商树里它也是 disabled）
- `model` 保持 `"Xiaomi Redmi Note 10 Pro (chopin)"` —— **判定用的是哪张树**

编译后 DTB 已逐项验证：`framebuffer` 在 `/chosen`（depth 2）、
`ufs@11270000`→`mt6893-ufshci`(okay)、`ufs-phy@11fa0000`→`mt6893-ufsphy`(okay)、
`usb@11201000`→`mt6893-mtu3`(okay)、`usb@11200000`→`mt6893-xhci`(okay)、
`phy@11e40000`→`mt6893-tphy`(okay)、`pwrap`→`mt6893-pwrap`(okay)、`pmic`→`mt6359`。

### 4. 配置缺口对照：**几乎全是"翻开关"，不是"移植驱动"**

| 结论 | 依据 |
| --- | --- |
| **存储零配置缺口** | `PHY_MTK_UFS=y`、`SCSI_UFS_MEDIATEK=y`、`SCSI_UFSHCD(_PLATFORM)=y`、`MMC_MTK=y` 全已开 |
| **USB 零配置缺口** | `USB_MTU3=y`（`DUAL_ROLE` 是 `GADGET` 超集）、`USB_XHCI_MTK=y`、`PHY_MTK_TPHY=y`、`EXTCON=y`、`USB_ROLE_SWITCH=y` |
| **真实缺口只有两处** | `RTC_DRV_MT6397`、`MEDIATEK_MT6359_AUXADC`（`m`→`y`）—— **已改** |
| `PHY_MTK_XSPHY` 是**假缺口** | `phy-mtk-xsphy.c` 只匹配 `mediatek,xsphy`（mt7988a 专用）；我们的 USB PHY 是 `mt6893-tphy` → `PHY_MTK_TPHY=y` |
| `SPMI_MTK_PMIF` 对 mt6893 原本是死路 | 只匹配 mt6873/mt8195/mt8196。子代理已按硬证据补 `mt6893-spmi`→`mt6873_pmif_arb`（`spmi-mtk-pmif.c:849`）；但 `CONFIG_SPMI_MTK_PMIF=m`，且 MT6315 驱动我们本来就缺，**PMIC 主线仍走 pwrap** |
| **显示是驱动级的墙，不是配置** | `mtk_drm_drv.c` 的 mmsys compatible 列表没有 mt6893，DSI 列表也没有，`mtk-mmsys.c` 也没有；我们 dtsi 里显示管线节点（`dsi@ mutex@ ovl@ rdma@ wdma@ …`）数量**全是 0** |
| 参考分支怎么出画面 | 它也没有 mt6893 DRM —— 靠 `FB_SIMPLE` + `/chosen` 里的 `simple-framebuffer` |
| 我们的 dtsi 与参考**几乎相同** | diff 仅 21 行 / 2 处，且都是表面差异 |

### 5. initramfs：**内核日志全量回收**

窗口只有 62 KB + 124 KB 且写满重来，`initcall_debug` 一开，启动中段的 probe 信息全被覆盖
（之前那些"0 命中"的判断就是这么来的）。现在 PID 1 用
`syslog(SYSLOG_ACTION_READ_ALL)` 取回**完整 512 KB printk 环**，回吐「开头 110 行」+
「驱动相关命中行（≤300 行）」，并 dump `/proc/fb`、`/proc/consoles`、simplefb 绑定状态、
平台设备清单（step 5/6/7）。

---

## ★★★★★★★★★★ 第九轮（2026-10-01）：**内核 + userspace 全部打通，观测面完整** ★★★

ringfix6 之后 expdb 里拿到了 userspace 的**完整自证**（12/12 次启动）：

```
 CHOPIN MAINLINE: userspace reached (PID 1)
init: kernel    : Linux version 7.2.8 (kily@Honor-FMI-76) ... #46 SMP PREEMPT
init: cmdline   : console=tty0 console=ttyS0,921600n8 earlycon=... chopinport=ringfix6 ...
init: step 1: mounting pseudo filesystems
init: step 2: pseudo filesystems mounted
init: DT model      : MT6891Z/CZA
init: DT compatible : mediatek,MT6893
init: DT bootprof   : yes (LK merged)
init: DT odm node   : yes (vendor dtbo applied)
init: step 3: device tree snapshot done
init: --- clk_summary (head) ---
   ulposc   0 0 0 260000000 ... Y
   osc_d20  0 0 0  13000000 ... Y
   osc_d10  0 0 0  26000000 ... Y
   ...
init: no shell in initramfs, staying alive
```

统计：`init: step` ×48（4 步 × 12 次）、`clk_summary` ×12、`DT model` ×12、
`no shell` ×12、**`short write` ×0**（没有丢输出）、`Unable to handle` ×0。

**至此整条链路的可观测性完全打通，且 userspace 侧也能自证：**

| 阶段 | 状态 |
| --- | --- |
| `head.S` K00–K03 | ✅ |
| `setup_arch` BC 00–08 | ✅ |
| `start_kernel` → 全部 initcall | ✅ |
| `free_initmem` → `run_init_process("/init")` | ✅ |
| PID 1 挂载 /proc /sys /dev /tmp | ✅ |
| 事件树读取（model/compatible/bootprof/dtbo） | ✅ |
| 时钟框架可读（`clk_summary`） | ✅ |
| **日志回传**：printk 钩子 + console + `/dev/kmsg` → 环 → LK → expdb | ✅ |

## 下一阶段（目标的后半段）：外设 bring-up

用当前这份日志把"还没有的东西"逐项确认了一遍（全是 0 命中）：

| 目标 | 现状 | 缺什么 |
| --- | --- | --- |
| **PMIC** | `mt6359`/`mtk-pmic-wrap` 探针 **0 命中** | **`mt6893` 不在 `mtk-pmic-wrap.c` 也不在 `spmi-mtk-pmif.c` 的 match 表里** → `pwrap@10026000`（`mediatek,mt6893-pwrap`）根本不会 bind。这是 PMIC 的前置 |
| **存储** | `mmc0:`/`ufs` **0 命中** | 驱动已编（`MMC_MTK=y`、`PHY_MTK_UFS=y`、`SCSI_UFSHCD_PLATFORM=y`），但 `mt6893.dtsi` 里 `mmc@11230000` / `ufs@11270000` 应是 `disabled`，板级未 `okay` |
| **显示** | `simple-framebuffer` / `fb0:` / `Console: switching to colour frame buffer` **全 0 命中** | `FB_SIMPLE=y`+`FRAMEBUFFER_CONSOLE=y` 已编，但驱动没有 bind（节点在根、`reg` 为 `0x78e0c000`，需查为何未 probe；也可改用 `DRM_SIMPLEDRM`） |
| **USB** | `usb` 命中都来自 initcall 名字 | 同上，需板级 `okay` + `mtu3`/`xHCI` 节点 |

**顺序按目标**：PMIC（`pwrap` 兼容 + `mt6359`）→ 存储（UFS/MSDC）→ USB → 显示（DSI）。

---

## ★★★★★★★★★ 第八轮（2026-10-01）：userspace 输出可见，但被 `/dev/kmsg` 两条限制截断

ringfix5 的 expdb 里**首次出现 userspace 的输出**（8/8 次启动都有）：

```
 CHOPIN MAINLINE: userspace reached (PID 1)
init: kernel    : Linux version 7.2.8 ... #45 SMP PREEMPT
init: cmdline   :        <-- 8 次全部停在这里
```

`kernel/printk/printk.c: devkmsg_write()` 里两条限制同时命中：

| 限制 | 后果 |
| --- | --- |
| `len > PRINTKRB_RECORD_MAX`（=1024）→ `-EINVAL` | `/proc/cmdline` ~1.6 KB 一次写完 → 内容全丢 |
| 默认 `devkmsg_log = ratelimit` → `___ratelimit()` 不过就丢弃 | 前几条之后全部静默消失 |

修正：cmdline 加 **`printk.devkmsg=on`**；`init.c` 输出改为**分块 ≤256 B** 并报短写，
再加 `init: step 1..4` 路点。

---

## ★★★★★★★★ 第七轮（2026-10-01）：**内核已启动到用户空间** ★★

修掉 `ioremap` 后日志不再被窗口溢出污染，内容变成**完整的一次启动**：

```
<6>Freeing unused kernel memory: 3392K
[    4.450919] Freeing unused kernel memory: 3392K
<6>Run /init as init process
[    4.451631] Run /init as init process
<7>  with arguments:            <7>    /init
<7>  with environment:          <7>    HOME=/   TERM=linux   chopinport=ringfix5
```

| 观察 | 含义 |
| --- | --- |
| 12 次 `Freeing unused kernel memory` + `Run /init as init process` | 12 次启动**都执行到 exec /init** |
| 日志覆盖到 `late_initcall`（`clk_disable_unused`、`genpd_power_off_unused`、`regulator_init_complete`、`fb_logo_late_init`…） | **全部 initcall 跑完** |
| `Attempted to kill init` / `Kernel panic` / `Failed to execute` 计数均为 0 | PID 1 没死、exec 没失败 |
| 全部 clk 驱动 probe 成功（`clk_mt6893_topck`/`infra_ao`/`mm`/`vdec`…） | 时钟子系统起来了 |
| 日志同时出现**带时间戳**（console 路径）与**不带时间戳**（钩子路径）两份 | 两个日志通道都在工作 |

**目标里的"使内核启动到用户空间"已达成。** 至此整条链路打通：
`head.S K00–K03 → setup_arch BC 00–08 → start_kernel → do_initcalls（全部）→
free_initmem → run_init_process("/init")`。

**附带的可见性修正**：`/dev/console` 绑在 dummy `tty0` 上，PID 1 的输出本来全部丢失。
`port/initramfs/init.c` 现在第一步把 fd 0/1/2 重定向到 `/dev/kmsg`，
于是 userspace 的输出也走 printk → 钩子 → 环 → LK → expdb。

---

## ★★★★★★ 第六轮（2026-10-01）：**根因 —— arm64 禁止 `ioremap()` 系统 RAM**

ringfix4 屏幕仍只有 1 条黄条（间距两格，可确定只有步 0），而步 0→步 1 之间只有
`ioremap(0x7ffbf000, 0x40000)`。翻源码立刻命中：

```c
/* arch/arm64/mm/ioremap.c */
        /* Don't allow RAM to be mapped. */
        if (WARN_ONCE(pfn_is_map_memory(__phys_to_pfn(phys_addr)),
                      "ioremap attempted on RAM pfn\n"))
                return NULL;
```

**arm64 不允许 `ioremap()` 系统 RAM**：WARN_ONCE + 返回 NULL。log_store 环是普通 DRAM
（reserved 但非 `no-map`），于是驱动器打印一条 WARN 后直接以 `-ENOMEM` 退出 ——
这既解释了 1 条黄条、没有 `[mtk-logstore]`，也解释了 dump 里的 `Unable to handle`/`Call trace`。

修正：改用**线性映射别名 `__va()` + `dcache_clean_poc()`** —— 正是 `paging_init()`
之前的面包屑一直在用的地址。`ioremap` 只保留给 SRAM（`0x11df00`，非 RAM，合法且默认关闭）。

---

## ★★★★★ 第五轮（2026-10-01）：修掉日志窗口溢出 + 黄条间距

`initcall_debug` 让一次启动打印 ~3500 行，暴露出**日志窗口"写满就停"**这个缺陷：
窗口满了之后就不再写，于是"最后一行"是**窗口写满的位置**而不是死点
（ringfix3 的 dump 因此把死点误指向一个 clk initcall）。

修正：窗口改为**"装不下就从头重来"**，保证最新输出从窗口开头连续存放 →
最后一行就是死点；黄条间距改为两格，可逐条计数。

---

## ★★★★ 第四轮实测（2026-10-01）：`initcall_debug` 一击命中 —— 卡在我们自己的驱动里

给 cmdline 加上 `initcall_debug`（`init/main.c` 顶部 `#define DEBUG` 是开着的，
所以每个 initcall 都会 `printk(KERN_DEBUG "calling  %pS @ %i")`，被日志钩子原样镜像）。
日志里最后一条是：

```
7calling  rcar_sysc_pd_init+0x0/0x320 @ 1
7calling  mtk_logstore_early_init+0x0/0xf0 @ 1     <-- 最后一条
```

**卡点在 `mtk_logstore_early_init()` 内部**，而且 `[mtk-logstore]` 自检行一行都没出
→ 卡在 `logstore_setup()` 的早期。最可疑的是**读 SRAM 头 `0x11df00`**：那是总线访问，
挂住表现为"静默死"而不是干净缺页。

三个修正（ringfix3）：

| # | 修正 | 原因 |
| --- | --- | --- |
| 1 | **SRAM 头访问改为可选、默认关闭**（`CONFIG_MTK_LOGSTORE_SRAM_HEADER=n`） | 低窗口本来就落在 LK 天然转存范围内，SRAM 头只买到高窗口；而它是这里唯一能"挂住总线"的操作 |
| 2 | **setup 期间不再关掉 printk 镜像**（原来 `ls = NULL`），并在 SRAM 读成功前绝不碰 SRAM | 原来的写法会让 `logstore_setup()` 内部出现的 oops 文本**全部丢失** —— 这正是"最后一条 calling 是我们、之后什么都没有"的成因 |
| 3 | **黄色步骤横条**（第 13 条绿色之后留空档） | 黄色数量 = logstore 驱动走到第几步，不用取 expdb 也能判定 |

另外确认了一件重要的事：**低窗口不依赖任何 SRAM 写入**（K00–K03 与 `[BC]` 都回来了），
所以即使完全放弃 SRAM 头，可观测性也不受影响。

---

## ★★★ 第三轮实测（2026-10-01）：**内核完整跑完 setup_arch，并一路开到 do_initcalls**

修掉我自己的 `__va` 回归后，`chopin-ringfix2-stockdt.img` 让内核走到了**远得多**的地方。
expdb 里拿到了**完整可读的内核日志**：

```
[BC 00] early_ioremap_init done      ... [BC 05] before paging_init
[BC 06] paging_init done             <- 从这里开始 vprintk_emit 钩子接管
<6>OF: reserved mem: 0x...ffe60000..0x...fffeffff (1600 KiB) nomap non-reusable mblock-31-DPM-reserved
... （全部 reserved-memory 逐条列出）
<6>Dentry cache hash table entries: 1048576 (order: 11, ...)
<6>Inode-cache hash table entries: 524288 (order: 10, ...)
<6>software IO TLB: mapped [mem 0xf4800000-0xf8800000] (64MB)
<6>SLUB: HWalign=64, Order=0-3, MinObjects=0, CPUs=8, Nodes=1
<6>rcu: Preemptible hierarchical RCU implementation.
<6>NR_IRQS: 64, nr_irqs: 64, preallocated irqs: 0
<6>GICv3: 832 SPIs implemented
<6>GICv3: CPU0: found redistributor 0 region 0:0x000000000c040000
<6>arch_timer: cp15 timer running at 13.00MHz (virt).
<6>Console: colour dummy device 80x25
<6>printk: legacy console [tty0] enabled
<6>Calibrating delay loop (skipped), ... 26.00 BogoMIPS
<6>VFS: Finished mounting rootfs on nullfs
<6>Timer migration: 1 hierarchy levels; 8 children per group; 1 crossnode level
```

| # | 结论 |
| --- | --- |
| 1 | **`setup_arch` 全部 9 个路点都过**（`BC 00`…`BC 08`），低窗口与高窗口都能读回 |
| 2 | **`vprintk_emit` 镜像钩子工作正常** —— 上面整段是内核真实 printk，不是 LK 的 |
| 3 | **GICv3 初始化成功**（832 SPIs、CPU0 redistributor 在 `0x0c040000`）、**arch_timer 13 MHz**、console、VFS cache、RCU 全部就绪 |
| 4 | 日志最后一行来自 `kernel/time/timer_migration.c` 的 **`early_initcall(tmigr_init)`** → 说明 `start_kernel` 跑完、`rest_init`/`kernel_init`/`do_basic_setup` 都到了，卡在**某个不打印的 early_initcall 里** |
| 5 | 因为我们的 `[mtk-logstore]` 一行没出现，死点就在 `tmigr_init` 与 `mtk_logstore_early_init`（`drivers/misc`）之间的 early_initcall 区间内 |

**下一步已就绪**：cmdline 加了 `initcall_debug`（`init/main.c` 里 `#define DEBUG` 是开着的，
所以每个 initcall 都会 `printk(KERN_DEBUG "calling  %pS @ %i")`，被我们的钩子原样镜像下来）。
日志里最后一个 `calling X` 就是卡住的那个 initcall。

> 注意：`chopin-ringfix2-owndtb.img` **不要用** —— LK 用我们自己的 DTB 时不跳内核（见第二轮第 5 条）。

---

## ★★ 第二轮实测（2026-10-01）：内核执行到 K03，以及一个我自己引入的回归

用户刷 `chopin-ringfix1-stockdt.img` 后屏幕出现 **4 条浅绿/深绿交替横条**，
`chopin-ringfix1-owndtb.img` 则**完全没有横条**。我用 adb 拉下 expdb 解析：

| # | 结论 | 证据 |
| --- | --- | --- |
| 1 | 内核执行到 **K03** 后停住 | 4 条横条 = `head.S` 的阶段 0/1/2/3 |
| 2 | **低窗口在真内核上同样有效** | 4 个 session 里找到 `K00`…`K03`，位置正是 ring `+0x800`…`+0x830` |
| 3 | GZ 以 64 位启动，**无异常** | `boot linux-64` ×2；`GZ kernel exception` 0 次 |
| 4 | **这次停在 K03 是我自己的回归** | 我把早期面包屑改成走 `logstore_append()` → `ring_write_va()` → `__va()`，而它在 `paging_init()` **之前**运行，线性映射尚不存在 → 第一个 `[BC]` 就取数异常。异常由内核自己的向量处理，所以 GZ 不报 —— 表现就是"4 条后静默死"。已改回 `early_ioremap` + `memcpy_toio`。 |
| 5 | `owndtb` 变体 **LK 根本没跳内核** | 对应 session 里没有 `jump to K64`；现阶段用 `stockdt` 变体 |

**教训（已写进代码注释）**：`__va()` 只在 `paging_init()` 之后可用；
`early_ioremap` 只在 `early_ioremap_init()` 之后、且**不能跨 `early_ioremap_reset()` 持有**。
这条边界在这个移植里踩了两次（第 3 条和第 4 条），现在两处都有显式注释。

另外给 `__no_granule_support`（`__enable_mmu` 里唯一可达的静默 `wfe/wfi` 死循环）
加了**蓝色**横幅 + `K09` 标签，用来区分"卡在 `__enable_mmu`"与"死在 MMU 打开后的盲区"。

---

## ★ 上机实测结果（2026-10-01，探针 + adb 取证）—— 可观测性已打通

用户刷了 `port/dist/boot-probe.img`（728 字节裸机载荷），屏幕顶部出现
**绿色带 + 青色带**；随后我用 `adb exec-out 'su -c "cat /dev/block/by-name/expdb"'`
把 40 MiB 的 expdb 分区直接拉回本地，用 `port/tools/parse-expdb.py` 解析。

| # | 结论 | 证据 |
| --- | --- | --- |
| 1 | **LK 确实加载、解压并执行我们的 arm64 镜像** | 屏幕出现阶段 1（绿）**和**阶段 2（青）色带 |
| 2 | **活的 framebuffer 是 `0x78e0c000`**，不是 `0x77a20000` | 色带是**绿**系；与 LK 日志最后一次 `mt_disp_config_fb_layer, addr = 0x78e0c000` 一致 |
| 3 | **SRAM 头 `0x11df00` 在 EL1 可访问**，不触发异常 | 阶段 2 是在写 SRAM 之后才画的，它出现了 |
| 4 | **expdb 转存通道可用**，且探测器的长度扩张被采纳 | 探针的 48 条横幅**全部**回来，覆盖 ring `+0x10000`…`+0x3f000` |
| 5 | **记录长度不跨复位保留** | 探针之后的 stock 启动只转存了约 126 KiB，`ring > 0x1D000` 的横幅一条都没回来 |
| 6 | 所以内核标记必须落在 **LK 天然转存的范围**内 | MT6895 已跑通的 `xaga-marker-writer.c` 把整个环放在 `0x2000..0x10000`，且**完全不碰 SRAM 头** |

**据此做的修正**：内核现在把**同样的内容写进两个窗口** ——
低窗口 `0x0800..0x10000`（MT6895 验证过的范围，不依赖长度字段）与
高窗口 `0x20000..0x3f000`（万一长度被保留就有用）；`head.S` 的 `K0n`
标签也两处都写。屏幕色带（13 个阶段）作为**完全独立**的第二通道保留。

另外，用探针的 48 个已知偏移把解析器**标定**了（`ring = implied - 0x1af3`），
现在报告的是**精确 ring 偏移**；自测见 `port/tools/test-parse-expdb.py`。

仍未取到的是**内核自己那次启动**的日志与色带 —— 见 `NEXT.md` 第 2 步。

---

## 〇、上一轮重大更正（先看这个）

**上一轮"内核零输出、死在 `early_initcall` 之前"的结论是错的。**

根因是内核自己的 `drivers/misc/mtk-logstore-console.c` 把 SRAM 头里的
`struct pl_lk_log` 当成了结构体起点，而它实际在 `+0x34`：

```c
/* 错的 */
off_lk = readl(l->hdr + offsetof(struct pl_lk_log, off_lk));   /* 读了 dram_buf.buf_point */
sz_lk  = readl(l->hdr + offsetof(struct pl_lk_log, sz_lk));    /* 读了 dram_buf.klog_addr */
```

LK 自己把它打印出来了，是权威证据：

```
LK_LOG_STORE:sram buff header 0x11df00,current log header 0x11df34, ...
                                                     ^^^^^^^ +0x34 = 0x11df34
```

厂商头文件 `drivers/misc/mediatek/log_store/log_store_kernel.h` 也完全吻合
（`sig/reboot_count/save_to_emmc` 12B + `dram_buf_header` 40B = 0x34）。

### 这个 bug 造成的全部后果

| 错误 | 后果 |
| --- | --- |
| `off_lk` 读到 `dram_buf.buf_point` | 得到垃圾值 |
| `sz_lk` 读到 `dram_buf.klog_addr` | 得到 printk 缓冲区的**物理地址**（如 `0x484aa0f8`），即一个巨大的伪长度 |
| `start_off = off_lk + sz_lk` | 溢出 |
| `logstore_setup()` 里第一次 `memcpy_toio()` | 往映射区外约 38 字节处写标记 |
| `logstore_sync_size()` 写进 `dram_buf.buf_size` | **LK 从来没被告知要多转存一个字节** |

也就是说：面包屑 `K00~K03` 和驱动自检行**从来没有进入 LK 会转存的那个区间**，
而且长度字段也写错了地方。所以"看不到输出"根本不是关于内核的测量结果。

### 旁证：为什么 LK 的 expdb 转存里一直只有 PL/LK 日志

厂商 DT 的 `/chosen/log_store` 是 `<0xdf1100 0x10000>`，而 LK 打印的是 `0x11df00`。
把三个值按 32 位字节序翻转，与 LK 的打印**三对三完全吻合**：

| DT 里的值 | byteswap | LK 打印/转存 |
| --- | --- | --- |
| `0xdf1100` | `0x11df00` | `sram buff header 0x11df00` ✓ |
| `0xd01100` | `0x11d000` | `data:0x11d000` (SYS_RAMCONSOLE_RAW) ✓ |
| `0x80000` | `0x800` | `SYS_RAMCONSOLE_RAW[0] 800/800@2000` ✓ |

即 **LK 写这两个 DT 属性时存在字节序 bug**，原厂内核照着 `0xdf1100` 去 ioremap，
签名校验必然失败 → 原厂内核的 log_store 其实也是坏的（这与"环里永远只有 PL/LK 日志"
完全一致）。同一个 DT 里的 `plat_dbg_info,base = <0x11d80c ...>` 没有被翻转，
所以正确地址就是 **`0x11df00`**，本移植的硬编码是对的。

---

## 一、本轮做了什么

### 1. 修好 log_store 驱动（`drivers/misc/mtk-logstore-console.c` 重写）

- 所有 SRAM 头字段统一走 `SRAM_CURLOG = offsetof(struct sram_log_header, dram_curlog_header)`；
- 不再"猜 LK 认为日志在哪结束"，改成**固定窗口 + 主动撑长度**：

  | ring 偏移 | 用途 |
  | --- | --- |
  | `0x20000`–`0x34000` | 注册 console 的文本（`CON_PRINTBUFFER` 回放全部历史 printk） |
  | `0x34000`–`0x3e000` | `vprintk_emit` 钩子的**同步**镜像（printk 时刻就落盘，卡死也不丢） |
  | `0x3e000`–`0x3f000` | 启动阶段面包屑 `[BC nn] ...`（纯存储，不经 printk） |

- 同时设置**三个**可能表达"日志有多长"的字段（全部按 `0x3f200` 上限）：

  | 字段 | SRAM 偏移 | 依据 |
  | --- | --- | --- |
  | `dram_curlog_header.sz_lk` | `+0x4c` | LK 打印的 `pl log size …@0x400` 那一族 |
  | `dram_buf.buf_point` | `+0x20` | 厂商头文件里它是"已用缓冲末端" |
  | `sram_log_header.reserve[0]` | `+0x5c` | 厂商头文件注释 `SRAM_RECORD_LOG_SIZE // sram_log record log size` |

  并同步 DRAM 里那份 `pl_lk_log` 副本（`+0x14`/`+0x18`）；上限刻意留在 `0x3f200`
  而不是整个 `0x40000`，避免触发 LK 的"环满"处理。
  `head.S` 的 `bc_tag` 宏和裸机探针也做同样的三处写入（已用 objdump 逐条核对：
  `str [x9,#76]` / `str [x9,#32]` / `str [x9,#92]`）。
- **不再碰 pstore 区（0x48090000）**：该区在 chopin 上确实不跨复位保留
  （见第三节第 1 条），上一轮的裸写只会把下一轮 Android 的 zone 弄坏。

### 2. 修好 `head.S` 面包屑（`arch/arm64/kernel/head.S`）

- 标签位置从 `0x1e000` 改到 `0x3e000`（驱动自己的窗口，确保会被转存），
  并在汇编里就把 `0x11df00+0x4c / +0x20` 撑到 `0x3f200`
  （已用 `objdump` 逐条核对立即数：`0x7fffd000`、`0x11df00`、`0xc85c`、`0x3f200`）。
- 去掉往 pstore 区的写入（死通道）。

### 3. 修正映射生命周期（`arch/arm64/kernel/setup.c`）

`mtk_logstore_attach_early()` 从 `early_ioremap_init()` 之后**移到 `paging_init()` 之后**：

- `paging_init()` 之前只能用 `early_ioremap()`，而那些 fixmap 槽会被
  `early_ioremap_reset()` 拆掉 —— 如果钩子一直握着它们，下一次 printk 就会
  取指/取数异常把内核打死；
- `paging_init()` 之后 ring（普通 DRAM，`map:1` 非 no-map）可以直接 `__va()` 访问，
  钩子写的是可缓存映射，所以每次写完都 `dcache_clean_poc()`（看门狗硬复位不刷 cache）；
- `ioremap()` 要等 `vmalloc_init()`（initcall 阶段）才可用，所以 console 仍在
  `early_initcall` 注册。

顺带发现：上一轮 setup.c 里两个"晚"面包屑用 `ioremap()`，而它们运行在
`vmalloc_init()` 之前 → **必然返回 NULL、静默失效**；现在全部可用。

### 4. 设备树与 config

- `simple-framebuffer` 从 `/chosen` **移到根节点**（LK 会重建 `/chosen`，放里面会被丢掉
  —— 这正是 MT6895 移植记录里的那条经验）。
- `defconfig` 命令行加 `panic=15`（失败 15 秒自动重启，测试循环自动化）和版本标记
  `chopinport=ringfix1`。
- 通过编译期常量核对：`logstore_set_length()` 编译出来后确实是
  `[x0+0x48]` 读、`[x0+0x4c]`/`[x0+0x20]` 写、DRAM 副本 `[x1+0x18]` 写。

### 5. 新增裸机探针（`port/fbtest/probe.S` + `build.sh` + `mkprobe.sh`）

584→728 字节、无 Linux、无 console 的 arm64 载荷，**一次刷机同时回答三个问题**：

1. LK 到底有没有加载/解压/执行 boot.img 的 kernel 字段？（画屏 → 屏幕即答案）
2. 哪一个 framebuffer 在交接时是活的？（5 个候选各画一种唯一颜色）
   - LK 日志显示它先把 OVL 层指到 `0x77a20000`，**最后又指到 `0x78e0c000`**，
     之后所有 `mt_disp_update` 都是 `0x78e0c000` —— 这解释了上一轮载荷
     （只画 `0x77a20000/0x76020000/0x76a20000`）为什么"屏幕完全没变化"。
   - 几何来自 LK 日志：`stride=4352`、`height=2400`、ARGB8888、一帧 `0x9f6000` 字节。
3. **EL1 能不能访问 SRAM 头 `0x11df00`？**
   - 载荷分两段画：先在 0–599 行画阶段 1 色带，然后写 log_store 环 + SRAM 头，
     再在 600–699 行画阶段 2 色带（另一种颜色）。
   - **只有阶段 1、没有阶段 2** ⇒ 访问 SRAM 时异常了 —— 这正是上一轮无法区分、
     也从未做过对照的那一类信息。
4. 顺带把 expdb 通道用"已知一定能写的写者"验一遍：
   如果连它的横幅都出不来，说明通道本身有问题，而不是内核没跑。

产物：`port/dist/boot-probe.img`（501760 字节）。
操作步骤见 `port/NEXT.md`。

### 6. 关掉 KASLR（与本 SoC 已跑通参考实现对齐）

`CONFIG_RANDOMIZE_BASE=y` 是从 arm64 `defconfig` 继承来的，而
**MT6893-Mainline（同 SoC、已跑通）明确关掉它**。这条很关键，因为
`arch/arm64/kernel/pi/kaslr_early_init()` 的落点在 head.S 里是：

```
K03 (3e030)  ->  bl __enable_mmu  ->  bl __pi_early_map_kernel   <- KASLR / 重定位在这里
                                       (之后才是 __primary_switched -> start_kernel)
```

也就是说 **KASLR 重定位正好落在"K03 之后、任何能打印的代码之前"** ——
与"内核完全没输出"的窗口完全重合，而且它是 `CONFIG_RELOCATABLE=y` 之外
唯一多出来的一整条早期代码路径。参考实现关掉它，bring-up 阶段它也没有任何收益，
所以已改为 `# CONFIG_RANDOMIZE_BASE is not set`。

补充一个容易忽略的点：`kaslr_early_init()` 在 DTB 里没有 `kaslr-seed` 时
会**回退到 CPU 的 `RNDR` 指令**取种子。厂商 DTB 里 `kaslr-seed = <0 0>`（即 0），
我们自己的 DTB 里根本没有这个属性 —— 两者都会走 RNDR 回退，也就是
KASLR 是否启用取决于 CPU 是否实现 ARMv8.5-RNG。这种"看运气"的行为在
盲目 bring-up 阶段是纯粹的噪声，关掉它让两条 DTB 路径都确定化。

### 7. 新增**屏幕**早期进度通道（不依赖任何日志链路）

这是本轮最有价值的一项：给内核加了一条**完全独立于 expdb / log_store 的可观测性**。

- **原理**：LK 把开机 logo 画进 framebuffer 后，显示控制器一直在扫出那块内存，
  而主线内核此时还没有任何显示驱动去重编程它 —— 所以**写像素立刻可见**。
- **安全性**（先验证再动手）：原厂 DT 的 `mblock-19-framebuffer`
  （`0x77a20000 + 0x27e0000`）**没有 `no-map`**，所以线性映射覆盖它，
  `__va()` 直接可用；而且地址就是普通 DRAM，**访问不可能缺页**。
  `0x78e0c000` 也在这个区间内。
- **实现**：`head.S` 的 `bc_tag` 宏（阶段 0–3）和驱动的 `bc_emit` /
  `mtk_logstore_breadcrumb_early`（阶段 4–12）在写完环标记之后，
  **各画一条 16 行横条**到两个候选地址：

  | 颜色 | 地址 | 说明 |
  | --- | --- | --- |
  | 绿色系（`0xff00ff00`/`0xff00a000` 交替） | `0x78e0c000` | LK 最后一次 `mt_disp_config_fb_layer` 指的地址 |
  | 红色系（`0xffff0000`/`0xffa00000` 交替） | `0x77a20000` | 原厂 reserved-memory 里叫 `framebuffer` 的那块 |

  **横条数量 = 走到的阶段号**（13 个阶段 ≈ 208 行，屏高 2400 行的 9%）。
  颜色告诉你哪块是活 framebuffer，数量告诉你能走到哪 —— 一次失败启动就够了。
- `paging_init()` 之前用 `early_ioremap`（20 页，远低于 `NR_FIX_BTMAPS = 65`），
  之后用 `__va()` + `dcache_clean_poc`（面板是 DMA master，走它自己的映射）。
- 开关：`CONFIG_MTK_BOOT_PROGRESS_FB`（默认 y，`=n` 即可完全不碰显示内存）。
- 已用 objdump 核对：`primary_entry` / `__primary_switch` 里四个阶段各生成两条
  填充循环，字面量 `0x78e0c000`/`0x77a20000`/`0x4400`（16×4352/4）与四个颜色
  全部正确。

### 8. 设备树：把活 framebuffer 保留下来，并把控制台指到证据支持的那块

- 新增 `reserved-memory/live-framebuffer@77a20000`（**不带 `no-map`**）：
  防止页分配器把正在扫出的显示内存发出去（那会直接把屏幕搞花）。
- `simple-framebuffer` 从 `0x77a20000` 改为 **`0x78e0c000`** ——
  LK 日志里最后一次 `mt_disp_config_fb_layer` 用的是后者，之后所有
  `mt_disp_update` 都用它。放错地址只是"控制台看不见"，不影响早期启动，
  所以这一改动不会污染下一次实测。

### 9. 新增 expdb 解析器（并自测）

`port/tools/parse-expdb.py` + `port/tools/test-parse-expdb.py`。
写它的过程中纠正了对 expdb 布局的两个错误假设，见第三节第 8/9 条。

### 10. 修正一处"会静默打死内核"的映射生命周期问题

见第一节第 3 条（`attach_early()` 移到 `paging_init()` 之后）。

---

## 二、仍然成立的事实（上一轮确证的，未变）

| # | 结论 | 证据 |
| --- | --- | --- |
| 1 | 引导链 **preloader → LK → GenieZone(EL2) → Linux(EL1)**，无 u-boot/UEFI | LK 日志 + GZ 日志 |
| 2 | **`bootopt=64S3,32N2,64N2` 必须在 boot.img 头部命令行里**，否则 GZ 按 32 位启动内核 | `boot linux-32 … x0=0x0` + `GZ kernel exception` vs `boot linux-64 … x0=0x47c80000` |
| 3 | LK 会把 boot.img DTB 的 `chosen/bootargs` **追加**进它构造的内核命令行 | 我们的 `earlycon=`/`chopinport=` 出现在 LK 命令行中间 |
| 4 | 设备**没有** `/dev/mem`、`/proc/kcore` | adb 实测 |
| 5 | `fastboot boot` 在该设备上不可用 | 实测 |
| 6 | LK 的 `decompress kernel image fail!!!` 是每次都有的常规消息 | 纯 Android 基线同样出现 |
| 7 | boot.img 的 dtb 字段是 DTBO 容器（大端头，1 条目） | `boot_a.img` 分析 |
| 8 | `expdb` 转存写在**分区末尾的 2 MB**（`file_size - 2MB`），不是分区开头 | 40 MB dump 里正文在 `0x2600000`+；20 MB 的尾部 dump 里正文在 `0x1200000`+ |

## 三、本轮新确证的（附证据）

| # | 结论 | 证据 |
| --- | --- | --- |
| 1 | **pstore 区（0x48090000, 0xe0000）在 chopin 上确实不跨复位保留** | 28 份 expdb dump（含纯 Android 基线）里 `SYS_PSTORE_RAW @0x2800` **全部为 0**；`SYS_RAMCONSOLE_RAW` 同样全 0 |
| 2 | 所有小的"总是转存"区域（`SYS_LAST_DRAM 0x484aa120/0x40`、`SYS_LAST_CPU_BUS`、`SYS_MCDI_DATA`）在每份 dump 里都是 0 | 同上，逐段统计 |
| 3 | LK 的 22 个 kedump 段**没有任何一段是内核 printk 缓冲区** | 完整 `kedump add:` 表 |
| 4 | 交接时活的 framebuffer 极可能是 **`0x78e0c000`** | LK 日志 `[999] [lk logo: mt_disp_config_fb_layer, addr = 0x78e0c000`，之后所有 `mt_disp_update` 都用它 |
| 5 | framebuffer 几何：**stride 4352 / 高 2400 / ARGB8888 / 一帧 0x9f6000** | `0x77a20000, x=0, y=0, pitch=4352, dst(0,0,540,2400)` + `display-others_logo_fb-addr sz 0x9f6000` |
| 6 | 早于 `paging_init()` 只有 `early_ioremap()` 能到 DRAM；`ioremap()` 在 `vmalloc_init()` 之前必然失败 | `start_kernel` → `setup_arch` → `mm_core_init` 顺序 |
| 7 | `early_ioremap()` 单次上限是 65 页（`NR_FIX_BTMAPS = 256K/PAGE + 1`），槽位 7 个 | `arch/arm64/include/asm/fixmap.h`；所以上一轮"映射 0x40000 会失败"的说法也是错的 |
| 8 | **pl/lk 日志不在 expdb 开头，而在分区最后 2 MB**；`log_emmc_header`（sig `0x785690ef`）在 `size-0x1000` | 40 MB dump 里正文全部落在 `0x2600000` 起；20 MB 尾部 dump 里落在 `0x1200000` 起，两者都等于 `size-2MB` |
| 9 | 最后 2 MB 是**逐次启动的 session 流**（每次约 125–235 KiB，一 boot 一段），不是环的逐字节镜像；session 起点 = preloader 日志首行 `[PMIC]Preloader Start` = 环偏移 `0x400` | 13 个 `[PMIC]Preloader Start`，间距 0x1e7ee–0x3909a；`[PMIC]BUCK_OC_SDN_STATUS` 只是 PL 日志中间一行，用它当锚点会整体偏移 |
| 10 | LK 的 22 个 `kedump add:` 段（`SYS_PSTORE_RAW`/`SYS_RAMCONSOLE_RAW`/… 以及环拷贝）在**每一份** dump 里都是 0 填充；找内核日志不能去那里 | 逐段统计 28 份 dump |
| 11 | `chopinport=` 与 `jump to K64` 在同一个 session 里只相隔约 1.1 KB，说明 session 内就是 LK 的连续日志流 | exp8.1：`ring+0x355c3` vs `ring+0x35a39` |
| 12 | `CONFIG_RANDOMIZE_BASE` 在参考实现里是关的，我们是开的；`kaslr_early_init()` 在无 `kaslr-seed` 时会回退到 `RNDR` | `work/ref/.config` vs 我们的；`arch/arm64/kernel/pi/kaslr_early.c` |

## 四、被证伪 / 已修正的判断（更新版）

| 曾以为 | 实际 | 如何证伪 |
| --- | --- | --- |
| 内核死在 `early_initcall` 之前（零输出） | **错**：log_store 驱动 SRAM 头偏移错 0x34，输出被写到了 LK 不转存的地方 | LK 打印的 `current log header 0x11df34` + 厂商头文件 |
| `early_ioremap(…, 0x40000)` 会因槽位不够而失败 | 不会：单次上限 65 页 | `NR_FIX_BTMAPS` 定义 |
| 用 `__va(0x11df00)` 读 SRAM 导致缺页 | 那是"即使不 wrong offset 也会错"的次级问题；主因是偏移 | 本次代码审查 |
| pstore 可以当通道 | 确认不行（28/28 份 dump 全 0） | 逐段统计 |
| framebuffer 写入会被 GZ trap | 待验证（探针将给出答案）；目前无任何直接证据 | —— |
| 上一轮其余判断（EFI/text_offset/gzip/体积/dump 忠实性） | 维持原结论 | 见上一轮记录 |

## 五、当前瓶颈与下一步

真正的瓶颈现在只剩**一次实测**：见 `port/NEXT.md`。

1. 刷 `port/dist/boot-probe.img` → 看屏幕颜色（30 秒，即时答案）；
2. 刷 `port/out/boot-owndtb.img` → 卡死后进 Android，**立刻**取 expdb；
3. 期望至少看到 `K00`…`K03`、`[BC nn] …`、`[mtk-logstore]`、`Linux version 7.2.8`。
   日志停在哪一行/哪个 BC，就是死点。

拿到日志后的 bring-up 顺序不变：PMIC（`pwrap@10026000` + `mt6359`）→ 存储
（`ufs@11270000` / `mmc@11230000`）→ USB → 显示（DSI）。

## Wi-Fi 状态（2026-10-18）

- 固件/电源/校准/RF/扫描/关联/DHCP 全部验证通过（`myron` @5745MHz，
  WPA2-PSK，inet 10.141.253.115/24）。
- 根文件系统：Alpine 3.24.2（initramfs root），含 `wpa_supplicant`/`iw`/
  `wifi-connect`；`apk` 源已切清华镜像。
- 内核侧保留 bring-up 用的 `CHOPIN-SCN`/`CHOPIN-RX` 打印与 pre-cal NVRAM
  等待（120s），后续可再收敛。
- Debian 13 (trixie) arm64 rootfs 也已跑通（systemd + `ttyGS0` serial-getty，
  root/chopin），GNOME 通过 `gnome-core` + `gdm3` 起在 1080x2400 DSI 上，
  当前 Mesa llvmpipe 软件渲染；CMA 从 32MB 提到 256MB 解决 mutter 的
  `DRM_IOCTL_MODE_CREATE_DUMB` ENOMEM。详见 `WIFI-PORT.md` 第十节。
- NetworkManager 已接管 `wlan0`（GNOME 面板可管理），连接 `myron` 自动重连；
  省电关闭（否则大流量下 apt hash/TLS 会随机损坏）。

## rootfs-only 分支（2026-10-21）

启动发行版不再依赖 bring-up init 的 USB 控制台分支：`port` 仓库的
**`rootfs-only`** 分支只保留一个极简 PID 1（`initramfs-rootfs/init.c`）：

- 挂 proc/sys/dev/devpts，等 `/dev/sdc75`（userdata）；
- 只读挂 ext4（失败再 f2fs），确认 `/sbin/init`，重挂 rw；
- 从 `/dev/sdc6` 复制 WiFi NVRAM；
- 挂新 root 的 proc/sys/dev/devpts/run/tmp，`MS_MOVE` + chroot + exec。

没有 USB/串口、recovery、诊断代码，所以**不插串口也能进 rootfs**。测试镜像
`out-rootfs-only/boot-stockdt.img`（md5 `aadfa0ec147f60c84368f5d940073e31`）验证通过。
`tools/mkinitramfs.sh` 用 `INIT_SRC=...` 选择这个 init；完整 bring-up 仍留在
`dev` 分支的 `initramfs/init.c`。

## GPU（Panfrost）状态（2026-10-21）

GPU 没有打通，详见 `GPU-PORT.md`：核心时钟已能到 350MHz、固件已就位，
但 SPM MFG 域在无 supply 时上电超时（-110），加
`buck_vgpu11`/`ldo_vsram_others` 会硬复位；MT6315 主线驱动又不绑定 vendor
`mt6315_7-regulator`，导致 SPM deferred。需要移植 vendor 的 MTCMOS 时钟门/
regulator 上电序列，工作量较大。当前继续用 Mesa llvmpipe 软件渲染。
内核 WIP 在 `dev` 分支 `466f013`，稳定启动仍用 `rootfs-only`。

## GPU 内核已打通（2026-10-22）

MTCMOS provider + nopd + 手动 rail 方案已让内核 panfrost 完整 probe：
`shader_present=0x660067`、`renderD128` 存在、`Initialized panfrost 1.6.0`。
用户态仍是 llvmpipe，因为 Mesa Panfrost 不支持 Mali-G77。详见
`GPU-HANDOFF.md`。触控左右半屏 Y 方向不一致，已在 NVT 驱动里按右半屏条件翻转。

## GPU 按真实 G77 接入（2026-10-23）

之前内核把 0x9080 折成 0x9000 当 G57，还把上报 ID 改成 0x9001 —— 但 Mesa
的 G57 表项是 0x9091/0x9093，0x9001 同样不认，所以一直 llvmpipe。现在内核
新增 `GPU_MODEL(g77, 0x9080)` 并优先精确匹配，上报真实 ID；Mesa 25.0.7 打
`git/mesa-25.0.7`（独立 Mesa 仓库）后交叉编译出 G77 版
`libgallium-25.0.7-2+deb13u1.so`（Debian 的 EGL 直接链 libgallium，
`dri/panfrost_dri.so` 只是 Xorg 空壳）。驱动先放 `/root/` 用
`LD_LIBRARY_PATH` 验证、再替换系统文件。实机已验证：`fastfetch` 显示
`Mesa Mali-G77 (Panfrost)`，GL 3.1 / GLES 3.1，`gpu sched timeout` 为 0。
GPU 这一串（MTCMOS clock provider、DT、panfrost 支持、首个 job 修复）已
整理进主分支 `7.2.8-mt6891-xiaomi-chopin`（`21d635c`/`e4b944b`/`8c66104`），
详见 `GPU-HANDOFF.md`。

## suspend 现状（2026-10-23）

主线 suspend 仍不能真正进入：mtu3 gadget 返回 -16 导致 abort，第一次 resume
花屏。当前用 systemd-sleep hook 自动重试一次 suspend，可恢复显示且不会睡死。
关闭 mtu3 wake 会睡死，禁用。90Hz 模式已从 connector 移除（DDIC 实际 91.2Hz，
与 DRM 90Hz 不匹配，会 underrun）。详见 `SUSPEND-NOTES.md`。

## PMIC IRQ / 电源键（2026-10-23）

电源键不可用：`mtk-pmic-keys` 与父中断 `mt6358-irq`（GIC 254）计数始终为 0，
插充电器也不触发；音量键（KPD）正常。已按原厂把 MT6359 子中断改成
SET/CLR 访问（WIP），但父中断线在主线 pwrap 下没有配置/路由。详见
`PMIC-IRQ-NOTES.md`。电源键暂用 `systemctl suspend` 或 GNOME 菜单代替。
