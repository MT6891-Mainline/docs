# 转 MT6895 路线（厂商 `mediatek_v2` 显示栈）—— 状态与评估

## 已完成

1. **厂商显示树已 staged**：`linux-7.2.8/drivers/gpu/drm/mediatek/mediatek_v2/`（116 项，3.5 MB，
   含 `mtk_disp_dsc.c` 21 KB、`mtk_cmdq_dummy.c`、`mi_disp/`、`ca/`、`mtk_corner_pattern/`）。
2. **自动化脚本**：`port/tools/import-mediatek-v2.sh`（`inventory` / `copy` / `wire` 三阶段，可重复执行）。
3. **缺口清点**：`port/audit/v2-import-inventory.txt`、`port/audit/v2-external-needs.txt`。

## 清点结论（决定工作量的关键数字）

| 依赖 | 出现次数 | 说明 |
| --- | --- | --- |
| **ION**（`ion_`）| **580** | ★ 最大的一块：厂商显示栈的缓冲区管理基于 ION，mainline 用 dma-buf/dma-heap ⇒ 需要一个 ION 兼容层或移植厂商 ION |
| `mtk_fence_` | 153 | 厂商 fence 框架（与 ION 配套）|
| `mi_disp` | 374 | 小米显示扩展（亮度/色彩/DC 调光等）——可先裁剪 |
| `mtk_panel_ext` | 72 | 厂商面板 API（DSC 参数、刷新率切换、ESD 都在这里）|
| `mtk_sync_` | 55 | 厂商 sync 框架（fence 的底层）|
| `dma_buf_` | 7 | 与 mainline 的接口点很少，这是好消息 |
| `mmdvfs` | 10 | DVFS 钩子（可先置空）|
| `mtk_cmdq_`/`m4u_`/`mmqos`/`scpsys`/`mtk_disp_notify` | 少量或 0 | 比预想轻；`mtk_cmdq_dummy.c` 可直接替代 CMDQ |

## 建议的最小可跑路径（按此顺序，每步可编译验证）

1. **只编 v2 的显示核心**（`mtk_drm_drv/crtc/ddp/dsi/disp_dsc/disp_ovl/disp_rdma/...`），
   把 `mi_disp/`、`ca/`、DP/HDMI 先排除（Kconfig/Makefile 只列出必需文件）；
2. 补桩：`mtk_cmdq_dummy.c` ✓ 自带、`mmdvfs` 空实现、`mtk_disp_notify` 空实现；
3. **缓冲区层**：实现 `ion_*` 到 dma-buf/dma-heap 的薄兼容层（580 处调用但模式统一：
   分配/导入/映射/取物理地址 ⇒ 可集中在一层里适配）；
4. fence/sync：厂商 `mtk_fence`/`mtk_sync` 一并编译（若依赖 ION 的部分能一起被上面的兼容层覆盖）；
5. DT：按 MT6895-Mainline 的 compatible 清单对齐（`mediatek,mt6893-disp` → `mt6885_mmsys_driver_data`、
   `mediatek,mt6893-dsi`、`mediatek,mt6893-mipi-tx`、`mediatek,mt6893-disp-dsc`）；
6. 切换：`CONFIG_DRM_MEDIATEK_V2=y`（同时关掉上游 `mediatek-drm` 的对象列表，照 MT6895-Mainline 的
   Makefile 门控），面板用厂商 k10a 等价物（DSC 参数直接来自厂商表）；
7. 逐项与 `port/DSC-REFERENCE.md` 的黄金值对账。

## 注意

- 切到 v2 后，现有 adopt 模式（LK 帧镜像）与 mainline DSI 命令通道都会退居二线；
  建议**保留**当前主线版本（`chopin-ringfix87-*` 镜像与 git 提交）作为可用回退。
- `linux-7.2.8/drivers/gpu/drm/mediatek/mediatek_v2/` 目前**未接入构建**（Makefile 门控未启用），
  因此不影响现在能正常显示的镜像。

## 首次编译结果（自动化跑通：厂商树已进构建）

`port/tools/import-mediatek-v2.sh` 的 `copy` + `wire` 之后首次编译，两个文件即失败：

