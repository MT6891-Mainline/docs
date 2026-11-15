# chopin Wi-Fi 移植（CONNAC2X2 SOC3_0 / conninfra）

> 本文记录把 **MT6893（chopin）的 Wi-Fi** 从原厂 4.14 内核搬到主线
> 7.2.8 的过程：借鉴哪份已现代化的代码、改了哪几处、DT 怎么接、
> 固件从哪来、怎么验。
>
> **状态（2026-10-18 上机）**：consys 完整 probe、A-die 6635、SPM 上电、
> ROM patch + N9 固件下载全部成功；`wlan0/wlan1/p2p0/ap0` 已创建，MAC 是
> 真实的 `04:e5:98:d6:4f:cd`。`fwload` 从 nvdata 的
> `/nvdata/APCFG/APRDEB/WIFI` 取到 NVRAM。见文末「上机结果」。

## 一、结论先放前面

- **不是从 4.14 硬搬**。MT6895-Mainline（`7.2-mt6895-xiaomi-xaga`）已经
  把同一套 downstream connectivity 栈（`conninfra` + `wlan_drv_gen4m`
  fullmac）现代化到 7.2，并且它的 `conninfra/platform/mt6893/` 与
  chopin 原厂 4.14 内核**逐字节相同**。所以直接以它为基线，再按 MT6893
  补三处平台差异即可。
- 整棵树的路径：`git/chopin-linux-7.2.8/drivers/misc/mediatek/connectivity/`
  （789 个文件，28 MB，仅构建进来的是 conninfra + connadp + connfem +
  wlan_adaptor + wlan_drv_gen4m；`bt/` 只带源码，不编译）。
- 编译产物：`build/chopin-net/arch/arm64/boot/Image`（48 MB）与
  `.../mediatek/mt6893-xiaomi-chopin.dtb`；符号表里有
  `conninfra_core_init` / `mtk_axi_probe` / `wlanProbe`。
- 新 DTB 里 `consys@18000000` / `wifi@18000000` 的属性已核对（见第三节）。

## 二、搬了什么、改了哪几处

基线：`port/mt6895-linux/drivers/misc/mediatek/connectivity/`（MT6895-Mainline
的 7.2 分支浅克隆，已包含 conninfra 的 `platform/mt6893`）。

相对基线改了 15 个文件（其中 3 个是换回原厂 `soc3_0` 版本），其余 774 个原样：

| 文件 | 改动 | 原因 |
| --- | --- | --- |
| `connectivity/Kconfig` | 6895 → 6893 措辞，`MTK_COMBO_CHIP_CONSYS_6893` | 只编 MT6893 平台 |
| `connectivity/Makefile` | `MTK_COMBO_CHIP=CONNAC2X2_SOC3_0`、`WLAN_CHIP_ID=6893`、`MODULE_NAME=..._6893` | 同上 |
| `conninfra/Kbuild` | **`MTK_CONNINFRA_CLOCK_BUFFER_API_AVAILABLE=0`** | 见下「电源」 |
| `conninfra/base/ring.c` | `__ring_segment_prepare` 加 `static` | 7.2 默认 `-Wmissing-declarations`，而 Kbuild 带 `-Werror` |
| `conninfra/platform/mt6893/mt6893_consys_reg.c` | `#include <linux/random.h>`；`get_random_int()` → `get_random_u32()` | 7.2 没有 `get_random_int()` |
| `conninfra/platform/mt6893/mt6893_pos.c` | `const static` → `static const`（5 处） | `-Werror=old-style-declaration` |
| `wlan_drv_gen4m/Makefile` | 加 `-I$(src)/../connadp/common` | 原厂该头在 `connectivity/common/`，这条线搬到了 `connadp/common/`；MT6895 只编 SOC7_0，所以基线没暴露 |
| `wlan_drv_gen4m/os/linux/gl_init.c` | NVRAM 路径 `mediatek/mt6895/WIFI` → `mediatek/mt6893/WIFI`；`XAGA-` 日志前缀 → `CHOPIN-` | 平台差异 |
| `wlan_drv_gen4m/os/linux/plat/mt6893/plat_priv.c` | `kalSetRpsMap()` 四处加 `#ifdef CONFIG_MTK_NET_RPS` | 该函数在 `CONFIG_MTK_NET_RPS` 下才存在；原厂 4.14 定义了它，主线没有 |
| `wlan_drv_gen4m/os/linux/gl_kal.c`、`chips/common/fw_dl.c`、`conninfra/platform/consys_hw.c` | 仅 `XAGA-` → `CHOPIN-` 日志前缀 | 可读性 |
| `wlan_drv_gen4m/chips/soc3_0/soc3_0.c` | 换回 **原厂 chopin** 版本；只把 3 处 `kalGetFwFlavor(&buf[0])` 改成基线的新签名 `kalGetFwFlavor()` | 见下「芯片侧」 |
| `wlan_drv_gen4m/chips/soc3_0/dbg_soc3_0.c`、`include/chips/hal_dmashdl_soc3_0.h` | 换回原厂 chopin 版本（逐字节） | 同上 |

