# chopin 主线 GPU 交接文档（2026-10-21）

> 新会话直接读这一篇即可。
> **最终状态（2026-10-23）：Mali-G77 已完整跑通内核 + 用户态硬件渲染。**
> 内核 panfrost 用 `g77`（ID 0x9080, r0p1）模型 probe；Mesa 侧新增 G77
> 设备表项，交叉编译出与 Debian 同 SONAME/符号版本的
> `libgallium-25.0.7-2+deb13u1.so` 并替换系统副本。
> `fastfetch` 显示 `Mesa Mali-G77 (Panfrost)`，`eglinfo -B` 的 GBM 平台
> 是 `OpenGL 3.1 / GLES 3.1 (Panfrost)`。
> 下面保留历史过程；最新实现见“G77 完整支持”一节。



## G77 完整支持（2026-10-23）

### 旧方案为什么只能当 G57

- 内核 panfrost 的模型表原来没有 G77；`panfrost_model_cmp()` 会把
  `0x9080` 的低半字节折掉变成 `0x9000`，于是只能拿 G57 的表项顶上。
- 之前加的“Mesa ID quirk”把上报值改成 `0x9001`，但 **Mesa 的 G57 表项是
  `0x9091`/`0x9093`，没有 `0x9001`**，`panfrost_get_model()` 直接返回
  NULL，所以用户态仍然拒绝加载 panfrost。
- 结论：内核要上报真实的 `0x9080`，Mesa 要认识 `0x9080`，两边都得改。

### 内核改动（已整理到 `7.2.8-mt6891-xiaomi-chopin`）

- `drivers/gpu/drm/panfrost/panfrost_gpu.c`
  - 新增 `GPU_MODEL(g77, 0x9080)`；
  - 删除 `0x9000` 的 G57 替身项和 `0x9080 -> 0x9001` 的 quirk。
- `drivers/gpu/drm/panfrost/panfrost_device.h`
  - `panfrost_model_cmp()` 先做精确匹配，再退回低半字节折叠，
    `0x9080` 不会再被折成 `0x9000`。
- `panfrost_features.h` / `panfrost_issues.h`
  - `hw_features_g77` / `hw_issues_g77` 先按 G57 别名（同为 Valhall JM，
    G57 的 errata 本来就以 TTRx 命名），后续有 G77 专属 errata 再细化。
- 期望日志（major/minor 是硬件 revision，以实测为准）：

```
panfrost 13000000.mali: mali-g77 id 0x9080 major 0x0 minor 0x0 status 0x0
```

### Mesa 改动

- `src/panfrost/lib/pan_props.c` 新增：

```c
MODEL(0x9080, 0, "G77", "TTRx", HAS_ANISO, 16384, {}),
```

  - `0x9080` 是 MT6893/MT6891 的 G77 product ID；
  - `"TTRx"` 来自 `src/panfrost/perf/G77.xml` 的 `<metrics id>`；写错会在
    用性能计数器时 unreachable，普通渲染不受影响；
  - TIB 16KB、anisotropic 从 r0 起支持，与 G57 相同。
- Mesa 源码单独成库：**`git/mesa-25.0.7`** —— 上游 25.0.7 一个 import
  提交，`panfrost: add the Mali-G77` 一个改动提交，`chopin/` 下有
  `build.sh`、`README.md` 和导出的补丁。`port/tools/build-mesa-g77.sh`
  现在只是它的 wrapper（兼容旧命令）。
- 交叉编译：`git/mesa-25.0.7/chopin/build.sh [out-dir]`。它在 Debian 13
  宿主机上解包 arm64 的 libdrm/zlib/expat/X11 依赖到私有 sysroot、装交叉
  g++、在 venv 里跑 meson，产物是 `libgallium-<VERSION>.so`。`VERSION`
  固定为目标 rootfs 的 Mesa 包版本（`25.0.7-2+deb13u1`），否则符号版本对不上。
