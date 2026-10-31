# chopin（Redmi Note 10 Pro / MT6893）Linux 7.2.8 主线移植 —— 交付说明

- **日期**：2026-10-02
- **设备**：Xiaomi Redmi Note 10 Pro（`chopin`，MT6893/MT6891Z，Dimensity 1100）
- **内核基线**：Linux 7.2.8（`aarch64-linux-gnu-gcc 14.2.0`，SMP PREEMPT）
- **引导链**：`preloader → LK → GenieZone → Linux`（**无 U-Boot**；内核在 `setup_arch()` 中用内嵌 board 树替换 LK 的 FDT，`CONFIG_CMDLINE_FORCE=y` 使内嵌 cmdline 生效）
- **一句话结论**：在保持"可调试、可恢复、可复现"的前提下，这颗 SoC 的**充电、电池状态、显示、USB 设备模式、USB 主机模式**均已在主线上**实测可用**。

---

## 一、交付物

### 1.1 推荐镜像（功能最全，已实测）

| 镜像 | sha256 |
| --- | --- |
| `port/dist/chopin-ringfix61-host-settle-stockdt.img` | `84dd3a3ef88aa089da67f568fa62aad315ab4cafa1d4098d1fc3d6f0e4a3fdb0` |

```bash
fastboot flash boot_a port/dist/chopin-ringfix61-host-settle-stockdt.img
```

**DRM 显示（2026-10-03，代码完成、编译通过，⚠️ 尚未上机验证）**：

| 镜像 | sha256 | 说明 |
| --- | --- | --- |
| `chopin-ringfix88-livemirror-stockdt.img` | `3b1132038717390f118e063a79dfbdab9bce1554fe2bee482f42c574eb207b5e` | 首次真 DRM：MT6893 MMSYS/DSI/面板。测试与判读见 **`NEXT-DRM.md`** |

**回退良品**（任一都可用于二分定位或降级）：

| 镜像 | sha256 | 特点 |
| --- | --- | --- |
| `chopin-ringfix60-usbhost-stockdt.img` | `2e3057f1986da873c8e0bc913307acadada118a19359462da0b7aa6d92c5f98c` | 含 host 模式，切回 device 需手动 |
| `chopin-ringfix59-shell-selfheal-stockdt.img` | `3279b638a026d067d0363ddfb2596996e59f646012e824b3aeb29033bc4eaf97` | 无 host 模式，最稳 |
| `chopin-ringfix37-dedup-fbprobe-stockdt.img` | `bae616304a4e21817e9a1b2a221ff0b6921d42b2bc99bc78726d093277203404` | 历史干净对照 |

### 1.2 源代码

| 仓库 / 路径 | HEAD | 内容 |
| --- | --- | --- |
| `git/chopin-linux-7.2.8`（内核）| `5baaae9` | 内嵌 board 树、logstore 驱动、defconfig |
| `port/`（移植工程）| `1b686bc` | initramfs、构建工具、镜像、文档 |

关键文件与规模：

| 文件 | 行数 | 作用 |
| --- | --- | --- |
| `linux-7.2.8/arch/arm64/kernel/chopin-board.dts` | 748 | **内嵌 board 树**（所有硬件接线的唯一权威）|
| `linux-7.2.8/drivers/misc/mtk-logstore-console.c` | 1142 | 能扛复位的日志环 + 屏幕条纹 + **fbcon 缓冲镜像** |
| `port/initramfs/init.c` | 1932 | 诊断 init + 交互 shell + 拔插自愈 |
| `port/tools/build-images.sh` 等 13 个脚本 | — | 打包、DTB 提取、initrd 修补、仓库审计 |

---

## 二、验收结果（全部为设备实测，非推理）

