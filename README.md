# chopin 文档

Redmi Note 10 Pro（MT6891 / "chopin"）把 Android 厂商内核换成主线 Linux 的
完整记录：踩过的坑、验证过的结论、以及为什么某些路走不通。

* 内核源码：`git/chopin-linux-7.3`
* 固件/服务/配置：`git/chopin-support`
* **启动 initramfs（PID 1）**：`git/chopin-initramfs`
  * 分支 `rootfs` —— 设备实际启动的极简 PID 1（557 行）
  * 分支 `recovery` —— 救援镜像：Alpine + 硬件诊断 + USB shell（4416 行）
* 文档：本仓库

---

## 从哪开始

| 想知道什么 | 读这个 |
| --- | --- |
| **现在能干什么、不能干什么** | [`status/STATUS.md`](status/STATUS.md) |
| 电量计为什么读不到、PD 为什么不协商 | [`power/CHARGING.md`](power/CHARGING.md) |
| GPU 怎么从 350 提到 810 MHz | [`gpu/GPU-PERF.md`](gpu/GPU-PERF.md) |
| 挂起为什么放弃（10+ 轮完整记录） | [`power/SUSPEND-ANALYSIS.md`](power/SUSPEND-ANALYSIS.md) |
| 显示时钟链路 | [`display/DRM-CLOCK-GAP.md`](display/DRM-CLOCK-GAP.md) |
| 厂商 DRM 有什么坑 | [`display/V2-DRM-AUDIT.md`](display/V2-DRM-AUDIT.md) |
| rootfs 怎么构建、GNOME 怎么配 | [`rootfs/DEBIAN-ROOTFS.md`](rootfs/DEBIAN-ROOTFS.md) |
| 开机那一刻发生了什么 | `git/chopin-initramfs`（`rootfs` 分支的 README） |

## 目录

### `status/`
当前状态快照：功能矩阵、性能实测、已知限制、镜像清单。

### `power/`
| 文档 | 内容 |
| --- | --- |
| `CHARGING.md` | 电量计（BQ27Z561）+ PD 快充。含**两个上游 bug**的定位过程 |
| `SUSPEND-ANALYSIS.md` | 挂起的三道坎，以及为什么放弃 |
| `SUSPEND-NOTES.md` | 更早的挂起笔记 |

### `gpu/`
| 文档 | 内容 |
| --- | --- |
| `GPU-PERF.md` | 三个性能缺口的全部实验：PLL 安全重锁（+77%）、电压、带宽 |
| `GPU-PORT.md` / `GPU-HANDOFF.md` | 早期移植笔记与移交 |

### `display/`
| 文档 | 内容 |
| --- | --- |
| `DRM-CLOCK-GAP.md` | 显示时钟链路调查（最长的一篇） |
| `V2-DRM-AUDIT.md` | 厂商 `mediatek_v2` DRM 的漏洞审计 |
| `DSC-REFERENCE.md` | DSC 参考资料（迁移主线的硬阻塞） |
| `NEXT-DRM.md` | 早期 DRM 计划 |
| `UEFI-PORT.md` | U-Boot/UEFI 尝试记录（最终走 preloader→LK 路线） |
| `PMIC-IRQ-NOTES.md` | PMIC 中断笔记 |

### `sensors/`
SCP RV33 传感器链、加速度计、自动旋转。`REVIEW-ACCEL-*.md` 是逐轮审查记录。

### `usb/`
| 文档 | 内容 |
| --- | --- |
| `OTG.md` | USB 主机/OTG 调查：供电链路已通，端口使能失败（未完成） |

### `connectivity/` `audio/` `rootfs/` `build/`
WiFi/BT、音频、Debian rootfs 与 GNOME、7.3 内核迁移。

### `archive/`
过期的手递文档、计划、总结。保留是因为里面有历史决策的推理过程。

---

## 几条被反复验证的硬结论

1. **没有 U-Boot/UEFI**：`preloader → LK → GenieZone → Linux`。
   boot 镜像必须带厂商 DTB，自有 DTB 在 `setup_arch()` 里替换。

2. **显示只能用厂商 `mediatek_v2`**：主线 `mediatek/` 缺 DSC 支持，
   而这块屏是 DSC 视频模式。迁移主线 = 先补几百行 DSC。

3. **挂起不可行**：`mtu3` + CMDQ wakelock 双重阻塞，且修复手段会截断
   CMDQ 事务导致卡死。熄屏已能省 0.5–1.5 W，足够。

4. **I2C 读全 0xFF 先查总线**：i2c0 在 400 kHz 下所有设备读不出数据，
   降到 100 kHz 一切正常。**对照实验比猜设备状态有效得多。**

5. **测量协议本身就是数据**：改频率后的首轮系统性偏慢；连续扫频会被热节流
   污染。结论错误往往来自测量方法，而不是硬件。

---

## 写文档的约定

* 结论要带**实测数据**，不要写"应该可以"
* 失败要写**为什么失败**，以及做了什么对照实验
* 推测要显式标注为推测
* 数据不可靠时**主动标注**（例如"这组用了有缺陷的扫频协议"）