- **必须带 `-Dplatforms=x11`**：Debian 的 `libEGL_mesa` 是用 `RTLD_NOW`
  dlopen libgallium 的，缺 `loader_dri3_*` 会整库加载失败：
  `undefined symbol: loader_dri3_get_buffers, version libgallium-...`，
  结果连 Wayland/GBM 都进不去（实测 `eglinfo -B` 只打印
  “Default display platform: eglInitialize failed” 就退出）。

**关键：Debian 的 EGL 不走 `dri/*_dri.so`。** trixie 的
`libEGL_mesa.so.0` 直接 `DT_NEEDED libgallium-25.0.7-2+deb13u1.so`，
`dri/panfrost_dri.so` 只是 `libdril_dri.so`（给 Xorg 用的空壳）的软链。
所以 G77 驱动必须伪装成 Debian 的 libgallium：
- 文件名、`SONAME`、`--default-symver` 的符号版本节点都必须是
  `libgallium-25.0.7-2+deb13u1.so`（Debian 的做法是把包版本写进 Mesa 的
  `VERSION` 文件；构建脚本已照做）；
- 否则 `dri_get_pipe_screen@libgallium-25.0.7-2+deb13u1.so` 解析不到，
  EGL 直接 `failed to create dri2 screen`，回落到 llvmpipe。

### 产物与测试

- `out-g77/libgallium-25.0.7-2+deb13u1.so`：G77 版 Mesa（已 strip，约 17MB）。
- `out-g77/boot-stockdt.img`：新内核 + **厂商 DTB**（`port/out/stock-base.dtb`）
  + initramfs。initramfs 把 `/lib/firmware/*.so` 复制到目标 rootfs 的
  `/root/`，**不覆盖系统文件**。
- 注意必须用 **stockdt**：LK 要在 boot.img 的厂商 DTB 上套 dtbo 覆盖层，
  用 `owndtb` 会 `ufdt_apply_overlay() failed` 直接不引导。我们自己的板级树
  由内核内嵌（`arch/arm64/kernel/chopin-board.dtb`，`setup_arch()` 里替换
  LK 传进来的 FDT），GPU 节点就在那棵树里。
- 刷入（变砖恢复镜像仍是 `out-rootfs-only/boot-stockdt.img`）：

```sh
fastboot flash boot_a /home/kily/project/mt6893/out-g77/boot-stockdt.img
```

- 串口验证：

```sh
dmesg | grep -i panfrost
# panfrost 13000000.mali: mali-g77 id 0x9080 major 0x0 minor 0x1 status 0x1

ls -l /root/libgallium-25.0.7-2+deb13u1.so
LD_LIBRARY_PATH=/root eglinfo -B 2>&1 | head -40
# 期望 renderer 变成 Mali-G77 (Panfrost)，不再是 llvmpipe
```

- 确认无误后再系统级替换（先备份 Debian 的原文件）：

```sh
cp -a /usr/lib/aarch64-linux-gnu/libgallium-25.0.7-2+deb13u1.so \
      /root/libgallium-25.0.7-2+deb13u1.so.debian
cp /root/libgallium-25.0.7-2+deb13u1.so /usr/lib/aarch64-linux-gnu/
systemctl restart gdm        # 或直接 reboot
```

  回滚：把 `.debian` 那个文件拷回去再重启 gdm。

  回滚：把 `panfrost_dri.so.debian` 拷回去。

### 验证结果（2026-10-23，实机）

```
[    1.526395] panfrost 13000000.mali: mali-g77 id 0x9080 major 0x0 minor 0x1 status 0x1
[    1.528452] panfrost 13000000.mali: features: 00000000,000019f7, issues: 00000001,80000400
[    1.530000] panfrost 13000000.mali: shader_present=0x660067 l2_present=0x1
```

```
GBM platform:
OpenGL core profile renderer: Mali-G77 (Panfrost)
OpenGL core profile version: 3.1 (Core Profile) Mesa 25.0.7-2+deb13u1
OpenGL ES profile version: OpenGL ES 3.1 Mesa 25.0.7-2+deb13u1
```

`systemctl restart gdm` 后 GDM 正常，`fastfetch` 的 GPU 行不再是 llvmpipe。
替换前建议先 `cp -a` 备份，回退就是拷回 `.debian` 再重启 gdm；
`dpkg -V libgl1-mesa-dri` 可以确认系统文件是否已被改过。