```
mtk_drm_crtc.c:15:10: fatal error: drm/drmP.h: No such file or directory
mtk_drm_drv.c:15:10:  fatal error: drm/drmP.h: No such file or directory
```

**结论（这决定了工程量）**：这份厂商显示栈是 **Android 4.19 时代**的代码（`drmP.h` 自 5.5 起已被移除，
后面还会遇到 `drm_gem_cma_helper`→`drm_gem_dma_helper`、`drm_display_mode`、
`drm_connector` 等一整套 API 迁移）。也就是说 v2 路线的真正成本是**把 ~111 个厂商文件从 4.19 DRM API
现代化到 7.2**，而不是显示逻辑本身。

**关键自动化机会**：MT6895-Mainline 已经对**同一份厂商代码家族**做完这件事（分支
`7.2-mt6895-xiaomi-xaga`，内核同为大版本 7.2）。因此下一步应当是：

1. 取他们的 `drivers/gpu/drm/mediatek/mediatek_v2/`（以及 `mi_disp/`、`Kconfig`、`Makefile`）；
2. 与我们的厂商树**逐文件 diff**：他们的文件 = "厂商代码 + 现代化补丁"；
3. **移植他们的补丁**（而不是自己重写），差异集中在头部 include、GEM/framebuffer/connector API、
   fence/ION 适配层；
4. 我们这边与 MT6895 不同的只有 SoC 数据（他们用 `mt6895_mmsys_driver_data`，我们要
   `mt6885_mmsys_driver_data` —— 他们仓库里已有 `"mediatek,mt6893-disp"` → `mt6885_mmsys_driver_data`
   的映射，可直接照抄）。

日志与证据：`port/audit/v2-build.log`（本次编译全量）、`port/audit/v2-external-needs.txt`（依赖统计）、
`port/audit/v2-import-inventory.txt`（厂商树清点）。

## 现代化循环（自动，已跑 3 轮）

工具：`port/tools/modernize-v2.py`（机械改名/头文件拆分，幂等）+ 每轮 `make Image` 收集错误到
`port/audit/v2-build.log`。

| 轮次 | 错误数 | 内容 |
| --- | --- | --- |
| 0 | 2 | `drm/drmP.h`（5.5 移除）—— 44 个文件受影响 |
| 1 | 13 | `mmdvfs_pmqos.h`、`drm_internal.h`、`drm/mediatek_drm.h`、`linux/reservation.h`、`mmprofile.h` |
| 2 | 13 | 上述四个厂商头文件**都在厂商树里找到并 staged** 到 `mediatek_v2/vendor-include/`；`reservation_object_*` → `dma_resv_*`；`drm_internal.h` 因与会遮蔽 mainline 同名头而**不 stage** |
| 3 | 12 | `mi_disp_notifier.h`/`mmprofile_static_event.h` 已 staged；剩余：`drm_internal.h`（需 API 适配，不能靠拷头文件）、`mmprofile_function.h`、`uapi/drm/mi_disp.h` |

**模式已经很清楚**：每轮暴露的是"再几个厂商头文件"或"一个 4.19→7.2 的 API 改名"，
全部是机械工作 ⇒ 继续跑循环即可收敛；只有三类需要人工判断：
`drm_internal.h` 相关的内部 API、fence/ION 适配层、以及 `mi_disp` 扩展（可先裁剪）。

## 第 4~5 轮：头文件层已通过，进入 API 层

- `v2-stage-headers.sh` 自动把编译日志里缺失的厂商头文件从厂商树 stage 进
  `mediatek_v2/vendor-include/`（`drm_internal.h` 例外：会遮蔽 mainline 同名头，改为删掉引用）；
- 现在编译已越过全部头文件问题，进入 **331 个 API 级错误**（`cmdq_pkt_write/wfe/destroy` 参数变化、
  缺 `linux/vmalloc.h`、`mtk_ddic_dsi_msg` 等）⇒ 这正是"4.19 → 7.2 API 迁移"，机械且可收敛；
- 优先级最高的一类仍是 **fence/ION**（`ion_` 580 次），建议直接 diff MT6895-Mainline 的
  `mediatek_v2/` 移植其适配层。

