# OP6893 的 DTB 与 LK 启动交接

OP6893 使用两套 DTB：boot.img 中的原厂 DTB 供 LK 使用；编入 Linux
Image 的 OP6893 DTB 描述主线驱动需要的硬件。LK 处理完原厂启动镜像和
DTBO 后，仍按 arm64 启动约定把它最终生成的 FDT 物理地址放在 x0。Linux
用这份 FDT 提取本次启动的内存布局，然后用合并后的树完成早期扫描。

## boot.img 中保留什么

打包以能正常启动的本机原厂镜像为模板，保留 boot header 的型号相关策略
和加载地址、header DTB，以及原来有的话 gzip 内核后的原厂 DTB 链。
21651 已用的模板有 header DTB、没有追加链；无需给它额外追加一份。
打包时放入主线 Image.gz，initramfs 编进内核，外部 ramdisk 仅保留一个
空归档。我们不把 Linux 板级 DTB 放进 LK 要处理的 DTB 槽位，也不要求
修改 dtbo 分区。这样保留了 LK 依赖的原厂 DTB/DTBO 配对。

## Linux 如何构造最终设备树

`setup_arch()` 在初始化 early fixmap 后、第一次
`setup_machine_fdt()` 前调用 `op6893_prepare_fdt()`。它只读映射
x0 指向的原厂 FDT，检查 FDT 头和 MT6893 compatible，再调用
`op6893_merge_fdt()`，以编入内核的板级 DTB 为底稿，在独立缓冲区构造
新的 FDT。板级 DTB 由所选构建配置决定；20615 与 21651 共用底稿，
21651 用板级 DTS 覆盖具体差异。

| LK 交接内容 | Linux 的处理 |
| --- | --- |
| 根下的内存节点 | 复制内存银行的 `reg` 和 `device_type`。至少需要一个内存节点，地址和大小使用两单元编码。 |
| `/reserved-memory` 和 FDT 预约表 | 保留有效的 `reg` 或动态 `size`，以及 `alignment`、`alloc-ranges`、`no-map`；合并按节点名定位。原厂的设备绑定和 phandle 不导入，主线树自有的节点与引用仍由主线树定义。 |
| `/chosen` | 使用主线 DTS 的启动参数，不导入原厂 bootargs 或外部 initrd 地址。可选的 `atag,devinfo` 经长度与字数校验后转交给 LVTS。 |
| 基带 LK 元数据 | 只有板级 DTB 提供了能核对该范围的元数据时，才会导入经校验的 `ccci,modem_info_v2` 头；校验还要求指向的范围同时落在板级约束和 LK 的保留内存内。 |
| 21651 显示交接 | 仅在启用 framebuffer 路径且 LK 保留的扫描缓冲区地址、长度均匹配时，才启用对应节点。 |

合并完成后释放原厂 FDT 的临时 fixmap，返回新 FDT 的物理地址。
`setup_machine_fdt()` 只扫描新树，因此 bootargs、内存银行和后续
reserved-memory 处理都基于同一份树。当前移植另有一个诊断约束：它要求
原厂 FDT 包含 ACK 的 pstore 保留区（基址 `0x48090000`、大小
`0xe0000`），验证后映射其中的早期日志缓冲区。这个约束用于现有
OP6893 镜像的取证，不是其他 MT6893 设备的启动要求。

## 和 Chopin 分支的区别

两边都让 LK 先处理原厂树。Chopin 当前先扫描 LK FDT，然后换为内嵌
Chopin DTB，并重新扫描 `/chosen`；OP6893 则在第一次扫描前合并，
把 LK 实际给出的内存银行和保留区带进最终设备树。共享分支第一步将
两条路径按构建配置隔离，Chopin 的固定地址早期日志写入也只在 Chopin
配置启用。后续可比较两边的内存差异，再决定是否共用一套合并代码。

旧 OP6893 工作树已有 20615、21651 的真机启动记录。基于 Chopin
`f1c79a79e` 的共享分支首轮集成已在 21651 上验证：镜像
`c979b8113198` 从 boot 分区启动，LK 传入的 FDT 地址为
`0x47c80000`，合并后的 FDT 由 `/sys/firmware/fdt` 导出，
`/memory/reg` 与原厂内核的现场 FDT 一致。UFS 已枚举 userdata，
系统切换到 ext4 根文件系统，systemd 和 USB SSH 正常。20615 和
Chopin 尚无共享分支的真机结果。

源码入口：`arch/arm64/kernel/setup.c`、
`arch/arm64/kernel/op6893-boot.c`、`arch/arm64/kernel/op6893-fdt.c`。
本地打包脚本：`mt6893-direct/pack.py`。
