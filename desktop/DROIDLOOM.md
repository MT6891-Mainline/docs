# Droidloom：Android 应用跑在 Wayland 窗口

Droidloom 把 Android 应用放进原生 Wayland 窗口：宿主上跑一个 Android cell
（私有 mount/PID/net 命名空间 + Android init），显示经宿主 Wayland 合成器，
GPU 经 `droidloom-gpu-server` 转发。

**结论先说**：在 chopin 上 Droidloom 的宿主侧和一个 Android cell 都能起来，
但 Android 的显示栈断在 EGL，SurfaceFlinger 崩溃循环，**跑不了任何安卓应用**。
根因是 Droidloom 镜像里 ANGLE 的打包位置（上游 bug），以及 Mesa 的 PanVK
根本不支持 Mali-G77（v9）。

## 架构与数据流

```
Android app → SurfaceFlinger → EGL → ANGLE → droidloom-vulkan (HAL)
            → droidloom-gpu-server (宿主) → 宿主 Vulkan → GPU
```

GPU 由 `Droidloom setup` 按 `cell.json` 的 `gpu` 选择写进
`/etc/droidloom/runtime.env`，`droidloom-gpu-server` 从 `DROIDLOOM_GPU` 读设备名。

## 我们做的改动

### 1. Droidloom：Mali/panfrost 支持（4 文件，+53/−19）

| 文件 | 改动 |
| --- | --- |
| `runtime/droidloom-supervisor/src/host_gpu.rs` | `panfrost → Vendor::Mali`；Mali 走 GPU 转发；平台设备（无 PCI）允许；`mali` 选择项；新增单元测试 |
| `runtime/droidloom-gpu-server/src/gpu.rs` | `render_node()` 支持平台设备（按 `/sys/class/drm/renderD*/device` 反查） |
| `runtime/droidloom-supervisor/src/bin/droidloomctl.rs` | `gpu` 子命令帮助 |
| `runtime/droidloom-package-support/src/main.rs` | setup 的 GPU 错误消息 |

`cargo test -p droidloom-supervisor --lib` → 122 passed（含
`a_mali_forwards_on_its_panfrost_render_node`）。

验证：`droidloomctl gpu` 输出

```
Mali GPU 13000000.mali (panfrost, /dev/dri/renderD128)
GPU choice auto: Android renders on the Mali GPU ... through droidloom-gpu-server
  (GPU forwarding); the cell's render node is /dev/dri/renderD128.
```

### 2. Droidloom：`unshare --forward-signals` 兼容

`--forward-signals` 是 util-linux 2.42 才有的选项；Debian 13 是 2.41，
cell 会以 `unshare: unrecognized option '--forward-signals'` 失败。
补丁在 `development.rs` 里检测 `unshare --help` 再条件添加。
（也可以在设备上把 2.42 的 `unshare` 单独装到 `/usr/local/bin`。）

### 3. Droidloom：GPU server 的 lavapipe 回退

当宿主 Vulkan 只有软件设备（lavapipe，无 PCI/平台设备名）时，
`probe.rs` 回退到第一个物理设备，`manual.rs` 把全部设备暴露给 Android。

### 4. 内核改动（v183–v187）

| 版本 | 配置 | 对应故障 |
| --- | --- | --- |
| v183 | `CONFIG_ANDROID_BINDER_IPC=y`、`ANDROID_BINDERFS=y`、`NF_TABLES=y` | Android IPC / 网络前置 |
| v184 | panfrost `DRIVER_SYNCOBJ_TIMELINE` | Denial 不广播 `wp_linux_drm_syncobj_manager_v1`，`droidloom-wayland` 绑定失败 |
| v185 | `DMABUF_HEAPS_SYSTEM=y`、`DMABUF_HEAPS_CMA=y` | `/dev/dma_heap/system` 不存在（原为模块，initramfs 不带） |
| v186 | `VETH=y` + 全套 nftables（`NF_CONNTRACK/NAT`、`NF_TABLES_*`、`NFT_*`） | `ip ... Unknown device type` / `execute nft` 失败 |
| v187 | `EROFS_FS=y`、`EROFS_FS_ZIP=y`、`FUSE_FS=y`、`OVERLAY_FS=y` | Android 镜像 system/product/system_ext 是 EROFS，挂载失败 |

