# chopin（MT6893）主线移植后续工作计划

> **2026-10-01 更新**：上一版计划的 P0「让内核开口说话」已定位到根因并修好
> （log_store 驱动 SRAM 头偏移错 0x34 字节，属本移植自身的 bug，不是平台限制）。
> 现在的 P0 是**一次实测验证**，操作步骤见 `NEXT.md`，完整证据见 `SUMMARY.md`。

---

## 一、现在的状态（一句话）

诊断通道已经修好（理论上），只差上机验证。**不需要再买串口**也能继续，
但串口仍然是最省事的路子（见第五节）。

## 二、P0 — 一次实测验证（待你操作）

| 步骤 | 命令 | 期望 |
| --- | --- | --- |
| 1 | `fastboot flash boot_a port/dist/boot-probe.img` | 屏幕出现**某一种纯色**+黑横条 → LK 确实执行了我们的镜像，颜色告诉你哪个 fb 是活的 |
| 2 | `fastboot flash boot_a port/out/boot-owndtb.img` | 卡 30 秒（`panic=15` + 看门狗）后自动重启 |
| 3 | 进 Android 后**立刻** `su -c 'dd if=/dev/block/by-name/expdb of=/sdcard/expdb.img bs=1M'` + `adb pull` | 拿到内核日志 |

判据（这一次至少有四个独立标记）：

- `K00/K01/K02/K03` —— head.S 汇编级，物理直写，MMU 关闭
- `[BC nn] <阶段>` —— `setup_arch()` 9 个路点
- `[mtk-logstore]` / `Linux version 7.2.8` —— printk 文本
- `chopinport=ringfix1` + `jump to K64 0x40080000` —— 确认刷的版本与 LK 行为

## 三、P1 — 拿到日志之后

1. 若日志停在某个 `[BC nn]` → 那一步之后就是死点，按栈/驱动逐个排查。
2. 若 `[BC nn]` 齐全但 printk 中途断掉 → 看最后一行属于哪个子系统。
3. 若**完全没有**任何标记 → 说明 LK 没执行我们的镜像（不是内核问题），
   转去验证 boot.img 打包（未压缩 `RAW_KERNEL=1`、MTK v4 头部、dtb 字段）。
4. 若探针屏幕无变化但 expdb 有横幅 → 只是 fb 地址/格式不对，换个候选即可。
5. 若探针屏幕无变化且 expdb 也没有横幅 → LK 没执行 kernel 字段或 fb/SRAM 写入被拦，
   需要重新设计实验。

## 四、P2 — 内核真正跑起来之后

1. 把 `simple-framebuffer` 的地址改成探针确认的那个（现在先写 `0x77a20000`），
   让 `tty0` 有真实控制台；
2. 打开 `CONFIG_DRM_SIMPLEDRM`/`FB_SIMPLE` + `fbcon`，屏幕上直接看到 dmesg；
3. 考虑接一个 `mtk_wdt`（否则失败时 30 秒被硬复位，虽然对自动化测试反而方便）；
4. 外设按 **PMIC（`pwrap@10026000` + `mt6359`）→ 存储（UFS `0x11270000` /
   MMC `0x11230000`）→ USB → 显示（DSI）** 的顺序 bring-up；PMIC 是一切的前置。
5. 板级 dts 以 MT6893-Mainline 的 `mt6893.dtsi` 为基（已采纳），按需逐项 `okay`。

## 五、仍然推荐但非必需：串口

- **USB 逻辑分析仪**（8 通道 24MHz，几十元）：只接 GND + UART TX，
  用 PulseView/sigrok 解 921600 波特率。只读、无需电平匹配、无损坏风险。
- 或 **1.8V USB-TTL**（CP2102/FT232RL）。
- UART0 = `0x11002000`，SPI 109；需要时可以从原厂 pinctrl/DT 查出测试点。

## 六、明确"不要再做"的事

- ❌ 往 pstore 区（`0x48090000`）裸写 —— 该区在 28/28 份 dump 里全 0，确认不保留，
  而且会把下一轮 Android 的 zone 弄坏。已在驱动里删掉。
- ❌ 依赖 `expdb` 全盘 `strings` 搜索 —— 转存在分区**末尾 2 MB**，且每轮覆盖。
- ❌ 再去动 `CONFIG_EFI` / `text_offset` —— MT6895 证明 PE 头与 `text_offset=0` 都能跑。
- ❌ 把 `pl_lk_log` 当 SRAM 头起点用 —— 它在 `+0x34`（这就是这轮的根因）。
- ❌ 在 `paging_init()` 之前让 `vprintk_emit` 钩子持有 `early_ioremap` 映射 ——
  `early_ioremap_reset()` 会把槽拆掉，之后一次 printk 就把内核打死。
- ❌ 靠"多刷几次碰运气" —— 每次测试都要有明确判据。
