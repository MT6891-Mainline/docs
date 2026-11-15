# 05 · 音频

外放可用。**chopin 没有 3.5mm 耳机口**，所以音频链路只有扬声器和麦克风。

## 硬件链路

```
AP AFE (0x11210000, IRQ SPI 202, I2S3 输出)
      └─> CS35L41 x2 (i2c6: 0x40 主, 0x42 副,
                       reset=GPIO30, switch=GPIO27, irq=GPIO31)
              └─> 扬声器

MT6359 (PMIC codec) 只用于麦克风/耳机检测，外放不经过它
```

## 状态

| 项 | 状态 |
| --- | --- |
| 外放（两路 CS35L41） | ✅ |
| 麦克风 | ✅（走 MT6359 codec） |
| 耳机 | —（无孔） |
| HDMI/DP 音频 | ❌（无 HDMI 输出） |

细节与调试过程：[`archive/AUDIO-HANDOFF.md`](archive/AUDIO-HANDOFF.md)。
