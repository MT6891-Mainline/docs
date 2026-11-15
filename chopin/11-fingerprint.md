# 11 · 指纹（Goodix GF3626ZS2）

走的是**路线 C：把厂商 Goodix trustlet（TA）跑在用户态**，匹配在 TA 内部完成。
采集通道仍未闭环。

## 状态

| 环节 | 状态 |
| --- | --- |
| 内核 SPI 传输（`goodix_fp`） | ✅ |
| 上电/复位/芯片 ID | ✅ `00 10 a0 00 25` |
| TA 在用户态跑起来 | ✅（`ta_host32` + MicroTrust TEE stub） |
| 指纹图像采集 | ❌ 只回来 4096 字节，其余全 0 |
| 录入 / 认证 | ❌ |

## 关键结论

* 厂商 TA 是 ARM ELF32，可以在用户态解释执行，不依赖真实 TEE；
* 命令 ABI：`ut_pf_fp_invoke_command(desc, req, len)`，`req` = 16 字节头
  `{cmd, in_len, out_cap, status}` + payload；
* `capture(1021)` 只是 `gf_hw_set_mode(1)`，真正读图走寄存器 `0xAAAA`（27,365 字节）；
* 传感器实际只驱动前 4096 字节就停（内核时钟了全部字节，不是截断）；
* 传感器自检报 `GF_ERROR_SENSOR_TEST_FAILED 1063`，`otp_parse_dac: need to update dac bit8`。

## 文档

* [`archive/FINGERPRINT-PORT.md`](archive/FINGERPRINT-PORT.md) — 移植总览
* [`archive/TA-IN-USERSPACE.md`](archive/TA-IN-USERSPACE.md) — 路线 C 的完整逆向记录（M1.42–M1.53）
* [`archive/fingerprint-HANDOFF-2026-11-08.md`](archive/fingerprint-HANDOFF-2026-11-08.md) — 交接

## 内核侧改动（`git/chopin-linux-7.3` 的 `fp` 分支）

| 提交 | 内容 |
| --- | --- |
| `input: add the Goodix GF5xxx SPI transport` | 新增 SPI 驱动 |
| `input: goodix_fp: add GOODIX_FP_IOC_XFER` | 全双工 raw SPI ioctl |
| `input: goodix_fp: compat_ioctl for AArch32` | 32 位 stub 的 ioctl 兼容 |
| `arm64: dts: chopin: give the fingerprint bus the vendor's CS timing` | CS 时序 |
