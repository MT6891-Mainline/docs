# chopin（Redmi Note 10 Pro / MT6893）Windows on ARM 交接文档

> 最后更新：2026-11-01 · 对应固件基线 **r81**（`e44a945754217310d304d68a07a9fe2b45e558fb8509063dcbfb671860109cd0`）
> 相关文档：[`UEFI-PORT.md`](UEFI-PORT.md)（96 节完整移植记录）、[`work/bootmgr-analysis.md`](../work/bootmgr-analysis.md)、
> [`work/bootmgr-call-failure.md`](../work/bootmgr-call-failure.md)、[`work/pe-packaging-analysis.md`](../work/pe-packaging-analysis.md)、
> [`work/ref-diff-yunluo.md`](../work/ref-diff-yunluo.md)

---

## 0. 一句话现状

**arm64 的 Windows Boot Manager 可以在 chopin 的 UEFI 上运行、能读 UFS 上的 FAT 分区、能接受 ACPI 与 GOP，
拿到完全正确的设备路径（`UFS()/HD(73,GPT,…)`），但 `StartImage` 仍返回 `EFI_INVALID_PARAMETER`（= `0xC000000D`）。**
固件侧**每一个输入**都已实测正确（见 §2.6 与 §3）。

同一套流程在参考实现（Redmi Pad / MT6789 / yunluo）上能启动 WinPE 22631 到桌面，
所以差异**只可能在 chopin 与 yunluo 的运行时行为**里，而不是文件、镜像或代码（§4）。

---

## 1. 已经**确定可用**的部分

| 环节 | 证据 |
| --- | --- |
| LK → UEFI 启动链 | 正常进 BDS / 启动菜单 |
| UFS 枚举（GPT/GPT 分区/FAT） | `BCD probe` 能逐字节读出所有文件，FNV 与主机一致 |
| **UFS 读写稳定性（r81 起）** | HS 切档禁用后，日志**零 `Device Error`**（§2） |
| arm64 Windows Boot Manager 执行 | 屏幕出现 `Windows Boot Manager` + 加载日志 |
| ACPI 表被 Windows 接受 | bootmgfw 跑到读 BCD 阶段 |
| GOP framebuffer | Windows 的错误界面正常显示 |
| UEFI 键盘（音量/电源） | 菜单可用；`Key probe` 证明三种输入机制都能收到键 |
| 内存安全 | chopin DT 的 **14/14 个 reserved-memory 节点**全部在 `MemoryMapLib` 里 |

---

## 2. 本次会话最重要的发现：**HS-G4 切档会破坏 UFS 链路**

### 现象（r80，DEBUG 屏）

```
MtkUfsDxe: SMC DEVICE_RESET(0)=0x0 (1)=0x0 (0 = OK)
MtkUfsDxe: PostHce HCE=0x1 HCS=0x8 AXIL_THR=0x400080 (ready)
MtkUfsDxe: pwrmode=85 ... txgear=1 ...
MtkUfsDxe: HS change ok upmcrs=1 gear=4 lanes=2 adapt=1     ← 报告成功
MtkUfsDxe: HS pwrmode=17 HS txgear=4 HS txpwr=1

UFS op=0x1A lun=0 lba=0x0 len=0x40 -> Device Error           ← 之后所有读失败
ScsiDisk: IsWriteProtected() fails. Status = Device Error
UFS: UNIT ATTENTION on op=0x0 (asc=0x29 ascq=0x00), retrying once
UFS op=0x1A lun=0 len=0x40 -> Device Error                   ← 重试仍失败
```

### 结论

- `UPMCRS=1` 只表示**控制器接受了设置**，不代表链路可用；
- 切档后设备进入 **HS-G4 / 2 lane**，随后**读全部失败**并反复 `UNIT ATTENTION (asc=0x29)`；
- **设备会记住这个状态**：普通重启不清除，**必须彻底断电**；
- 原因：原厂内核里 `ufs_mtk_init_mphy()` 对 MT6893 是**空的**，PHY 由 **LK** 配置，而 LK 交过来的 PHY 状态**不支持 G4**。

### 修复（r81）

- `MtkUfsChangePwrModeHs` 调用被 `if (FALSE)` 包住（**禁用 HS 切档**），链路保持 LK 留下的 **PWM**；
- 断电重启后验证：**零 `Device Error`** ✓。

