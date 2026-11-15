# 08 · 桌面

## 现在的桌面：denial

自研的 Wayland 合成器 + Flutter UI，以普通用户 `kily` 运行，登录入口是
`getty@tty1` 的 autologin（[`01-boot.md`](01-boot.md)）。

| 项 | 状态 |
| --- | --- |
| 合成器 / 锁屏 / 桌面 | ✅ |
| 输入（触摸、音量键） | ✅ |
| 渲染（Mesa + Panfrost） | ✅（GPU 锁 350 MHz，见 [`03-gpu.md`](03-gpu.md)） |
| 解锁（PAM） | ❌ `pam_unix(login:account): setuid failed`——会话不是 root，PAM 无法 setuid |

**PAM 这条是架构问题**：登录/锁屏必须由特权进程做 PAM 会话（greetd/ly/SDDM 那样），
或者走 logind+polkit 解锁，不能自己以普通用户跑 PAM。当前失败路径还可能请求
logind 重启整机。

## Droidloom（Android 应用跑在 Wayland 窗口）

尝试过，**跑不了安卓应用**：宿主机与 Android cell 都能起来，但 Android 显示栈断在 EGL，
SurfaceFlinger 崩溃循环。根因是镜像里 ANGLE 的打包位置（上游 bug）+ Mesa 的 PanVK
不支持 Mali-G77（v9）。

细节：[`archive/DROIDLOOM.md`](archive/DROIDLOOM.md)。