## 关键发现：错误高度集中（决定"最快路径"）

按出错文件统计（`port/audit/v2-build.log`）：

| 文件 | 错误数 | 说明 |
| --- | --- | --- |
| `mtk_drm_ddp.c` | 186 | 厂商的巨型 DDP/路由表文件 |
| `mtk_drm_ddp_comp.c` | 98 | 厂商组件抽象层 |
| `mtk_debug.c` | 87 | 调试/寄存器 dump（**可以暂时不编**）|
| `mtk_drm_fbdev.c` | 40 | 厂商 fbdev（**可以暂时不编**，用 mainline 的 drm_fbdev 或先关掉）|
| `mtk_drm_debugfs.c` | 20 | 调试（**可以暂时不编**）|
| `mtk_drm_fb.c` / `mtk_drm_mmp.c` | 12 / 9 | 辅助 |
| `mtk_drm_drv.c` / `crtc.c` / `gem.c` / `plane.c` | **3 / 2 / 2 / 3** | ★ **核心几乎编译通过** ✓ |

**结论 / 最快路径**：
1. 先把 87+40+20+9 的**调试与 fbdev 辅助文件**从最小 Makefile 里摘掉（可后补），错误立刻从 ~330 降到 ~200；
2. 真正需要动脑的只有 **`mtk_drm_ddp.c` + `mtk_drm_ddp_comp.c`** 两个文件（284 个错误）；
3. 这两个文件在 MT6895-Mainline 的 `mediatek_v2/` 里**已被现代化过** ⇒ **只 diff 这两个文件**即可拿到绝大部分补丁（而不是 diff 全部 111 个文件）——这是把"剩多少工程量"压到最小的关键操作。

对应地，`port/tools/import-mediatek-v2.sh` 的 `copy` 阶段产出的目录结构与他们一致，
所以 `diff -u` 可以直接按文件名对齐。

## 第 6 轮：调试/fbdev 辅助移出最小构建后 —— **只剩 2 个文件要现代化**

```
mtk_drm_ddp.c       186 处错误
mtk_drm_ddp_comp.c   98 处错误
（其余：mtk_drm_fb.c 12、mtk_drm_plane.c 3、mtk_drm_drv.c 3 …）
error 总数：331 → 254
```

`mediatek_v2/Makefile` 现在只编"点灯必需"的文件（DDP/DSI/DSC/OVL/RDMA/panel_ext + 核心 + cmdq 桩），
调试、fbdev、mmp、dsc_wrap、od/rsz/wdma/merge 等都可后补。

**下一轮的唯一动作**（高成功率、机械）：

```bash
# 取 MT6895-Mainline 已现代化过的同名文件
for f in mtk_drm_ddp.c mtk_drm_ddp_comp.c; do
  curl -o /tmp/$f https://raw.githubusercontent.com/MT6895-Mainline/linux/7.2-mt6895-xiaomi-xaga/drivers/gpu/drm/mediatek/mediatek_v2/$f
  diff -u linux-7.2.8/drivers/gpu/drm/mediatek/mediatek_v2/$f /tmp/$f | head -200
done
# 把他们的现代化改动移植过来（我们只保留 MT6885/MT6893 的 SoC 数据分支）
```

移植后重编 ⇒ 若只剩零散错误即可进入 DT/面板接线，然后 `port/tools/build-images.sh` 打包 v2 镜像。

## 第 7 轮：CMDQ 垫片（331 → 170）

- 25 个 v2 源文件 `#include <linux/soc/mediatek/mtk-cmdq.h>`（mainline 版，签名不同）
  改为指向本地 **vendor 时代垫片** `vendor-include/linux/soc/mediatek/mtk-cmdq.h`
  （由 `mtk_cmdq_dummy.c` 的原型自动生成 ⇒ 与 v2 调用点一致）；
- `cmdq_base` 参数改为 `void *`（类型无关）；
- 错误：331 → 254 → **170**；
- 剩余主要三类：`cmdq_pkt_write` 仍有一处指针参数不匹配（101）、`cmdq-sec.h` 缺失（2）、
  `cmdq_register_device`/`cmdq_async_flush_cb` 等厂商独有符号（4~6）。