**首 job 挂起（已解决，2026-10-23）**：之前每次开机 ~17s 都会出现一次

```
gpu sched timeout, js=1, config=0x7b00, status=0x8,
head==tail, js_state=0x0, int_rawstat=0x0, fault=0x0
```

也就是**首个 userspace job 被 JM 接受却从未执行**（槽位空闲、无中断、无 fault），
只有超时恢复里的 `panfrost_device_reset()` 才能让 GPU 恢复，之后一切正常。
把 reset 放到 `panfrost_gpu_init()` 里（power_on 之后、JM/MMU 初始化之前）没用；
放到 `panfrost_device_init()` 末尾、`panfrost_mmu_init()`/`panfrost_jm_init()`
之后才复现出“恢复后的状态”，首个提交 ~2ms 完成，`gpu sched timeout` 计数为 0。
修复提交：`8c66104`（主分支；旧测试分支上是 `4e95b2d` / `9a36347`）。

### 2026-10-23 追加：G77 专属配置（MFG AXI/门控 + errata 对齐）

内核提交 `8c66104`（主分支；旧测试分支上对应 `22d764e` / `784a2f7`）：

1. **`chopin_mfg_axi_config()`**：在 `panfrost_clk_init()` 之后（`bus` 即
   `g3d_config` 门控时钟已经打开）按 vendor
   `mt_gpufreq_external_cg_control()` 写 MFG 寄存器：
   - `0x13fbf020` / `0x02c`：打开 MEM0/1、MEM0_1/MEM1_1 端口的时钟门控；
   - `0x13fbf0b0`：清 `GPU_CLK_FREE_RUN`、`SOC_IN/OUT_AXI_FREE_RUN`、
     `PWR_CG_FREE_RUN`；
   - `0x13fbf0b4`：打开 Q-channel；
   - `0x13fbf024`：打开 `FAXI_CK_SOC_IN_EN`（SOC-in AXI 时钟）。
   目的：和原厂上电序列一致；空闲时能真正关门控（省电/发热），
   并避免 free-run 时钟下的总线时序差异。
2. **errata 对齐**：`hw_features_g77` / `hw_issues_g77` 不再别名 G57，而是按
   vendor DDK（`n10p/vendor/.../mali_valhall/mali-r32p1` 的
   `base_hw_features_tTRx`、`base_hw_issues_tTRx_r0p0/r0p1`）显式列出，
   模型表补了 r0p0/r0p1 两个 rev 项（MT6893 报 r0p1）。
   vendor 表里 11 条 errata 中 mainline 已实现的仍是 3 条
   （9435 / 2968+3162 / 3076）；其余注释里标明是 Mesa 编译器侧
   （1337/3414/3083/3470/3464/TSIX_2033）或 vendor 用户态发起的
   dummy-job（3485）。
3. **超时诊断**：`panfrost_job_timedout()` 额外打印 `js_state`、
   `int_rawstat`、`GPU_FAULT_STATUS/ADDRESS`、`JS_COMMAND(_NEXT)` 和 job 的
   `jc`，下次真出问题能直接区分“槽位卡住 / 真 fault / 命令没被消费”。
4. **首 job 修复**：`panfrost_device_init()` 末尾补一次
   `panfrost_device_reset(pfdev, true)`（JM/MMU 初始化之后），
   消除开机后首个 job 永不完成的问题；详见上一节。

验证：刷入后 `dmesg | grep -i "MFG AXI"` 应出现
`chopin MFG AXI/CG configured`；对比待机功耗/温度与
`dmesg | grep -c "gpu sched timeout"`。

### 注意

- 新内核上报 `0x9080` 后，Debian 自带 Mesa（没有 G77 表项）会继续回落
  llvmpipe；必须配合 `out-g77/panfrost_dri.so` 才是硬件渲染。
- 如果 Mesa 在 G77 上触发 GPU fault，先看 `dmesg` 里的 `panfrost` 报错，
  再决定 `hw_features_g77`/`hw_issues_g77` 是否需要和 G57 分家。