树外配套（同样是本次新增）：

| 文件 | 作用 |
| --- | --- |
| `drivers/misc/mediatek/Kconfig`、`Makefile` | `source`/`obj-$(CONFIG_MTK_COMBO)` 接入 |
| `drivers/misc/Makefile` | `obj-$(CONFIG_MTK_COMBO) += mediatek/` |
| `include/linux/soc/mediatek/devapc_public.h` | `conninfra_dev.c` 在 `CONFIG_MTK_DEVAPC=m` 下需要；基线树有，我们树没有 |
| `arch/arm64/configs/chopin_defconfig` | `CONFIG_MTK_COMBO*` + `CFG80211/MAC80211/RFKILL=y` |
| `arch/arm64/boot/dts/mediatek/mt6893-xiaomi-chopin-ovl-wifi.dtsi` | consys/wifi 的 DT 补充（见下） |

### 芯片侧：为什么 `soc3_0` 用原厂版本，而核心用 7.2 化版本

MT6895 线虽然带着 `soc3_0` 的源码，但它只编 SOC7_0，**`soc3_0.c` 从未被
编译/验证过**；而且那份 `soc3_0.c` 和 `hal_dmashdl_soc3_0.h` 来自比
chopin 原厂更新的一个 MTK release，两处硬件配置不一样：

- `soc3_0_wfmda_host_tx_group[]`：AP DATA1/2、MD DATA1/2 对应的 WFDMA TX
  ring 号不同（原厂 `RING1/2/9/10`，新 release `RING2/4/10/12`）；
- `hal_dmashdl_soc3_0.h`：DMASHDL 的 group 1/4 refill enable、max/min
  quota 整组对调。

这两样都是**和固件/硬件配套的板级配置**，不是内核 API；driver 与固件
通过 init 命令交换 ring 号，配置错了不一定立刻报错，而是 DMA 行为异常。
设备上的固件是原厂 `1a` 一代（`WIFI_RAM_CODE_soc3_0_1a_1.bin`、
`soc3_0_patch_wmmcu_1a_1_hdr.bin`），所以这三份文件按「参考原厂内核」
用 chopin 自己的版本；它们不含主线 API（diff 里没有 `kalMem*`/`linux/`
/netdev 之类），唯一要改的是 `kalGetFwFlavor()`：原厂是
`void kalGetFwFlavor(char *)`，7.2 基线改成了返回 `const char *`，三处
调用点改一行即可。

`chips/common/*`（`cmm_asic_connac2x.c`、`fw_dl.c`、`pre_cal.c` 等）保留
7.2 化版本：那里的差异是通用修复（DMA reset、RXD group 校验、
radiotap、日志），不是板级寄存器表。

### 电源：为什么关掉 `CLOCK_BUFFER_API`