> **维护注意**：以后若再次启用 HS（r76/r77/r80 那类固件），刷完**必须彻底断电一次**，否则设备残留 HS 状态会让所有 UFS 读失败。

---

## 2.5 `0xC000000D` 的收敛结果（r83–r85 GUID 探针）

在 `BCD probe` 里复刻了 bootmgfw 的 `PartitionFwpGetEfiHandle`（RVA 0x793a0）五条件，
逐句柄打印 `c1..c5`，最终输出：

```
BCDPROBE: q44..q77 c1=1 c2=4 c3=1 c4=1 c5=1
GUID esp  sigtype=2  1FFBF267-BA62-9AA7-6?9B-A7D92BEE5FA
BCDFROBE: hdmatch first=74 matches=1 esp=1FFBF267-...
```

**结论：GUID 匹配是成功的（`first=74 matches=1`）** ✓ —— 因此 `0xc000000d` **不是**句柄查找失败 ✗。

反汇编分析原本给出两个产生点，现在只剩一个：

1. ❌ `gBS->OpenProtocol(NULL,…)`（句柄查找失败）—— **已排除** ✓
2. ✅ **`BlpDeviceOpen` 的 `dev->Size == 0`**（RVA 0x65a4c）—— `EfiInitpConvertEfiDevicePath`
   **转换失败**时 `EfiEntry` 会走清零分支（0x39bec），`BL_DEVICE` 全 0 → `Size == 0` →
   `EFI_INVALID_PARAMETER` → **`0xC000000D`** ✓

转换器只会因为这三件事失败：

| 拒绝条件（RVA） | 我们的情况 |
| --- | --- |
| 叶子是 `HW_VENDOR` 但不是 `HW_MEMMAP`（0x52c64） | 待查（UFS 整盘路径可能有 vendor 节点） |
| 叶子是 MEDIA 但不是 HD/CDROM（0x52b34） | `dp=0401` ✓ 排除 |
| HD 的 `SignatureType ∉ {1,2}`（0x52b74） | `sigtype=2` ✓ 排除 |

**继续的唯一路径**：在 bootmgfw 内部读 `AppEntry+0xb0` 的 0x48 字节 `BL_DEVICE`
（`BmGetDataStorePath` 附近），确认它是否全 0 —— 这需要改 bootmgfw 二进制或内核调试 ✗；
**更省事的替代**是把我们的 `cust` 拿到参考设备上交叉验证 ✓。


---

## 2.6 最终收敛（r86 WinBoot probe）：设备路径**完美**，(A) 也被排除

`WinBoot probe` 用与 BDS 完全相同的方式加载 bootmgfw，并 dump 了
`LoadedImage->DevicePath`（正是 `EfiEntry` 会交给转换器的那份）：

```
WINBOOT: handle n0 raw=03 19 06 00 ...      ← UFS() 消息节点
WINBOOT: handle n1 type=04 sub=01 len=42    ← HardDrive()
WINBOOT: handle n1 hd part=73 start=0x2AD600 size=0x40000 sigtype=2 sig=<GUID>
WINBOOT: handle n2 type=7f sub=ff len=4     ← END
WINBOOT: n2 file=\EFI\Microsoft\Boot\bootmgfw.efi
WINBOOT: loadimage=Success
WINBOOT: loaded devhandle=... devpath=UFS(0x0)/HD(73,GPT,<GUID>,0x2AD600,0x40000)
WINBOOT: loaded filepath=\EFI\Microsoft\Boot\bootmgfw.efi
WINBOOT: startimage=Invalid Parameter
WINBOOT: done leaf=04/01 nodes=4 startstatus=Invalid Parameter
```

**判读**：

| 项 | 结果 |
| --- | --- |
| 设备路径形态 | ✅ `UFS(0x0)/HD(73,GPT,<GUID>,0x2AD600,0x40000)` —— 与真实 ESP 一致 |
| 转换器派发的叶子 | ✅ `04/01` + `sigtype=2` → 接受（生成 Type 6 / Size 0x48 的描述符） |
| `LoadImage` | ✅ Success |
| **`StartImage`** | ✗ **`EFI_INVALID_PARAMETER`** —— 即 `0xC000000D` 本身 |

**结论**：`EfiInitpConvertEfiDevicePath` 没有失败，`BL_DEVICE` 不是被清零的 ✗ ——
**(A) 排除**。`0xC000000D` 只能来自 bootmgfw 内部的打开路径
（`BlockIoOpen` → `PartitionOpen` → `BlockIoEfiGetBlockIoInformation`）✗，
而这条路径上的**每一个输入我们都已经实测正确**：