- 想快速验证“Mesa 的 Valhall 路径在这块板子上能不能跑”，也可以临时把
  内核 quirk 改成上报 `0x9093`（Mesa 的 MT8188 G57 ID），用 Debian 原装
  Mesa 就能加载；但那是把 G77 当 G57 用，只适合排查。

## 分支整理（2026-10-23）

GPU 这一串已经整理进主分支 `7.2.8-mt6891-xiaomi-chopin`，从 `fabd97a` 之后是：

```
8c66104 drm: panfrost: support the MT6893 Mali-G77
e4b944b arm64: dts: chopin: wire up the Mali-G77 GPU
21d635c clk: mediatek: add a minimal MT6893 MFG MTCMOS clock provider
75b277a drm: panel: k10a: disable the broken 90 Hz mode
d06a58b input: nvt36672c: fix chopin touch Y orientation
fabd97a arm64: chopin: raise CMA to 256MB for mutter/GNOME
```

整理时做的事：

- 删掉 `panfrost_pm_domain_init()` 里的调试打印和 `get_sync` 探针，恢复上游的
  `DL_FLAG_RPM_ACTIVE`（本板 0 个 PM domain，这段本来就是死路径）；
- 删掉 `chopin_mtcmos_on()` / `chopin_gpu_rails_on()` 的逐条 `dev_info`；
- 只保留 `mediatek,mt6893-mali` 一个 compatible，删掉调试期的
  `-nopd` / `-pd` / `-pd1` / `-single` 变体；
- 删掉 DT 里没用上的 `gpu_scpsys` power-controller 节点；
- 旧工作分支 `gpu-g77`（`92258ca` + GPU）和 `gpu-mtcmos`（含 PMIC WIP）
  保留不动，PMIC 那两个 WIP 提交没有进主分支。

整理后重打的镜像：`out-g77/boot-stockdt.img`，内核 `7.2.8-g8c6610447790`，
md5 `23bb1967800d577aaabc8d5b1b84ce9a`。

## 最终实现（2026-10-23）

> 这一节记录让内核 probe 通过的过程；其中“用户态仍是 llvmpipe”和
> `GPU_MODEL(g57, 0x9000)` 的结论已被上一节 G77 支持取代。

### 结果

- 内核 Panfrost 完整 probe：
  - `shader_present=0x660067`、`l2_present=0x1`；
  - `[drm] Initialized panfrost 1.6.0 for 13000000.mali`；
  - `/dev/dri/renderD128` 存在。
- 用户态仍是 llvmpipe：Mesa 官方 Panfrost 支持表没有 G77（只有
  G57/G68 等），`panfrost_dri.so` 没有 G77 后端。

### 最终方案

不使用主线的 SPM `power-domains`（会 `-110`），也不使用 MT6315/SPMI
（会 `SWINF_WFVLDCLR` 超时）。改用：

1. **GPU rail 手动开**：panfrost probe 时按 vendor 顺序
   `vsram_others -> vgpu11`，不通过 SPM `domain-supply`；
2. **MTCMOS clock provider**：
   `drivers/clk/mediatek/clk-mt6893-pg.c`，注册
   `mtcmos_mfg_async`、`mtcmos_mfg`、`mtcmos_mfg_core0/1_2/3_4/5_6/7_8`；
   enable 时写 `POWERON_CONFIG_EN`、`PWR_ON/PWR_ON_2ND`，再清
   `CLK_DIS/ISO/SRAM_PDN`、置 `PWR_RST_B`；
3. **GPU 节点**：`mediatek,mt6893-mali-nopd`，clocks 引用上述 MTCMOS；
4. panfrost 在 reset 前按顺序 `clk_prepare_enable()`；
5. GPU ID 0x9080 被 `panfrost_model_cmp()` mask 成 0x9000，模型表加
   `GPU_MODEL(g57, 0x9000, ...)`；另加 0x9080 -> 0x9001 的 Mesa ID quirk。

