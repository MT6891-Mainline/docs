# chopin 主线 GPU（Panfrost / Mali-G77）移植记录

日期：2026-10-21。结论先行：**没有打通**，当前 Debian/GNOME 仍用 Mesa
llvmpipe 软件渲染。这里记录已经确认的边界，避免下次重复踩。

## 硬件与固件

- GPU：Mali-G77（Valhall Gen2），**Panfrost** 路线，不是 MT6895 的
  Mali-G610/Panthor + GPUEB。
- 固件：Android 包（super vendor / boot / vendor_boot）里**没有**
  `mali_csffw.bin`；linux-firmware 的 `arm/mali/arch10.8/mali_csffw.bin`
  可用（282624 字节，sha256
  `a27847ea11f8efb3136340c3ba8aab413ae25145eeb4a7f64ff5edd829a2405b`），
  已放进 `firmware/chopin-wifi/` 并由 rootfs-only init 复制到
  `/lib/firmware/`。

## 已经做过的 DT/驱动

- 在 vendor board tree 上新增：
  - `syscon@10006000` + `mediatek,mt6893-power-controller`（MFG0/1/2..6）；
  - `g3d_config@13fbf000` 增加 `mediatek,mt6893-mfgcfg`；
  - GPU 节点用 `mediatek,mt6893-mali*` 兼容，接 `job/mmu/gpu` 中断、
    `topckgen` 的 MFG_SEL（ID 105）作为 core clock、`g3d_config` 作为
    bus clock。
- Panfrost 侧加了本地 compatible/分支，用于：
  - 无 supply/无 PM domain 的最小 probe；
  - 5 个 MFG domain + 可选 supply；
  - 在 reset 前手动 `pm_runtime_get_sync()` 的尝试。

## 实测结果（重要）

1. **时钟**：`core` 用 `topckgen` MFG_SEL 后 `clock rate = 349999848`
   （350MHz），不再是 0；`bus_clock rate = 0`（BG3D 门控，无 rate）。
2. **无 supply / 无 PD**：reset 超时（`gpu soft reset timed out`），
   但**不会重启**。
3. **5 个 MFG PD、不接 supply**：`pm_runtime_get_sync()` 每个 domain
   返回 **-110（超时）**，PM summary 里 `mfg0..6` 始终 `off-0`。
   说明 SPM 电源域本身没有上电。
4. **加 `domain-supply = buck_vgpu11` / `ldo_vsram_others`**：`modprobe`
   时立刻硬复位，串口最后一行是
   `regulator: Not disabling unused regulators`。
5. **改 `domain-supply = MT6315 7_vbuck1`**：不再复位，但主线
   `mt6315-regulator` 没有绑定 vendor 的 `mt6315_7-regulator` 节点，
   SPM `power-domain@1` 报 `failed to get power supply`，整个
   `10006000.syscon:power-controller` 一直 deferred，panfrost 也 deferred。

## 根因判断

chopin 的 GPU 上电不是主线的 SPM 域 + regulator 模型：

- vendor `gpufreq_v1/mt6893` 的顺序是
  `vsram_gpu → vgpu → mtcmos_mfg_async → mtcmos_mfg → mtcmos_mfg_core*`，
  其中 MTCMOS 是 vendor clock 框架里的门控时钟，不是主线 SPM 电源域；
- 主线没有 MT6893 的 `mtcmos_mfg*` clock provider，也没有让 MT6315
  绑定 vendor SPMI 节点的 glue；
- 所以要么把 vendor 的 MTCMOS/regulator 上电序列做成一个最小驱动，
  要么继续用 llvmpipe。

## 文件/分支

- 内核 WIP：`dev` 分支 commit `466f013`（**不要合并回稳定分支**）。
- 测试镜像（module/initramfs，未进 release）：`out-gpu-*` 目录。
- 稳定启动：`rootfs-only` 分支 + `out-rootfs-only/boot-stockdt.img`。
