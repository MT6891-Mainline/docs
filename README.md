# docs

各机型把 Android 厂商内核换成主线 Linux 的记录：踩过的坑、验证过的结论、
以及为什么某些路走不通。

**按机型分目录**，每个机型目录下再按**功能**分文件、并有一条**时间线**。

| 机型 | 平台 | 目录 |
| --- | --- | --- |
| Redmi Note 10 Pro（chopin） | MT6891Z / MT6893 | [`chopin/`](chopin/README.md) |
| Redmi K40 Gaming（ares） | MT6893 | [`ares/`](ares/README.md) |

---

## 每个机型目录里有什么

机型目录下先按**路线**分（Linux / UEFI），路线内再按**功能**分文件 + 一条时间线：

```
chopin/
├── README.md          机型总览：硬件、两条路线的入口
├── linux/             ✅ 在用的路线
│   ├── README.md         功能状态矩阵
│   ├── TIMELINE.md       按时间的移植过程
│   ├── 01..13-功能.md    启动/显示/GPU/电源/音频/无线/传感器/桌面/rootfs/媒体/指纹/USB/构建
│   └── archive/          原始长文（48 篇）
└── uefi/              ❌ 探路（Windows on ARM）
    ├── README.md
    └── archive/          UEFI / WoA 原始文档（2 篇）
```

> 以前的布局是"一个目录一个功能"，文档散在十几个目录里；现在改成
> 机型 → 功能文件 + 时间线，`archive/` 只做存档，不再增长。

## 相关仓库

| 仓库 | 内容 |
| --- | --- |
| `git/chopin-linux-7.3` | 内核与设备树 |
| `git/chopin-support` | 固件、服务、配置 |
| `git/chopin-initramfs` | 启动 initramfs（`rootfs` 分支：极简 PID 1；`recovery` 分支：救援镜像） |
| `git/docs` | 本仓库 |