关键修复点：iomem 必须在 runtime resume 之前映射，否则
`panfrost_gpu_soft_reset` 空指针；MFG1 的 `PWR_ON` 写完后不能再用
主线 SPM 轮询状态，否则会挂总线。

### 提交（内核 `gpu-mtcmos`）

- `11c3c19 clk: mediatek: add minimal MT6893 MFG MTCMOS provider`
- `2f3a15d clk: mediatek: release MT6893 MFG islands after power-on`
- `1af0b9a drm: panfrost: bring up the MT6893 Mali-G77`
- `9b9f7e7 input: nvt36672c: fix chopin touch Y orientation`（触控）

### 镜像

- `out-safe/boot-stockdt.img`
  - md5 `5cb59a4d80cedc05334924aef3326ad4`
  - GPU 内置、触控半屏修复、90Hz 移除、suspend 自动重试
- 验证：
  ```sh
  dmesg | grep -E "panfrost|mali-g|shader_present"
  ls -l /dev/dri
  ```

### 不要重复的坑

- 不要用 `g3d_config` 当 core clock（rate 0）；
- 不要用 SPM `domain-supply = buck_vgpu11`（硬复位）；
- 不要用 MT6315 7_vbuck1 走主线 SPMI（`SWINF_WFVLDCLR` 超时）；
- 不要把 `mtcmos_mfg*` 接到主线 SPM 再轮询 PWR_STATUS（会挂）；
- 不要指望 Mesa 用 G77：需要单独移植 Mesa 后端。

## 1. 设备与启动链

- 设备：Redmi Note 10 Pro 5G / POCO X3 GT（chopin，MT6893/MT6891Z，8GB）。
- 启动链：preloader → LK → GenieZone → Linux EL1（无 U-Boot/UEFI）。
- 分区（GPT）：`userdata=/dev/sdc75`、`nvdata=/dev/sdc6`、
  `super=/dev/sdc61`、`misc=/dev/sdc1`、`boot_a=/dev/sdc32`、
  `vendor_boot_a=/dev/sdc33`。
- 串口：USB ACM `/dev/ttyGS0`，主机 minicom `/dev/ttyACM0` 115200。
- 稳定系统：Debian 13 + GNOME（llvmpipe），`rootfs-only` initramfs 启动
  userdata；内核稳定分支 `7.2.8-mt6891-xiaomi-chopin` HEAD `fabd97a`。

## 2. 内核/仓库状态

- 内核源：`git/chopin-linux-7.2.8`。
- **GPU WIP 分支：`dev`，commit `466f013`**
  （`wip: chopin GPU bring-up (panfrost + SPM/MFG nodes)`）。
  **不要合并到稳定分支**；稳定分支仍是 `fabd97a`。
- WIP 只改了 3 个文件：
  1. `arch/arm64/boot/dts/mediatek/mt6893-xiaomi-chopin-gpu.dtsi`
     - 新增 `syscon@10006000`（`mediatek,mt6893-scpsys`）+ `power-controller`
       （`mediatek,mt6893-power-controller`），MFG0/1/2..6；
     - `g3d_config@13fbf000` 增加 `mediatek,mt6893-mfgcfg`；
     - GPU 节点：
       ```
       compatible = "mediatek,mt6893-mali*"（本地调试用多个变体）
       reg = <0 0x13000000 0 0x4000>;
       clocks = <&{/topckgen@10000000} 105>,   /* core: MFG_SEL */
                <&{/g3d_config@13fbf000} 0>;    /* bus: BG3D gate */
       clock-names = "core", "bus";
       interrupts = <0x16c>,<0x16b>,<0x16a>;   /* job/mmu/gpu */
       power-domains = <&gpu_spm 3..7>;        /* 当前 WIP 状态 */
       power-domain-names = "core0".."core4";
       status = "okay";
       ```
     - 注意：vendor 原 `mali@13000000` 是 `mediatek,mali`/`arm,mali-valhall`，
       没有 clocks/PD，panfrost 不认。
  2. `drivers/gpu/drm/panfrost/panfrost_drv.c`
     - 增加本地 compatible/data：
       `mediatek,mt6893-mali-nopd`（0 supply/0 PD）、
       `mediatek,mt6893-mali-pd`（0 supply/5 PD）、
       `mediatek,mt6893-mali-pd1`（1 supply/5 PD，调试用）。
  3. `drivers/gpu/drm/panfrost/panfrost_device.c`
     - `device_link_add` 去掉了 `DL_FLAG_RPM_ACTIVE`
       （带它时 `pm_runtime_get_sync(supplier)` 返回 <0，link 失败）；
     - `panfrost_device_init()` 在 `panfrost_pm_domain_init()` 之后
       `pm_runtime_enable()+pm_runtime_get_sync()`，并逐个
       `pm_runtime_get_sync(pm_domain_devs[i])` 打印结果。
     - 这些是调试态，不是可提交的最终方案。