原厂 4.14 的 `&scpsys 1` 是一个 **MTCMOS-as-clock** provider，conninfra 用
`clk_prepare_enable()` 打开 CONN 域。主线没有这个 provider（`mediatek,scpsys`
不是 CCF clock），而基线代码在 7.2 上会选 `pm_runtime_get_sync()` 分支 ——
那需要一个 genpd，xaga 是用 `power-domains = <&spm MT6895_POWER_DOMAIN_CONN>`
接的；mainline `mt6893.dtsi` 里这个 CONN 域是**注释掉的**，chopin 的板级树
也没有 `spm` 节点。

`MT6893` 平台代码自带**直接写 SPM 的完整上电/下电序列**
（`consys_conninfra_on_power_ctrl_mt6893()`，`mt6893_pos.c` 的 `#else` 分支：
`SPM_POWERON_CONFIG_EN` → `SPM_CONN_PWR_CON` → 等 `SPM_PWR_STATUS(_2ND)` →
开总线时钟 → 解隔离/复位）。把
`MTK_CONNINFRA_CLOCK_BUFFER_API_AVAILABLE` 置 0 就走这条路，**不需要 genpd**，
也避免给板级树硬加一个 mainline `spm` 节点。这是本次移植最关键的一个选择。

## 三、DT：补的是「地址从哪来」

原厂 live DT 里 consys/wifi 节点已经有 `reg`/`interrupts`，缺的是**保留区
和电源**的引用 —— 原厂靠 LK 的 mblock 和 PMIC 直读，主线得在 DT 里说清楚：

```dts
&{/consys@18000000} {
        memory-region = <&{/reserved-memory/consys-reserve-memory}>;
        status = "okay";
        vcn13-supply = ...;            /* mt6359regulator/ldo_vcn13        */
        vcn18-supply = ...;            /* ldo_vcn18                        */
        vcn33_1_bt-supply = ...;       /* ldo_vcn33_1_bt                   */
        vcn33_1_wifi-supply = ...;     /* ldo_vcn33_1_wifi                 */
        vcn33_2_wifi-supply = ...;     /* ldo_vcn33_2_wifi                 */
};

&{/wifi@18000000} {
        memory-region = <&{/reserved-memory/wifi-reserve-memory}>;
        flavor_bin = "a";              /* 原厂固件是 1a 一套 */
        status = "okay";
};
```

- `consys` 的 `memory-region` 给 `conninfra/platform/emi_mng.c` 用：
  `ALLOCATE_CONNSYS_EMI_FROM_DTS=1` 时它从 reserved-memory 拿 base+size，
  不再读 `emi-addr`/`emi-size`，也不再依赖 LK 注入。
- `wifi` 的 `memory-region`（`shared-dma-pool`，0xf20000）给 AXI HIF 的
  预分配 DMA 用：`of_reserved_mem_device_init()` + `_init_resv_mem()`。
- `flavor_bin = "a"`：固件名由 `kalGetFwFlavor()` 拼出来，
  原厂文件是 `WIFI_RAM_CODE_soc3_0_1a_1.bin`、`soc3_0_*_1a_1_hdr.bin`，
  即 flavor `a` / IP set 1 / ECO 1。BT 节点本来就是 `flavor_bin = "a"`。
- 五个 `*-supply` 对应 `mt6893_pmic.c` 里
  `devm_regulator_get*("vcn13"/"vcn18"/"vcn33_1_bt"/"vcn33_1_wifi"/"vcn33_2_wifi")`；
  主线 `mt6359-regulator.c` 里 `ldo_vcn33_1_wifi`/`ldo_vcn33_2_wifi` 是
  `LDO_VCN33_1/2` 的 alias（`MT6359_LDO_NOOP`），所以照名字引用即可。

`mt6893_pmic.c` 有个坑：`devm_regulator_get_optional()` 失败返回的是
`ERR_PTR()`，代码却只判 `!reg`。所以 DT 里这五条 supply **必须存在**，
否则会把错误指针当 regulator 用。

## 四、固件与 NVRAM

Wi-Fi 起不来最常见的两个原因，这里都不是「驱动没编」：