| # | 能力 | 实测证据 |
| --- | --- | --- |
| 1 | **真正充电** | `mt6360-chg.2.auto: status=Charging, online=1, input_limit=2000000` |
| 2 | **电池状态可见** | MT6360 PMU ADC：`VBAT raw=3175 ×1250 = 3.968 V`（自 3.83 V 上升）／`IBAT raw=705 ×2500 = +1.762 A`／`VSYS 4.005 V`／`IBUS 1.825 A` |
| 3 | **充电电流可控** | 开机自动写入 2 A 限额，IBAT 实测 1.76 A 即其证据 |
| 4 | Type-C 端口状态可读 | `/sys/class/typec/port0/{data_role,power_role,port_type}` 正常可读（值见"已知限制"）|
| 5 | **USB 主机模式** | `xhci-mtk` 起来 → 枚举 **USB Hub**（4 ports）→ **YICHIP 无线键鼠**作为 `USB HID v2.00 Keyboard/Mouse` 注册 |
| 6 | host ↔ device 双向可靠 | 限时 host 窗口结束后自动切回 device，`pullup D+` + 串口重新连上 |
| 7 | **热重启后屏幕可读** | 面板扫另一缓冲时，fbcon 日志仍可见；不再"卡第一屏/只有红线" |
| 8 | **热拔插 shell 自愈** | 拔线后插回：设备重新枚举（PC `dmesg` 见新 `ttyACM0`），shell 自动重新接管控制台 |
| 9 | 基础设施 | UFS（128 GB LUN）、8 路 SPI、I2C0–I8、pinctrl、PMIC/RTC、看门狗喂狗、29+ reserved-memory 区 |

---

## 三、复现步骤

```bash
# 构建内核（改 defconfig 后必须重跑 defconfig —— 否则 Image 里是旧标签）
cd linux-7.2.8
make O=../build/chopin ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- chopin_defconfig
make O=../build/chopin ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j16 Image

# 构建 initramfs（★ 改过 init.c 必须先重跑）
cd .. && ./port/tools/mkinitramfs.sh && ./port/tools/build-images.sh
cp port/out/boot-stockdt.img port/dist/<新镜像名>.img

# 刷机
fastboot flash boot_a port/dist/<新镜像名>.img
```

**提交前必须做的一致性检查**：

```bash
port/tools/verify-drm.sh            # DRM 移植的全部本地校验（一条命令，见 NEXT-DRM.md）
port/tools/audit-repo.py            # 应输出 OLD COPY IN REPO: 0
port/tools/test-mmsys-routes.py     # DRM 路由表回归（模拟 LK 的 DSC 初值，见 NEXT-DRM.md）
port/tools/test-panel-seq.py        # 面板 53 条初始化命令 vs 厂商驱动 + 时序校验
git -C port status --short          # 应为空
git -C git/chopin-linux-7.2.8 status --short
```

`audit-repo.py` 会比对"工作树 vs 内核仓库"，并列出落后/缺失的文件 —— 改完内核代码记得 `cp` 进 `git/chopin-linux-7.2.8`。

**打包产物自检**（`port/tools/` 已有脚本可复用）：解压 Image == 构建的 Image、ramdisk == `port/out/initramfs.cpio.gz`、DTB 区 == `port/out/stock-base.dtb`。

---

## 四、实现要点

### 4.1 内嵌 board 树（`chopin-board.dts`）

以 LK 的树（`mt6893-xiaomi-chopin-androidbase.dts`）为底，用 `&{/path}` 覆盖成主线绑定：

- **pinctrl**：`mediatek,mt6893-pinctrl`，九个 `reg` 窗口 + `reg-names`，`#interrupt-cells = <2>`，删除 `reg_bases`/`reg_base_eint`
- **I2C0–I8**：`mediatek,mt8192-i2c` + `CLK_INFRA_AO_I2C<N>` + `CLK_INFRA_AO_APDMA`
- **MT6360（在 i2c5 / `0x11F00000`）**：
  - `subpmic@34` → `compatible = "mediatek,mt6360"`，`interrupts = <24 IRQ_TYPE_EDGE_FALLING>`（PMU IRQB = GPIO 24），`#interrupt-cells = <1>`，`wakeup-source`
  - 子节点 `charger`（`mediatek,mt6360-chg` + `richtek,vinovp-microvolt = <14500000>` + **`otg_vbus: usb-otg-vbus-regulator`（4.425–5.825 V，host 模式的 5 V 来源）**）
  - 子节点 `adc`（`mediatek,mt6360-adc`）、`tcpc`（`mediatek,mt6360-tcpc`，`interrupts = <25 IRQ_TYPE_LEVEL_LOW>` = PD IRQB = GPIO 25，含 `usb-c-connector`）