- 内核配置：`.config` 里 `CONFIG_DRM_PANFROST=m`（模块方式测试，避免开机
  被 GPU probe 拖死）；`chopin_defconfig` 未改，仍是 `m`。

## 3. 固件（已解决）

- Android 包（super 的 vendor 2.2GB、boot.img、vendor_boot_a 64MB）里
  **没有** `mali_csffw.bin`，只有 SELinux file_contexts 引用。
- 使用 linux-firmware 的 Valhall Gen2 固件：
  ```
  arm/mali/arch10.8/mali_csffw.bin
  282624 bytes
  sha256 a27847ea11f8efb3136340c3ba8aab413ae25145eeb4a7f64ff5edd829a2405b
  ```
  已放在 `port/firmware/chopin-wifi/mali_csffw.bin`；`rootfs-only` init 会把它
  复制到目标 rootfs 的 `/lib/firmware/mali_csffw.bin`。
- 设备上校验：
  `sha256sum /lib/firmware/mali_csffw.bin`。

## 4. 实测结果（按时间顺序）

| 配置 | 结果 |
| --- | --- |
| vendor `mali@13000000`（无 clocks/PD） | panfrost 不绑定/无日志 |
| 本地 no-PD、core 时钟用 `g3d_config` | `clock rate = 0`，soft/hard reset 超时 |
| core 时钟改 `topckgen MFG_SEL` (105) | `clock rate = 349999848`（350MHz） |
| 5 个 MFG PD、无 supply | `pm_runtime_get_sync(coreN)` 全部 **-110**；`mfg0..6` 保持 `off-0`；reset 超时 |
| 加 `domain-supply = buck_vgpu11` + `ldo_vsram_others` | `modprobe` **立刻硬复位**，最后一行 `regulator: Not disabling unused regulators` |
| 改 `domain-supply = MT6315 7_vbuck1`（MFG0） | 不重启，但主线 `mt6315-regulator` 未绑定 vendor 节点 → SPM `power-domain@1: failed to get power supply`，`10006000.syscon:power-controller` 一直 deferred，panfrost deferred |
| 只加 `mali-supply = buck_vgpu11`（pd1） | `modprobe` 立刻重启 |

关键日志：

```
panfrost 13000000.mali: clock rate = 349999848
panfrost 13000000.mali: bus_clock rate = 0
panfrost 13000000.mali: gpu soft reset timed out, attempting hard reset
panfrost 13000000.mali: gpu hard reset timed out
...
pm domain core0 get_sync=-110
...
regulator: Not disabling unused regulators     # 然后复位
10006000.syscon:power-controller: deferred probe pending:
  /syscon@10006000/power-controller/power-domain@1: failed to get power supply
platform 13000000.mali: deferred probe pending: supplier ... not ready
```

`/sys/kernel/debug/pm_genpd/pm_genpd_summary` 里 `mfg0..6` 始终 `off-0`。

## 5. 根因判断

chopin 的 GPU 上电不是主线 SPM 电源域 + regulator 模型：