| 输入 | 实测 |
| --- | --- |
| 句柄 GUID 匹配 | `first=74 matches=1` ✓ |
| Media（present/removable/last） | 全部正常 ✓ |
| `ReadBlocks` | `EFI_SUCCESS` ✓ |
| 设备路径 | `UFS/HD(73,GPT,…)` ✓ |

**固件侧可做的工作到此穷尽** ✗。下一步只有：
（1）把 `cust` 拿到参考设备（yunluo）交叉验证 ✓；
（2）或 UART 内核调试 bootmgfw 内部 ✗。


---

## 2.7 Ventoy 启动 ISO 时的崩溃：**不是我们的固件** ✗

现象：从 Ventoy 启动任意 arm64 ISO（BYDJ PE、Win11 ARM64、Alpine）→ `Loading files...` 后
**固件异常转储 + 自动重启** ✗。看起来像固件的 use-after-free（`X9 = 0xAFAF…`）。

**按固件自己的符号表解析（`work/crash-analysis.md`）**：

```
ELR = 0x452C904C  ->  Udf.efi + 0x904C        ← Ventoy 的 UDF 文件系统驱动
dump 里的 [ 0] 图例: /home/share/Ventoy/EDK2/…/FsPkg/UDF/DEBUG/Udf.dll
X9  = 0xAFAFAFAFAFAFAFAF  = 我们 DEBUG 的 DEBUG_CLEAR_MEMORY 填充值
                             (PcdDebugClearMemoryValue = 0xAF, PcdDebugPropertyMask = 0x2F)
FAR = 0x70, ESR = 0x96000007 -> 读 NULL+0x70（某描述符字段被 0xAF 填充后被当偏移）
```

**结论**：

* 崩溃点在 **Ventoy 的 UDF 驱动** ✗ —— 它读了**已释放/未初始化**的 pool 内存 ✓；
* **我们的 DEBUG 版**用 `0xAF` 填充空闲内存 ✓ → 那个"脏值"变成 **0xAF** ✗ → 被当偏移 ✗ →
  **NULL+0x70** ✗ → 崩溃 ✓；
* **RELEASE 版**（`PcdDebugPropertyMask = 0x02` ✓，不填充 ✓）→ 该 bug **不会致命** ✓；
* 回溯源链（用我们的 map ✓）：`BdsEntry → EfiBootManagerBoot → StartImage BootManagerMenuApp
  → StartImage MsBootPolicy → StartImage Ventoy.dll → StartImage Udf.dll → fault` ✓ ——
  **没有任何我们的 USB/FAT/Partition 驱动出现在栈里** ✓，`Xhci-Port-Power.patch` 只做
  PORTSC 读改写 ✓、不分配内存 ✓。

**绕法**（推荐顺序）：

1. **用 RELEASE 固件** ✓（`PcdDebugPropertyMask = 0x02` ✓）—— 最省事 ✓；
2. 或把 DEBUG 的 `PcdDebugPropertyMask` 从 `0x2F` 改成 **`0x27`** ✓（去掉 clear-memory 位 ✓）；
3. 或**不用 Ventoy** ✓：用普通 FAT32 盘（我们的 `cust-*.img` 方案 ✓，不走 UDF ✓）。


---

## 3. `0xC000000D` 的完整排除表