| 文件 | 来源 | 去向 |
| --- | --- | --- |
| `WIFI_RAM_CODE_soc3_0_1a_1.bin` | `/vendor/firmware/` | `/lib/firmware/` |
| `soc3_0_patch_wmmcu_1a_1_hdr.bin` | 同上 | 同上 |
| `soc3_0_ram_wifi_1a_1_hdr.bin` | 同上 | 同上 |
| `soc3_0_ram_wmmcu_1a_1_hdr.bin` | 同上 | 同上 |
| `soc3_0_ram_mcu_1a_1_hdr.bin` | 同上 | 同上 |
| `wifi.cfg`、`conninfra.cfg` | 同上 | 同上 |
| `mediatek/mt6893/WIFI`（NVRAM 校准） | `nvdata` 分区（原厂） | `/lib/firmware/mediatek/mt6893/WIFI` |

`CFG_POWER_ON_DOWNLOAD_EMI_ROM_PATCH=1`（SOC3_0 默认）意味着 **ROM patch 和
WiFi RAM code 都要从 EMI 下载**，所以 `soc3_0_ram_*_hdr.bin` 不是可选项。

这些 blob 在设备上受 SELinux 保护（`shell` 读 `/vendor/firmware` 是
`EACCES`），但 **initramfs 的 `/init` 已经是 root**。为此给
`port/initramfs/init.c` 加了 `fwload` 命令：

```
fwload [dev [offset-sectors [size-sectors]]]
```

它把 `super` 分区里 vendor 逻辑分区的那一段做成 loop（带 offset），
只读挂到 `/vendor`，再把上表拷到 `/lib/firmware`。偏移来自 stock 上
`lpdump` 的输出（chopin，slot a）：

```
Name: vendor_a
Extents:
  0 .. 4428399 linear super 13246464
```

即 `/dev/sdc61` 的 sector 13246464，长度 4428400 sectors，已写成默认值；
换机/换槽时用参数覆盖。在 shell 里也可以直接 `ls /vendor/firmware`
核对。

**驱动侧已经配合好了**：`gl_init.c` 里 `wlanLoadNvramFirmware()` 失败不会
让 probe 失败，而是把「NVRAM 加载 + `mtk_wcn_wlan_func_ctrl(FUNC_ON)`」
丢进 delayed work，每 250 ms 重试、最多约 120 s，等 initramfs 把 blob 放好。
这段是 MT6895 线为同样的问题加的，原样保留。

## 五、构建

```bash
R=/home/kily/project/mt6893/git/chopin-linux-7.2.8
B=/home/kily/project/mt6893/build/chopin-net
make -C $R O=$B ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- chopin_defconfig
make -C $R O=$B ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j16 Image dtbs

KERNEL=$B/arch/arm64/boot/Image \
DTB=$B/arch/arm64/boot/dts/mediatek/mt6893-xiaomi-chopin.dtb \
STOCK_DTB=/home/kily/project/mt6893/port/out/stock-base.dtb \
  port/tools/build-images.sh port/out-wifi
```

构建日志：`build/net-build7.log`（`EXIT=0`），芯片侧换回原厂 `soc3_0`
之后是 `build/net-build9.log`（`EXIT=0`）。

> **陷阱：`make dtbs` 不重建内嵌树。** 运行时用的是
> `arch/arm64/kernel/chopin-board.dts`（`#include` 同一份板级 dts）编出来的
> `chopin-board.dtb`，由 `make Image` 链接；`make dtbs` 只更新
> `arch/arm64/boot/dts/...` 那份（给 LK/打包用）。改了 dts/dtsi 之后只跑
> `make dtbs` 会得到「boot.img 里的树是新的、内核里跑的是旧的」这种最难查的
> 状态。要么跑 `make Image`，要么 `touch arch/arm64/kernel/chopin-board.dts`
> 后再 `make Image`。本次 `pmic` 属性就踩过一次。

已确认 DTB（内嵌那份，`dtc -I dtb -O dts .../kernel/chopin-board.dtb`）：

```
consys@18000000: memory-region=<0x285> pmic=<0x108> status="okay" 5 条 *-supply
wifi@18000000:   memory-region=<0xb2> flavor_bin="a" status="okay"
```