- vendor `drivers/misc/mediatek/base/power/gpufreq_v1/mt6893/mtk_gpufreq_plat.c`
  的顺序是：
  1. `mt_gpufreq_buck_control(POWER_ON)`：
     `regulator_enable(vsram_gpu)` **先**，再 `regulator_enable(vgpu)`；
  2. `mt_gpufreq_mtcmos_control(POWER_ON)`：
     `mtcmos_mfg_async` → `mtcmos_mfg` → `mtcmos_mfg_core0/1_2/3_4/5_6/7_8`
     （vendor clock 框架里的**门控时钟**）；
  3. `mt_gpufreq_cg_control` 里写 MFG 寄存器：
     `0x13fbf024`/`0x13fbf0b0` 等（`MFG_ASYNC_CON_1`/`MFG_GLOBAL_CON`）。
- 主线没有 **MT6893 的 `mtcmos_mfg*` clock provider**；`mtk-pm-domains`
  的 MT6893 数据直接操作 SPM，但在这块板子上电源域上电超时；
- 主线 `mt6315-regulator` 的 compatible 是 `mediatek,mt6315-regulator`，
  vendor 节点是 `mediatek,mt6315_7-regulator`，没有 glue，所以
  `7_vbuck1` 拿不到；
- `buck_vgpu11` 很可能不是 GPU 实际供电轨（vendor 用的是 MT6315
  `7_vbuck1`），强行 enable 会硬复位。

## 6. 下一步（按优先级）

1. **让 MT6315 真正绑定** vendor `mt6315_7_regulator`：
   - 在节点上同时给 `mediatek,mt6315-regulator`（已试，仍 deferred）；
   - 检查父 `spmi@10027000` 的 compatible 是否被主线 SPMI 驱动绑定；
   - 对照 `drivers/regulator/mt6315-regulator.c` 的 probe 要求
     （`regulator-id`、`#regulator-cells`、`mediatek,regulator-name` 等）；
   - 成功标志：`/sys/class/regulator/` 出现 `7_vbuck1`，SPM 不再 deferred。
2. **移植 MTCMOS 门控**：
   - vendor 的 `mtcmos_mfg*` 由 vendor clock 驱动实现，主线没有；
   - 参考 vendor `clk-mt6885-pg.c` / `clk-mt6873-pg.c`（同类 MTCMOS 门控）
     和 MT6893 SPM 头文件，给 MT6893 写一个最小 clock provider，或者在
     power driver 里直接写 SPM MTCMOS 寄存器；
   - 顺序必须是 `mtcmos_mfg_async → mtcmos_mfg → cores`，且先
     `vsram_gpu` 后 `vgpu`。
3. 在 GPU probe 前执行 vendor 的 `MFG_ASYNC_CON_1`/`MFG_GLOBAL_CON` 写序列
   （`0x13fbf024`、`0x13fbf0b0` 等），否则 reset 可能仍超时。
4. 用**模块方式**测试：`panfrost=m`，从 initramfs `/lib/modules` 复制到
   rootfs，手动 `modprobe panfrost`；失败重启即可，不影响开机。

## 7. 测试镜像与构建

GPU WIP 内核构建：

```bash
cd /home/kily/project/mt6893
git -C git/chopin-linux-7.2.8 checkout dev   # 466f013
sed -i 's/^CONFIG_DRM_PANFROST=m/CONFIG_DRM_PANFROST=m/' build/chopin-net/.config
make -C git/chopin-linux-7.2.8 O=$PWD/build/chopin-net \
     ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j16 Image dtbs modules
make -C git/chopin-linux-7.2.8 O=$PWD/build/chopin-net \
     ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- \
     INSTALL_MOD_PATH=$PWD/work/gpu-modules modules_install
```

只打包 panfrost 依赖的最小模块树（`panfrost.ko`、
`drm_shmem_helper.ko`、`gpu-sched.ko` + `depmod`），然后用
`port/tools/build-images.sh` + `INIT_SRC=port/initramfs/init.c` +
`MODULES_DIR=...` 生成 initramfs 测试镜像。

已生成过的镜像（本地 `out-gpu-*`，不在 git 里，可能被清理）：