这些都不是 hack：内核把容器/网络基础功能编成模块，而我们的 initramfs 不带模块，
内置是正确取舍；`DRIVER_SYNCOBJ_TIMELINE` 是 panfrost 相对 panthor/msm/nouveau
的真实缺失。

### 5. 用户态组件（设备上）

`nftables`、`iptables`、`pkexec`、`uidmap`（newuidmap/newgidmap）、
`mesa-vulkan-drivers`、`vulkan-tools`、`wayland-utils`。

## 卡住的地方

### ① EGL/ANGLE：SurfaceFlinger 崩溃循环

```
surfaceflinger: Abort message: 'no suitable EGLConfig found, giving up
  (format: 1, vendor: Android, version: 1.4 Android META-EGL, ...)'
  SkiaGLRenderEngine::chooseEglConfig
```

`vendor: Android ... META-EGL` 说明加载的是**桩实现**，不是 ANGLE。

镜像里 ANGLE 存在，但位置不符合 Android EGL 加载器的搜索路径：

```
/system/lib64/libEGL_angle.so        ← 实际位置
/system/lib64/libGLESv2_angle.so

/odm/lib64/egl/libEGL_angle.so       ← 加载器搜索（不存在）
/vendor/lib64/egl/libEGL_angle.so    （不存在）
/system/lib64/egl/libEGL_angle.so    （不存在）
```

`libEGL.so` 内含 `"/egl"` 与 `"/system/lib64"`，证实加载器拼的是
`/system/lib64/egl/...`。

**试过的修法**：用 `debugfs -w` 把两个 ANGLE 库写进 `vendor.raw.img` 的
`/lib64/egl/`（合法搜索路径）。结果 cell 变成 SIGSEGV（不是修好），已回滚。
说明这些 ANGLE 库在该配置下本身不完整/加载即崩，不是挪对位置就能用。

**这是 Droidloom 镜像的打包 bug，影响所有 forwarding 后端用户**
（NVIDIA 用户也会遇到同样的 `no suitable EGLConfig found`）。

### ② PanVK 不支持 Bifrost v9（G57/G77）

`mesa 26.2.6 panvk_physical_device.c`：

```c
case 6: case 7: case 14:   // 需要 PAN_I_WANT_A_BROKEN_VULKAN_DRIVER
case 10: case 12: case 13: break;      // 正式支持
default: "%s not supported"            // ← v9 走这里
```

v9 是 G57/G77 的架构。即使把镜像 ANGLE 修好，ANGLE→Vulkan 也只能落到
lavapipe（软件渲染）。PanVK 的 v9 支持是 Mesa 上游的功能空缺，不是移植问题
（PanVK 是 Mesa 自己的驱动，安卓内核里只有闭源 kbase，uAPI 与 panfrost 不同）。

## 当前设备状态

* 内核：v187（上述全部内置）
* Droidloom：已安装、已配置（`/etc/droidloom/cell.json`、`runtime.env` 指向 Mali）
* `droidloom-doctor`：**READY（0 阻断）**
* Droidloom 服务：**已停止**（cell 无法完成 Android 启动）
* Denial 桌面：正常，panfrost GL 直接驱动 Mali

## 复现步骤（排查用）

```sh
# 安装
#   pacman -U droidloom-*.pkg.tar.zst（或 Debian 包）
#   droidloomctl start --mode mobile   （tty1 上做 polkit 授权）

# 观察
droidloomctl status          # cell 是否运行、Android 启动到哪
droidloomctl logs            # Android logcat（看 surfaceflinger 崩溃）
droidloomctl crashes         # 崩溃缓冲
journalctl -b -u droidloomd.service -n 200 --no-pager

# GPU 检测
droidloomctl gpu             # 应显示 Mali GPU ... through droidloom-gpu-server
```

## 上游材料（如要提 issue/PR）

1. **Droidloom issue**：镜像的 `libEGL_angle.so` 在 `/system/lib64/`，
   但 EGL 加载器只搜 `/{odm,vendor,system}/lib64/egl/`，ANGLE 永远加载不上，
   forwarding 后端一律 SurfaceFlinger 崩溃。证据：上文的崩溃日志 + 文件布局。
2. **Droidloom PR**：本文件的 1/2/3 三组补丁（Mali 支持、unshare 兼容、
   lavapipe 回退）。
3. **mesa**：PanVK 缺 v9（Bifrost G57/G77）支持。
4. **内核**：panfrost 缺 `DRIVER_SYNCOBJ_TIMELINE`（一行修复，mainline 可接受）。