- **mtu3（`/usb@11201000`）**：`dr_mode = "otg"` + `usb-role-switch` + **`role-switch-default-mode = "peripheral"`** + `vbus-supply = <&otg_vbus>`
- **t-phy（`11e40000`）**：`ranges` 扩到 4 KiB（厂商模拟寄存器在 `+0xb04`）+ **`mediatek,force-session-valid`**（这块板子没把 PHY 的 VBUS 接到 Type-C）
- **UFS**：`mediatek,mt6893-ufshci` 兼容串 + 主线时钟名 + `freq-table-hz` + `phys` + `vmediatek,ufs-disable-mcq`
- **显示**：`/chosen/framebuffer@78e0c000`（LK 的缓冲）+ 相关 reserved-memory 修正（mblock-19 需 `no-map`）

> **注意**：`&{/path}` 覆盖**必须写在被覆盖节点之后** —— dtc 在解析时求值，前向引用会报 `Label or path ... not found`。

### 4.2 logstore 驱动（`mtk-logstore-console.c`）

1. **能扛复位的日志环**：DRAM 固定区 `0x7ffbf000`（256 KiB，mblock-6），调试接口 `/sys/kernel/debug/logstore/{low,high}`。
   **限制：只装得下一次启动的内核日志** ⇒ 事后复盘必须在启动别的镜像**之前**读。
2. **启动条纹**：早期用 `early_ioremap` 分块画（`NR_FIX_BTMAPS` 预算有限），示 **CAND0=绿 / CAND1=红**、`fb_note` 用黄色两块都写。
3. **fbcon 缓冲镜像**（本会话新增）：每 2 秒把 `CAND0 (0x78e0c000, fbcon 那块)` 复制到 `CAND1 (0x77a20000)`，
   使**面板扫哪一块都能看到文字**。读写都走驱动已持有的 **NonCacheable 映射**（`ioremap_wc`）+ `wmb()`。
   ⚠️ **绝对不要用 `__va` 访问该区**：`mblock-19` 是 `nomap`，不在线性映射里（首次实现这么做，故障发生在 kworker 中，直接导致看门狗复位、只能重刷）。

### 4.3 initramfs（`init.c`）

- **诊断**：DT 快照、clk_summary、中断表、framebuffer 状态、内核日志回收、USB/UDC/PHY 状态
- **交互 shell**（USB gadget 串口）：`help dmesg cat head tail find ls lsr grep write clk irq udc phy usb reboot poweroff i2cscan i2creg mt6360 power battery usbhost reconnect`
- **充电**：启动即写 2 A 限额（`input_current_limit` / `constant_charge_current`）
- **host 模式**：`usbhost <秒>` 开限时窗口（到点自动切回 device）；启动时把角色钉一次 `device` 作为保险
- **自愈**：
  - **读失败（EIO/EOF）不再退出**，改为等 `/dev/ttyGS0` 回来并重新接管 ⇒ 拔插后提示符自动恢复
  - **切回 device 后轮询等主机真的来配置（≤5 s）**，等不到才重绑 mtu3（重跑 `phy_init` + 重启 gadget，清掉 host 遗留状态）
  - 为什么不能单次读 UDC 状态：`configured` 可能是上一个 host 留下的**陈旧值**（曾因此漏判、切回后不可用）

---

## 五、关键内核配置（`chopin_defconfig`）

initramfs **不加载任何模块**，因此下列符号必须 `=y`（写成 `=m` 会被依赖链静默降级）：