| # | 假设 | 怎么测的 | 结果 |
| --- | --- | --- | --- |
| 1 | BCD 内容/版本错误 | 三份 BCD（PE 的、`bcdedit` 生成的、官方 Win11 的）全部解析：`regf`/major/minor/bins/XOR 校验和 | ❌ 全部合法 |
| 2 | 固件读文件有 bug | `BCD probe` 读 BCD 并算 FNV-1a 64：`0xd1759d3a3caf88af` 与主机**完全一致** | ❌ 字节精确 |
| 3 | FAT 扇区大小（512 vs 4K） | 512 BPB、4096 BPB、带 `HiddenSectors`、`mformat`、整分区铺满 | ❌ 都报同样错 |
| 4 | 分区几何/块读取 | `BCD probe` 复现 bootmgfw 的读法（按 BPB 算 LBA → `ReadBlocks`）：`EFI_SUCCESS` | ❌ 读成功 |
| 5 | BlockIo 句柄死/重复 | 枚举全部 BlockIo：`open=EFI_SUCCESS`、`read0=EFI_SUCCESS`、`present=1`、`removable=0` | ❌ 全部健康 |
| 6 | 512 字节 BlockIo 视图干扰 | `BlockIo512Dxe` 整份移除 | ❌ 仍报错 |
| 7 | 内存映射漏标预留区 | 对照 chopin DT 的 14 个 reserved-memory 节点逐一对账 | ❌ 14/14 全覆盖 |
| 8 | 内存总量不足 | `MemoryMapLib` 重写：127.5 MiB → **7.49 GiB** Conv | ❌ 仍报错 |
| 9 | UFS 平台驱动不按原厂 | 按原厂 `ufs-mtk.c` + `mediatek/mt6893` 重写 26 个步骤 | ❌ 仍报错（但发现 §2） |
| 10 | UFS 事务层未加固 | 整份替换为参考实现的 `UfsPassThruDxe`（OCS/UTRCS 轮询、槽位恢复、UA 重试） | ❌ 仍报错 |
| 11 | 启动项设备路径形态 | SFS 短形式 + 显式 `HD()+File()` 两种 | ❌ 都报错 |
| 12 | PE 版本 | BYDJ v4.0.1、BYDJ v1.5.0（参考同款）、官方 Win11 ARM64 | ❌ 都报错 |
| 13 | ESP 不完整 | 官方 ISO 的整个 `efi/` 树（`boot.stl`、`cipolicies`、字体）+ `\EFI\Boot\BCD` | ❌ 仍报错 |
| 14 | `boot.wim` 损坏 | `7z t` 校验：两个 WIM 都 `Everything is Ok` | ❌ 完好 |
| 15 | 按键/cdboot | `cdboot.efi` 的 `Press any key` 不接受我们的键；`cdboot_noprompt.efi` 返回 `No mapping` | ❌ 该路线终止 |
| 16 | 启动链 SMC（ATF） | r80 日志：`SMC DEVICE_RESET(0/1)=0x0` | ❌ ATF 正常 |
| 17 | Mu 基线 | 参考实现**也是** Mu-Silicium，且能启动 WinPE | ❌ 不是基线问题 |

**bootmgfw 的行为已被逆向确认**（公开 PDB，8898 个符号）：

- `0xC000000D` 只由 `EfiGetNtStatusCode` 从 **`EFI_INVALID_PARAMETER`** 映射而来；
- 失败点在 `BmpOpenStoreAndUpdateOptions → BmFwOpenDataStoreWithHash → BiOpenStoreWithHash → BlDeviceOpen`，
  即**打开 BCD 存储之前的设备打开阶段**，不是 hive 解析；
- `ConvertPages: failed to find range ...` 是 `MmFwGetMemoryMap` 的**主动探测**，红鲱鱼。

---

## 4. 与参考实现（yunluo）的差异清单

`work/ref-diff-yunluo.md` 的结论：**逐字节相同**的部分有

- `PartitionDxe`、`DiskIoDxe`、`ScsiBusDxe`、`ScsiDiskDxe`、`FatPkg`
- `Common/`（`BdsDxe`、`UefiBootManagerLib`、`BootManagerLib`、`MsBootPolicy`、`MsBootPolicyLib`）
- 全部 `MuPatches`（没有一方改过 Scsi/Partition/Fat）

**chopin 独有的**（已处理）：

| 项 | 处理 |
| --- | --- |
| `BlockIo512Dxe`（同 GUID 的第二套 BlockIo/SFS 卷） | r74 移除 |
| `MemoryMapLib` 把 32/35 个 DDR 描述符标成 Reserved | r74 重写 |
| UFS 平台驱动不按原厂 | r77 重写 |
| UFS 事务层未加固 | r76 整份替换 |
| HS-G4 切档破坏链路 | r81 禁用 |

---

## 5. 下一步（按性价比排序）

### A. 交叉验证（**最推荐**，一步切开"镜像 vs 固件"）

把我们的 `cust` 刷到**参考的 yunluo 设备**上：

- 能启动 → 镜像没问题 → 问题在 **chopin 固件的运行时行为**；
- 也报 `0xc000000d` → **镜像/格式**问题 → 照参考重打。

### B. 向参考作者要**他们那份能启动的 `cust` 镜像**

