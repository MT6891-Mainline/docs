# 10 · 媒体（硬件解码）

**现状：没有硬件解码，视频全靠软解。** 播放 1080p 时 `wiliwili` 占 **625% CPU**、
系统 92% user，GPU 810 MHz 满频仍然卡——瓶颈在解码，不在 GPU。

## 状态

| 项 | 状态 |
| --- | --- |
| `/dev/video*` / `/dev/media*` | ❌ 不存在（无 V4L2 解码器） |
| `CONFIG_VIDEO_MEDIATEK_VCODEC` | `=m`（模块，initramfs 不带） |
| `CONFIG_VIDEO_HANTRO` | `=m` |
| 软解 | ✅（能用，但吃满 CPU） |

## 已试过

`rv` SCP 驱动的尝试（2026-11-07）失败；MT6893 的 VDEC 走的是 SCP 上的固件，
和传感器一样属于"AP 侧没有现成通路"的类别。

细节与下一步：[`archive/VDEC-PORT.md`](archive/VDEC-PORT.md)。
