# 硬件视频解码（VDEC）移植记录

Droidloom/flatpak 里的视频（wiliwili 等）目前是**软件解码**：实测播放时
wiliwili 占 **625% CPU**、系统 92% user、GPU 810 MHz 满频但仍然卡——瓶颈在
解码，不在 GPU。

## 现状

| 项 | 状态 |
| --- | --- |
| `/dev/video*`、`/dev/media*` | 不存在（无 V4L2 解码器） |
| `CONFIG_VIDEO_MEDIATEK_VCODEC` | `=m`（模块，initramfs 不带） |
| `CONFIG_VIDEO_HANTRO` | `=m` |
| DTS vcodec 节点 | **没有**（只有供应商绑定的 `mediatek,mt6885-vcodec-dec`，在 `-multimedia.dtsi` 里，主线驱动不认） |

## 已经具备的基础设施

MT6893 的 VDEC 与主线已支持的 **MT8192 布局完全一致**：

| 组件 | 状态 |
| --- | --- |
| 时钟驱动 | ✅ `clk-mt6893-vdec.c`（VDEC0/1 门控 + `CLK_TOP_VDEC_SEL`） |
| 电源域 | ✅ DTS `MT6893_POWER_DOMAIN_VDEC0/VDEC1` |
| 子系统时钟控制器 | ✅ `vdecsys@1602f000`、`vdecsys_soc@1600f000` |
| IOMMU 端口 | ✅ `M4U_PORT_L4/L5_VDEC_*` |
| 关键地址 | ✅ 0x16000000 / 0x1600f000 / 0x1600d000 与 MT8192 相同 |

## rv SCP 驱动尝试（2026-11-07，失败）

目标：让主线 `mtk-vcodec` 用上主线的 SCP IPI 接口（vcodec 与固件通信的唯一途径）。

做法：

1. `CONFIG_MTK_SCP=m` → `=y`（启用 rv SCP 驱动，提供 mbox/Tinysys IPI）
2. DTS `core_0 = "enable"` → `"disable"`（rv 自己的启动序列会卡在固件锁，
   历史上已验证；启动交给自研 `scp-chopin`）
3. 把 rv 驱动和 IPI 驱动里的 8 处 `BUG_ON(1)` 改成 `WARN_ON(1)`（避免 panic 重启）
4. 关闭厂商 sensorhub（`CONFIG_MTK_SENSORHUB=n`，我们用自己的 chopin-als/accel）

结果：**v188、v189 都是刷完直接重启** ✗，连日志都拿不到。

判断：问题不在 `BUG_ON`（已改成 WARN_ON 仍重启），而是 **rv 驱动 probe 的硬件
操作与自研启动序列冲突**：

- `scp_init()` 里有 `SCP_DVFS_INIT_ENABLE=1` → `scp_dvfs_init()` +
  `wait_scp_dvfs_init_done()`（DTS 没有 `scp_dvfs` 节点）
- rv probe 还会做 `scp_awake_init()` 的握手
- 这些操作与自研驱动已经启动的 SCP 撞车 → SCP 看门狗复位整机

## 下一步（下次继续）

**把自研的启动序列搬进 rv 驱动**，而不是让两个驱动并存：

1. `SCP_DVFS_INIT_ENABLE` 改成 0（跳过 DVFS 初始化）
2. rv 的启动路径（`scp_awake.c` 的 `scp_sys_full_reset()` +
   `scp_helper.c` 的 boot 流程）替换为 `scp-chopin.c` 里验证过的寄存器序列：
   ```
   scp_enable_sram() → 时钟 → 双核复位 → cleartcm/loadloader/reginfo/dramrestore
   → 写 DRAM_RESV/SIZE → 释放 core0
   ```
3. rv 的 probe 只保留 mbox + IPI 表 + 平台设备注册
4. 关掉 `scp-chopin` 的启动职责（保留 debugfs 调试）
5. 通了之后再加 vcodec 节点（阶段 2）

**备选**：在 `scp-chopin.c` 里自己实现最小 mbox/IPI（避开 rv 驱动），
参考 `drivers/soc/mediatek/mtk_tinysys_ipi.c` 和 rv 的 `scp_ipi` 表。

## 风险

- 厂商 SCP 固件里是否含 VDEC 任务未验证（从原厂分区提取的 scp.img）
- SCP 的 IPI 表必须与固件里的 `scp_ipi_table.h` 完全一致（DTS 里有注释警告）
- 每次验证都要刷 boot，失败就重启循环，迭代成本高