**下一步（无网络依赖，纯机械）**：把 `cmdq_pkt_write` 的声明与定义都改成
`s32 cmdq_pkt_write(struct cmdq_pkt *pkt, void *base, ...)`（变参即可吞掉任意实参），
再为 `cmdq-sec.h`/`cmdq_register_device`/`cmdq_async_flush_cb` 补空实现，
预期错误降到个位数，然后进入 DT/面板接线并打包。


## ★ 转折：改用 MT6895-Mainline 的**已移植树**作为基座（本地 clone：`port/mt6895-linux`）

不再自己猜着现代化。做法（全部本地、按他们的代码为准）：

1. 备份我们的厂商树 → `vendor-v2-base/`（留作对账），把**他们的 `mediatek_v2/`（133 文件 / 4.9 MB）**
   原样放进 `linux-7.2.8/drivers/gpu/drm/mediatek/mediatek_v2/`；
2. 抄他们的 **Makefile 门控**（`CONFIG_DRM_MEDIATEK_V2=y` 时构建 `mediatek_v2/` 而**不是**上游 mediatek-drm）
   与 **Kconfig**（`source .../mediatek_v2/Kconfig`）；
3. `port/tools/v2-copy-from-mt6895.sh`：反复执行 —— 读编译日志里缺的头文件，从他们的树里按原路径
   （或按文件名）拷进我们的树。每跑一轮少几个；
4. 同时拷入他们 v2 需要的姊妹件：`drivers/gpu/drm/mediatek/mml/`、`drivers/interconnect/mediatek/`、
   `drivers/soc/mediatek/mtk-scpsys-bringup.c`、`drivers/misc/mediatek/mtk-interconnect/`、slbc 等头文件。

**结果（关键变化）**：错误性质从"API 级迁移（284 处）"变成"**只缺头文件**" ✓✓ ——
说明他们的代码在 7.2 上本体是编得过的，剩下的是把依赖补齐的机械循环。

继续方式（可反复执行直到通过）：

```bash
port/tools/v2-copy-from-mt6895.sh
cd linux-7.2.8 && make O=../build/chopin ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j16 Image
```

### 循环收敛记录（MT6895 树为基座）

| pass | 新拷头文件 | 错误总数 | 下一个缺失 |
| --- | --- | --- | --- |
| 1 | 1 | 39 | mmprofile_function.h |
| 2 | 2 | 37 | cmdq-util.h |
| 3 | 1 | 35 | mtk_clkbuf_common.h |
| 4 | 0 | 35 | ——（不再缺头文件）|

**剩余错误的性质变了**：不再是缺头文件/API 迁移，而是**厂商日志宏与 7.2 的格式检查冲突**
（`printk(KERN_INFO pr_fmt(fmt), ##__VA_ARGS__)` 一类，`-Wformat` 报错），集中在
`mtk_drm_crtc.c`(101)/`mtk_drm_drv.c`(60)/`mtk_disp_ovl.c`(40)，属于**受控的一类**：
把他们 `mtk_log.h` 的用法与其配置对齐（或去掉我们 `include/` 下拷贝了他们改过的核心头）即可推进。

证据（"这是他们的方案"）：v2 树 `diff -rq` 与 `port/mt6895-linux` **完全相同**；
`mediatek,mt6893-disp → mt6885_mmsys_driver_data` 是他们代码里原有的（mtk_drm_drv.c:6024）；
Makefile 门控逐行照抄。

### 修复进展（40 → 4）

| pass | 修复 | 错误 |
| --- | --- | --- |
| — | 用他们 `include/linux/mailbox/mtk-cmdq-mailbox-ext.h` 正确版本（先前按文件名拷到了 drivers/ 那份，缺 typedef）| 32 |
| — | `cmdq_mminfra_power`/`cmdq_mminfra_gce_cg` typedef 提到 `#if IS_ENABLED(CONFIG_MTK_CMDQ_MBOX_EXT)` 之外 | 30 |
| — | 拷 `double_click.h`、`mt-plat/mrdump.h`、`soc/mediatek/mmqos.h`、`dt-bindings/interconnect/mtk,mmqos.h` | 10 |
| — | `CFLAGS_REMOVE_mtk_drm_crtc.o := -mgeneral-regs-only`（该文件用浮点）| 4 |