| 镜像 | md5 | 行为 |
| --- | --- | --- |
| `out-gpu-nopd` | `234f10706ed63922486d87b485cf75c7` | 无 PD/supply，350MHz，reset 超时 |
| `out-gpu-pd` | `602094be321099c0eef7c58c8579ae84` | 5 PD 无 supply，get_sync -110 |
| `out-gpu-rpm5` | `ba00474c481caf3ed653d4cd23811aa8` | 逐个 genpd get_sync=-110 |
| `out-gpu-reg` | `f2bde5c15745ee7478445af832ce1077` | buck_vgpu11+vsram，modprobe 硬复位 |
| `out-gpu-mt6315` | `534ce7c7dee4f9d381abd2fead925581` | MT6315 supply，SPM deferred |

稳定恢复镜像：`out-rootfs-only/boot-stockdt.img`
（md5 `7db5aed6fed6adda3fbf58e042da1b63`）。启动失败时：

```bash
fastboot flash boot_a /home/kily/project/mt6893/out-rootfs-only/boot-stockdt.img
```

## 8. 复现命令/检查点

```sh
# 串口 shell（initramfs module 测试镜像）
uname -r
ls /lib/modules/$(uname -r)/kernel/drivers/gpu/drm/panfrost/
modprobe panfrost
dmesg | grep -iE "panfrost|pm domain|power-controller|regulator|reset"
cat /sys/kernel/debug/pm_genpd/pm_genpd_summary
cat /sys/kernel/debug/clk/clk_summary | grep -iE "mfg|gpu"
ls -l /dev/dri
sha256sum /lib/firmware/mali_csffw.bin
```

验收标准：`/dev/dri/renderD128` 出现、`eglinfo -B` 的 GBM platform 能用
硬件 renderer（而不是 llvmpipe）、`glxinfo`/`gnome-shell` 不再软件渲染。

## 9. 不要重复踩的坑

- 不要把 `mali@13000000` 只改成 `mediatek,mali` 就指望 panfrost；
- 不要用 `g3d_config` 当 `core` 时钟（rate 0，reset 超时）；
- 不要在开机阶段 enable `buck_vgpu11`/`ldo_vsram_others`（会硬复位）；
- 不要用 `DL_FLAG_RPM_ACTIVE` 建 device link（supplier get_sync 失败）；
- 不要合并 `dev` 的 GPU WIP 到稳定分支；
- Android 包里确实没有 `mali_csffw.bin`，用 linux-firmware arch10.8。

## 2026-10-22 更新：内核 GPU 已打通，Mesa 不支持 G77

最终走通的是 **MTCMOS clock provider + nopd + 手动 GPU rail**，不再使用主线的
SPM `power-domains`：

- `drivers/clk/mediatek/clk-mt6893-pg.c` 注册 7 个
  `mtcmos_mfg*` clock，enable 时写
  `POWERON_CONFIG_EN`、`PWR_ON/PWR_ON_2ND`，再清
  `CLK_DIS/ISO/SRAM_PDN`、置 `PWR_RST_B`；
- GPU 节点改成 `mediatek,mt6893-mali-nopd`，clocks 引用这些 MTCMOS；
- panfrost 在 reset 前按 vendor 顺序 `clk_prepare_enable()`；
- 0x9080（G77）被 `panfrost_model_cmp()` mask 成 0x9000，模型表加
  `GPU_MODEL(g57, 0x9000, ...)` 后 probe 通过；
- 实测：`shader_present=0x660067`、`renderD128` 出现、
  `[drm] Initialized panfrost 1.6.0`。

**用户态仍是 llvmpipe**：Mesa Panfrost 官方支持表没有 G77（只有
G57/G68 等），`panfrost_dri.so` 没有 G77 后端，强制
`MESA_LOADER_DRIVER_OVERRIDE=panfrost` 也会失败。要继续只有重编 Mesa
加 G77 支持，或接受软件渲染。

触控：NVT36672c 驱动不解析 `touchscreen-inverted-y`，已加
`novatek,invert-y`，并且这块面板**右半屏 Y 方向与左半屏相反**，所以只在
`input_x >= abs_x_max/2` 时翻转。

内核分支：`gpu-mtcmos`，提交
`2f3a15d` / `1af0b9a` / `9b9f7e7`（provider 初始提交为 `11c3c19`）。