## 六、上机步骤（已执行，记录操作）

1. `fastboot flash boot_a port/out-wifi/boot-stockdt.img && fastboot reboot`；
2. PC 上 `screen /dev/ttyACM0 115200`（gadget 枚举 `0525:a4a7`）；
3. shell 里 `fwload`，看：
   ```
   fwload: /dev/loop0 mounted on /vendor (ro)
     /vendor/firmware/WIFI_RAM_CODE_soc3_0_1a_1.bin      <size> bytes
     ...
   ```
4. **不要重启**，等 250 ms × N 的重试把 NVRAM 拉起来；`dmesg CHOPIN-NVRAM`、
   `dmesg CHOPIN-CONSYS`、`dmesg wlan`、`dmesg fw`；
5. 期望看到：`CHOPIN-CONSYS: probe done` → `A-die chipid detection` →
   `wlanProbe` → `N9 FW download` → `netdev wlan0`；
6. `ip link` / `iw dev`（initramfs 里没有 iw 的话，至少看
   `/sys/class/net/wlan0` 与 `dmesg wlan`）。

### 已知风险（按可能性排序）

1. **固件 ABI**：我们用的是 MT6895 线的 `soc3_0` 驱动代码（比 chopin
   原厂 4.14 的 `soc3_0` 新一个 release），而设备上的固件是原厂 1a 版。
   若 RAM code 头部/ROM patch 版本校验失败，第一处报错会在
   `CHOPIN-FWDL:` / `ConnacFormatDownload` 附近。备选方案：把
   `n10p/.../wlan/core/gen4m/chips/soc3_0/` 换回原厂版本再试。
2. **电源序列**：`MTK_CONNINFRA_CLOCK_BUFFER_API_AVAILABLE=0` 走 SPM 直写。
   若 `SPM_PWR_STATUS` 轮询超时，看 `mt6893_pos.c` 里的
   「Check conn_infra_on primary/secondary power fail」。
3. **EMI 大小**：DT 沿用原厂 `consys-reserve-memory` 的 0x450000；若
   RAM code + ROM patch 超过这个尺寸，`emi_mng` 会报 base/size 不对。
4. **NVRAM 路径**：`mediatek/mt6893/WIFI` 是按 MTK 命名推的，要在设备上
   `ls /vendor/firmware/mediatek/mt6893/` 或 `nvdata` 里核对。

## 七、与 MT6895 线的关系

| | xaga（MT6895） | chopin（MT6893） |
| --- | --- | --- |
| connectivity 基线 | 同一份 7.2 化代码 | 复制过来 |
| 平台目录 | `conninfra/platform/mt6895` | `mt6893`（与 chopin 原厂逐字节相同） |
| 芯片 | `CONNAC2X2_SOC7_0` / `WLAN_CHIP_ID=6895` | `CONNAC2X2_SOC3_0` / `6893` |
| 上电 | genpd `MT6895_POWER_DOMAIN_CONN` + `pm_runtime` | SPM 直写（本移植的选择） |
| 固件 flavor | `b_t` | `a` |
| NVRAM | `mediatek/mt6895/WIFI` | `mediatek/mt6893/WIFI` |

如果以后 mainline 给 MT6893 补上 CONN 电源域，可以把
`MTK_CONNINFRA_CLOCK_BUFFER_API_AVAILABLE` 改回 1、DT 加
`power-domains`，与 xaga 完全对齐。

## 八、上机结果（2026-10-18）

设备刷 `port/out-wifi/boot-stockdt.img`，USB ACM shell 里执行 `fwload` 后：