```
CONFIG_MFD_MT6360=y
CONFIG_CHARGER_MT6360=y
CONFIG_MEDIATEK_MT6360_ADC=y
CONFIG_REGULATOR_MT6360=y
CONFIG_TYPEC=y
CONFIG_TYPEC_TCPM=y
CONFIG_TYPEC_TCPCI=y
CONFIG_TYPEC_MT6360=y
CONFIG_I2C_CHARDEV=y
CONFIG_I2C_MT65XX=y
CONFIG_BATTERY_BQ27XXX=y
CONFIG_MEDIATEK_MT6359_AUXADC=y
CONFIG_REGULATOR_MT6359=y
```

显示（DRM）新增的、同样必须 `=y` 的符号（`ringfix62`）：

```
CONFIG_DRM=y
CONFIG_DRM_MEDIATEK=y
CONFIG_MTK_MMSYS=y          # 它才会创建 clk-mt6893-mm 与 mediatek-drm 两个平台设备
CONFIG_PHY_MTK_MIPI_DSI=y
CONFIG_DRM_PANEL_K10A_36_02_0A=y
# CONFIG_MTK_CMDQ is not set  # 否则 MTK_MMSYS 被依赖链压到 =m（depends on MTK_CMDQ）
```

另有若干符号**不写在 defconfig 里**，而是由 `CONFIG_ARCH_MEDIATEK` 触发 Kconfig 的 `default` 自动选中，
构建后的 `.config` 里确实是 `=y`（例如 `CONFIG_PINCTRL_MT6893=y`，见
`drivers/pinctrl/mediatek/Kconfig: default ARM64 && ARCH_MEDIATEK`）。
**核对方法**：不要只看 defconfig，要看 `build/chopin/.config` 与 `build/chopin/drivers/.../*.o` 是否真的编出来了。

---

## 六、实现过程中修正的缺陷与失败路线

| 缺陷 / 问题 | 根因 | 处置 |
| --- | --- | --- |
| 显示"cannot reserve video memory" | mblock-19 未标 `no-map`、reg 过大 | 修正 reserved-memory |
| USB 设备模式从不枚举 | 厂商 PHY 驱动需 U2 0.9 V + 强制 OTG session 信号 | `mediatek,force-session-valid` |
| 每 ~31 秒自发重启 | toprgu 兼容串命中 `mt6589-wdt` | 改主线 `mt_wdt` |
| `Initramfs unpacking failed` | LK 留下的陈旧 `/chosen/linux,initrd-*` | 打包时用 `fdt-initrd.py` 修正 |
| 黑匣子只记一行 / 重复记录 | `win_write()` 未推进游标 + console 回调重复 | 修正（`d834b63`）|
| `mt6360` 在 i2c6 `-EBUSY` | `0x34` 被 `speaker_amp` 占用；实际在 **i2c5** | 以 base 树为准重定位 |
| **动 mtu3 到 OTG 后整机失去观测通道** | 判断依据是"代码里会读 `role-switch-default-mode`"，但当时无第二通道，串口一失守只能重刷 | **撤回**，教训：动唯一调试通道前必须先有替代通道 |
| **屏幕镜像首版把内核弄死** | 用 `__va` 读 `nomap` 的 framebuffer 区，故障在 kworker 中 | 改用驱动已持有的 WC 映射；从 `late_initcall` 起、周期 2 s |
| 热重启后"只剩条纹" | LK 把活动缓冲留在另一块，fbcon 只写 DT 那块 | 缓冲镜像（`62119d0`）|
| 拔插后 minicom 连上没有提示符 | shell 在拔线读错误时**退出**了（内核一直正常）| 读失败改为等控制台回来并重接管（`f0ab97e`）|
| 切回 device 后 PC 看不到 | 单次读 UDC 状态可能是陈旧 `configured` | 改为轮询等主机配置，超时才重绑（`f782c93`）|

**三条通用教训**：