**剩余 4 个**（同一处）：`mtk_mipi_tx.c` 的 `backup_mipitx_impedance` / `refill_mipitx_impedance`
（以及 `_mt6983` 两个）在 `mtk_mipi_tx_driver_data` 初始化里被引用，但
**在 MT6895 树里搜不到定义**（既不在 `mediatek_v2/`，也不在 `drivers/`、`include/`）。
⇒ 下一轮定位：这些多半来自他们另一处 vendor 文件（不同拼写或按配置条件编译），
或需要在我们的树里给出对应实现/去掉这四个字段的初始化。

### 剩余 4 个错误的定位结论（已查明，下一轮直接修）

`mtk_mipi_tx.c` 里：
- 它们的**定义确实在**（MT6895 树同文件 **第 2980 / 3066 行起**：`static void backup_mipitx_impedance(struct mtk_mipi_tx *)` 等）✓
- 但**驱动数据表在第 3463/3568 行就在用它** ⇒ 顺序问题 ⇒ 需要前向声明；加上前向声明后，
  报错从"undeclared"变成 **"incompatible pointer type ‘void (*)(struct mtk_mipi_tx *)’ from ‘void (*)(struct mtk_mipi_tx *)’"**
  ⇒ 说明该文件里 **存在两个不同的 `struct mtk_mipi_tx` 定义**（本地定义 vs 头文件里的），
  两处同文本类型的函数指针互不兼容。
- 正确修法（小改动，下一轮做）：把这两处**驱动数据表移到定义之后**，或统一 `struct mtk_mipi_tx` 的来源
  （删掉/改名重复定义），而不是加前向声明。

（我试过的两版改动都已回退，当前树 = 他们的原始文件 + 我们的 Makefile 编译选项，
错误稳定在 **4**，是干净的可继续点。）

### CMDQ 链接 + 最后 6 个编译错误（现状）

- 从 MT6895 树拷入 `drivers/misc/mediatek/cmdq/`（264K）并接线（`drivers/misc/mediatek/{Kconfig,Makefile}` +
  `drivers/misc/{Kconfig,Makefile}` 挂载，`CONFIG_MTK_CMDQ_MBOX_EXT` 跟随 `DRM_MEDIATEK_V2` 自动开启）；
- `undefined reference` 从多处 → **0** ✓（`cmdq_pkt_write` 等已由 `mtk-cmdq-helper-ext.c` 提供）；
- 又拷入 `include/dt-bindings/gce/mt6895-gce.h`、`include/linux/soc/mediatek/mtk_sip_svc.h`、
  `drivers/misc/mediatek/iommu/iommu_debug.h` 等；
- **剩余 6 个编译错误**，三类：
  1. `cmdq-util.c`: `mtk_smi_dbg_hang_detect` 未声明（从他们 `smi` 头拷即可）；
  2. `mtk-cmdq-helper-ext.c`: 缺 `vcp.h`（拷他们的或将该文件排除出构建）；
  3. **`mtk_mipi_tx.c`: 我加的 4 行前向声明与真实定义「conflicting types」** —— 根因仍是该文件里
     **存在两个 `struct mtk_mipi_tx` 定义**；正确修法是**把两处驱动数据表移到定义之后**（或统一 struct），
     而不是加前向声明＋强制转换（我的临时补丁到此为止，需要按结构修）。

### ★ v2 全树编译通过（0 compile error），进入链接期

从 MT6895 的 build 里找到关键宏：`MTK_FILL_MIPI_IMPEDANCE`（他们在构建时定义），
`mtk_mipi_tx.c` 的 4 个回调定义与使用它们的驱动数据表都在这个宏里 —— 我们**定义它**之后，
之前所有 mipi_tx 报错全部消失（我加的临时前向声明/强制转换已全部回退，文件恢复原样 ✓）。

```
compile errors: 0   undefined reference: 20（链接期）
```
剩余未定义符号清单：`port/audit/v2-undefined-symbols.txt`（下一轮按符号去 MT6895 树里找实现文件并纳入构建）。