| 阶段 | 结果 |
| --- | --- |
| consys probe | `platform ops / reg mng / clk / emi / pmic / probe done` 全过 |
| A-die | `A-die chipid detection done, found chipid=[6635]` |
| SPM 上电 | `Polling chip id success (0x20010101)`；`wmmcu power-on` 后 N9 下载 |
| 固件 | `soc3_0_ram_wmmcu/wifi_1a_1_hdr.bin`、`soc3_0_patch_wmmcu_1a_1_hdr.bin`、`WIFI_RAM_CODE_soc3_0_1a_1.bin` 全部 `ret=0`，`downloadFirmware N9 ret=0` |
| NVRAM | `/nvdata/APCFG/APRDEB/WIFI`（6146 B）→ `/lib/firmware/mediatek/mt6893/WIFI`，`CHOPIN-NVRAM: loaded ..., state=READY` |
| netdev | `wlan0`（04:e5:98:d6:4f:cd）、`wlan1`、`p2p0`、`ap0`，`phy80211`/`rfkill` 都在 |

**仍存在、但不阻塞的报错**：

- `conninfra.cfg` 在 1.67s 被读，而那时 initramfs 的 `/init` 还没跑，所以
  `Get conf fail`。文件内容只有一行 `co_clock_flag=1`；MT6893 的
  `consys_co_clock_type_mt6893()` 读的是 `tcxo_gpio`（默认 0 → COTMS），
  当前 `conf == NULL` 时返回 -1、反而跳过了 COTMS 分支，而上电/固件都正常。
  **在弄清 COTMS 对主线（modem 未起）是否安全之前，不要把这个文件塞回去。**
- 开机时 pre-cal 失败（`WIFI pwr_on callback is not back`）：pre-cal 在
  `initWlan` 注册 ops 时就被触发，那时 NVRAM/固件还没就绪。50s 后真正的
  `wlanProbe` 自己上电 + 下载+启动固件成功，所以不阻塞 bring-up；后续可
  把 NVRAM 预置到 initramfs 让 pre-cal 在开机时就具备条件。
- `wifi_sigma.cfg` / `txpowerctrl.cfg` 缺失：可选覆盖配置，`fwload` 已补拷。

**待做**：`wifiscan`（initramfs 新增的 nl80211 最小实现）验证 RF；再到
关联/吞吐。

## 九、关联、DHCP 与上网（2026-10-18）

Alpine 3.24.2 rootfs 里离线装好 `wpa_supplicant 2.11` / `iw 6.17`
（`tools/build-alpine-rootfs.sh`，用 `apk.static --arch aarch64` 从清华源拉包，
`--no-scripts` 避免在 x86 主机上执行 aarch64 脚本），连接结果：

```
ssid=myron  freq=5745  wifi_generation=5
pairwise_cipher=CCMP  key_mgmt=WPA2-PSK  wpa_state=COMPLETED
address=04:e5:98:d6:4f:cd
```

`udhcpc` 随后拿到 `10.141.253.115/24`（网关 10.141.253.7），`wlan0` 状态
`UP,LOWER_UP`，IPv6 link-local 也起来了。也就是 **扫描 → 关联 → WPA2 四次握手
→ DHCP → 数据通路** 全部打通。

配套改动：

- init 里 exec Alpine `/bin/sh` 前设置 `PATH`，并加了 `/usr/bin/wifi-connect`
  一键脚本（`wpa_passphrase` + `wpa_supplicant -B` + `udhcpc`）；
- init 作为 PID 1 安装了 `SIGTERM/SIGINT/SIGUSR1/SIGUSR2` 处理函数：
  之前 busybox `reboot` 给 PID 1 发信号但没人处理（PID 1 未安装处理函数的信号
  会被丢弃），所以 `reboot` 卡住；现在收到信号后 `sync()` + `reboot(2)`；
- chopin 的 RTC 没有电池，开机会把系统时间设成 2086 年，导致所有 TLS 证书
  看起来过期（`apk` 报 `server certificate not trusted`）；`wifi-connect`
  在 DHCP 后检测年份异常并 `ntpd -q` 从 ntp.aliyun.com 校准；
- 参考 mt6895 的 init 增加了 **真实 rootfs 启动模式**：`chopin.root=` 或
  `/etc/chopin/rootfs` 指定分区（支持分区名，如 `userdata`），只读探测 →
  rw 重挂 → 把固件镜像到目标 rootfs → 挂 `proc/sys/dev/devpts/run/tmp` →
  `MS_MOVE` + `chroot` + `exec /sbin/init`；失败自动回落到 initramfs shell。
  chopin 的 `userdata` 是 `/dev/sdc75`（不是 mt6895 的 sdc86）。