1. **动唯一调试通道之前，必须先有能替代它的通道。**
2. **"代码里读得到该属性" ≠ "设备行为符合预期"** —— 未上机验证的机制不能为"切断观测"背书。
3. **`nomap` 保留区只能 `ioremap`，不能 `__va`**；屏幕镜像这类改动要从 **late initcall** 起、周期**秒级**。

---

## 七、已知限制与遗留问题

| 项 | 现状 | 影响 / 后续方向 |
| --- | --- | --- |
| **TCPC 的 CC/PD 检测不工作** | `power_role` 停在 `source`、PD IRQB（GPIO 25）无事件 | 自动角色切换、PD 协商不可用；host 模式因此走**手动/限时窗口**。修它才能实现"USB 正解"：充电器提供 VBUS 事件（extcon）+ gadget 用 `EXTCON_USB`，取代 `force-session-valid` |
| 显示依赖 LK 遗留状态 | 用 LK 留的缓冲 + 内核镜像维持可见 | 已在 `ringfix62` 中改为真 DRM（MT6893 MMSYS/DSI/k10a 面板），但**只到 30 Hz 不压缩**；60/90/120 Hz 需要 DSC，主线 mediatek-drm 尚无编码器（见 `NEXT-DRM.md`）|
| 无电池计 | BQ27Z561 在主线无支持（Android 用 MTK 私有栈，现读数为 `0xFFFF`）| 电池百分比/健康度不可得；可自写小驱动 |
| 相机 / NFC | 厂商树依赖 mux/EEPROM 驱动，日志里只有 `-EBUSY` 噪声 | 未实现 |
| USB host 测试有电气风险 | 手机作主机会**输出 5 V**；对面若是 PC 口（也在供电）即为双向对灌 | 建议用 **OTG 转接头 + U 盘/键鼠**；恢复串口请**直插 PC** |
| 日志环容量 | 仅约一次启动 | 复盘须在启动别的镜像前读取 |

---

## 八、文件索引

```
port/
  DELIVERY.md           ← 本交付说明
  NEXT-DRM.md           ← DRM 显示上机测试方案与判读（ringfix62）
  HANDOFF.md            ← 中心技术文档（背景 / 现状 / 每个坑与失败路线）
  SUMMARY.md            ← 按轮的成果史（最新为第二十八轮）
  initramfs/init.c      ← 诊断 init + 交互 shell + 自愈
  tools/                ← build-images.sh / mkinitramfs.sh / audit-repo.py /
                          test-mmsys-routes.py / test-panel-seq.py /
                          verify-drm.sh（DRM 回归 + 打包自检） /
                          fdt-initrd.py / extract-stock-dtb.py / mkdtbo.py /
                          wrap-dtb.py / parse-expdb.py / read-boot-log.sh ...
  dist/                 ← 镜像 + SHA256SUMS
  out/                  ← 打包产物（Image / initramfs.cpio.gz / stock-base.dtb）
linux-7.2.8/            ← 内核工作树
  arch/arm64/kernel/chopin-board.dts        ← 内嵌 board 树（硬件接线的唯一权威）
  arch/arm64/configs/chopin_defconfig       ← 配置
  drivers/misc/mtk-logstore-console.c       ← 日志环 / 条纹 / 缓冲镜像
git/chopin-linux-7.2.8/ ← 内核仓库（提交用；工作树改完需 cp 进来）
work/audit/             ← 构建日志、反编译产物、备份
```

---

**调试入口**（拿到设备后第一条命令）：

```bash
sudo minicom -D /dev/ttyACM0 -b 115200      # USB gadget 串口（首选通道）
chopin# help                                # 命令列表
chopin# battery                             # 电池电压/电流 + 充电器状态
chopin# usbhost 60                          # 限时 host 窗口（会自动切回）
chopin# drm                                 # 显示/DRM 状态 + 显示相关内核日志
```

**屏幕**是 USB 串口不在时的现场通道（内核日志已被镜像到面板正在扫的那块）；**日志环** `/sys/kernel/debug/logstore/high` 能扛复位，但只保留一次启动。