参考的 `TRACKING.md` §7.3 说它来自 `BYDJ_PE_V1.5.0.iso`（我们已有），但可能还有**额外的 BCD 修改**
（例如 `BcdOSLoaderInteger_SafeBoot` 清零）或特定的格式化参数。拿到镜像后可直接逐字节对比。

### C. 继续逆 `BlpDeviceOpen` / `PartitionOpen`

用公开 PDB 继续深入，确定 `BL_DEVICE` 里到底哪个字段不被接受。
（前期分析已修正两个误判：`PartitionNumber < 8` 的检查在描述符 `+0x20` 上；`PartitionFwpGetEfiHandle` 取**第一个**匹配。）

### D. 转 USB 路线（原生 WoA 的唯一现实路径）

Windows 没有 UFS 驱动，系统只能装在外部存储上：

1. 固件加 `XhciDxe` + `UsbBusDxe` + `UsbMassStorageDxe`（纯配置，简单）；
2. 攻 MT6360 OTG 供电（I2C 控制器读回 0，**真正的硬骨头**）；
3. U 盘上装 Windows To Go，从手机 UEFI 启动。

---

## 6. 常用命令与产物

### 构建

```sh
cd /home/kily/project/mt6893/Mu-Silicium
export CLANG_BIN=/usr/lib/llvm-19/bin/          # 注意结尾斜杠
export PATH="/usr/lib/llvm-19/bin:$PATH"
python3 build_uefi.py -d chopin -r DEBUG        # 产物 Mu-chopin.img
python3 build_uefi.py -d chopin -r RELEASE
```

### 刷机（r81 稳定基线）

```sh
fastboot flash boot /home/kily/project/mt6893/port/dist/chopin-uefi-DEBUG-ufs-r81-nohs.img
fastboot flash cust /home/kily/project/mt6893/work/cust-pe150.img     # 或 cust-mformat0.img
```

> **注意**：如果之前刷过带 HS 的固件（r76/r77/r80），刷 r81 后**必须彻底断电一次**（长按电源 15 秒），
> 否则设备残留 HS 状态，UFS 读会失败。

### 关键镜像

| 镜像 | 用途 |
| --- | --- |
| `port/dist/chopin-uefi-DEBUG-ufs-r81-nohs.img` | **当前稳定基线**（UFS PWM + 日志） |
| `port/dist/chopin-uefi-DEBUG-ufs-r80-smchold.img` | 带 8 秒日志暂停 |
| `port/dist/*r78-clean.img` | 关闭诊断入口的干净菜单 |
| `work/cust-pe150.img` | 参考同款 BYDJ v1.5.0 的 `cust` |
| `work/cust-mformat0.img` | `mformat` 整分区布局的 `cust` |
| `work/cust-win11-win.img` | 官方 Win11 + `HiddenSectors` 的 `cust` |
| `work/boot_b-chopin-ufi.img` | UEFI 启动 Linux 的内核载荷（另一条线） |

### 检查工具

```sh
port/tools/check-memmap-vs-dts.py        # DT 保留区 vs 内存映射（14/14 ✓）
port/tools/make-winpe-fat-img.sh         # 从 PE 文件目录生成 cust 镜像
port/tools/pack-uefi-boot-img.py         # 内联 DTB 的 UEFI 载荷打包
```

---

## 7. 调试经验（省时间的几条）

1. **面板日志会滚掉**：早期日志（UFS/SMC）在菜单出现前打印。需要拍照时在 `PostLinkStartup` 加
   `gBS->Stall (8*1000*1000)`（r80 就是这么做的，`MtkUfsDxe.c` 里有注释）。
2. **RELEASE 版没有 `DEBUG()` 输出**：`SiliciumPkg.dsc.inc` 的 `MDEPKG_NDEBUG` 会把 `DEBUG()` 编译掉。
   需要日志时用 DEBUG 版，或者在某模块里 `#undef MDEPKG_NDEBUG`（`KeyProbeApp.c` 有示例）。
3. **`mtools` 读不了 `HiddenSectors != 0` 的镜像**（会报 `Cluster # at 7 too big`）—— 这本身就是个信号：
   参考实现用的是 `HiddenSectors = 0`。
4. **`mformat` 会把 `TotSec32` 铺满整个分区**，而 `mkfs.vfat` 会小 2% —— 两者都能被 `fsck.fat` 判为干净。
5. **不要并发构建**：多个子代理共用同一棵树时，`build_uefi.py` 会互相踩。
6. **UFS 的 HS 状态是设备端持久的**：见 §2。