## 十、Debian 13 rootfs 与 GNOME（2026-10-18）

Debian 13 (trixie) arm64 rootfs 已构建并跑通，systemd 作为 PID 1，USB 串口
`ttyGS0` 上跑 `serial-getty`，root 密码 `chopin`：

- 构建脚本：`tools/build-debian-rootfs.sh`。主机没有 root，采用
  `debootstrap --foreign`（user namespace + `TAR_OPTIONS=--no-same-owner`）
  → `dpkg-deb --build --root-owner-group` 重打包 → `proot -0 -q
  qemu-aarch64-static -r rootfs` 跑 second stage；`chown/chgrp` 在配置阶段
  用 wrapper 吞掉 service gid（mail/shadow/...）的 EINVAL；
- rootfs 烧到 `userdata`（`/dev/sdc75`，106G），init 只读探测 → rw → 镜像 WiFi
  固件 → `switch_root` 到 systemd；
- 串口登录：`chopin login: root` / `chopin`。

### GNOME

- `gnome-core`/`gnome-shell` + `gdm3`，默认 Wayland；
- 关键坑：`mtk_drm_gem_create()` 用 DMA/CMA 连续内存，默认
  `CONFIG_CMA_SIZE_MBYTES=32` 只够一个 1080x2400 dumb buffer，mutter 一开就
  `DRM_IOCTL_MODE_CREATE_DUMB failed: Cannot allocate memory`。已把
  `chopin_defconfig` 改成 **256MB**（cmdline `cma=` 会被 LK 覆盖，不可靠）；
- 没有 Panfrost 模块（`CONFIG_DRM_PANFROST=m`，rootfs 无 `/lib/modules`），
  当前是 Mesa llvmpipe 软件渲染，能用但慢；要 GPU 需把模块装进 rootfs +
  镜像 `mali_csffw.bin`；
- rootfs 模式下 init **不再跑 `display_bringup`**：它会把 DSI panel 留在
  powerdown，GDM 又在“模式已设置”的情况下不重新点亮，表现为黑屏；交给
  systemd/GDM 做 modeset。

### 网络：NetworkManager 接管

之前是 `wpa_supplicant@wlan0` + `systemd-networkd` + resolved，GNOME 面板里
没法管理。改成 NetworkManager 后 GNOME 设置/状态栏直接可用：

```sh
apt install -y network-manager
systemctl disable --now wpa_supplicant@wlan0 systemd-networkd
systemctl enable --now NetworkManager
nmcli device wifi connect myron password '12345678'
nmcli connection modify myron connection.autoconnect yes
```

- 连接配置落在 `/etc/NetworkManager/system-connections/myron.nmconnection`
  （0600），重启自动连；
- 省电：`/etc/NetworkManager/conf.d/wifi-powersave-off.conf` 里
  `[connection] wifi.powersave = 2`（2 = 关闭省电；默认 3 会在大流量下造成
  包损坏，表现为 apt hash mismatch / TLS 证书错误）；
- DNS 仍由 `systemd-resolved` 处理，`/etc/resolv.conf` 指向
  `/run/systemd/resolve/stub-resolv.conf`。

### apt 注意事项

- RTC 无电池，开机时间可能是 2069/2026-04；证书校验失败先 `date -s` 校准；
- 大流量下必须关 WiFi 省电，否则会出现 **同样文件大小但 SHA256 不符** 的
  `Hash Sum mismatch`（Tsinghua/USTC 都一样，不是镜像问题）；
- apt 的 HTTP pipeline 会放大这种损坏，建议固化：

```sh
printf 'Acquire::http::Pipeline-Depth "0";\nAcquire::https::Pipeline-Depth "0";\nAcquire::ForceIPv4 "true";\n' \
  > /etc/apt/apt.conf.d/99chopin
```
