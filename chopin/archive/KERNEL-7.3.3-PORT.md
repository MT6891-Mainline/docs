# Linux 7.3.3 移植记录

2026-11-08。把 chopin port 从 Linux 7.3.0 迁到 7.3.3，并重新整理提交结构。

## 为什么升级

原来基于 Linux 7.3（上游 release 提交 `8c08b0269`），适配了 7.3.0 的 API。
7.3.3 是 7.3 系列的稳定版本，包含约三个月的上游修复。跟进可以避免 port 在
老接口上越陷越深。

## 结果

| 项 | 值 |
| --- | --- |
| 基线 | `b536183 Linux 7.3.3` |
| port 提交 | 17 个（按子系统） |
| 变更文件 | **1,347**（与 7.3.0 的 port 完全一致） |
| 内核版本 | `Linux version 7.3.3-g<commit>` |
| 构建 | 通过，35 个警告、0 个错误 |
| 启动 | v198 实测开机正常 |

## 提交结构

原来 7.3.0 的树有 **261 个提交**，其中大量是 DTS 迭代产生的噪音（切片拆分、
单文件化、label 化、覆盖化、phandle 删除与回退……）。7.3.3 的树按**子系统**
重新整理成 17 个：

```
54f1d5f  firmware: add the chopin panel and touch blobs
cfa29f4  chopin: remaining platform bits
d23cb37  include: add the chopin driver interfaces
a3aac2b  misc: mediatek: add the chopin platform drivers
73ef03e  arm64: mediatek: add the chopin board and DTS
49db9cd  drm: mediatek: add the chopin display stack
0f5f0d4  net: add the chopin WLAN hooks
88c0206  ASoC: mediatek: add MT6893 audio support
4a24a02  usb: add the chopin USB and PD support
420d7a1  input: add the chopin touchscreen drivers
2b42b65  power: supply: add the chopin charger and gauge
4be6e0c  regulator: add the chopin PMIC regulators
4b8a3f1  pinctrl: mediatek: add MT6893 pinctrl support
5605d82  interconnect: mediatek: add MT6893 interconnect support
fcf5daf  soc: mediatek: add MT6893 power, chipid and SMI support
fd8114a  clk: mediatek: add MT6893 clock controllers
8b5f3a6  dt-bindings: add MT6893 support
b536183  Linux 7.3.3
```

顺序按依赖排：先 bindings、再 SoC 控制器、最后板级和固件。每个提交可以独立
review，适合作为上游提交序列。

## 迁移步骤

```sh
# 1. 解压 7.3.3 并建基线
mkdir -p work/linux-7.3.3
tar -xJf linux-7.3.3.tar.xz -C work/linux-7.3.3 --strip-components=1
cd work/linux-7.3.3 && git init && git add -A && git commit -m "Linux 7.3.3"

# 2. 从旧树导出完整的 port 差异
git -C git/chopin-linux-7.3 diff --binary 8c08b0269..HEAD > work/port-7.3.0.patch

# 3. 应用
git apply --whitespace=nowarn /path/to/port-7.3.0.patch

# 4. 补固件（见下）
cp git/chopin-linux-7.3/firmware/novatek/*.bin work/linux-7.3.3/firmware/novatek/

# 5. 配置（注意 O= 的规则）
cp build/chopin-7.3/.config work/build-7.3.3/.config
make -C work/linux-7.3.3 O=$PWD/work/build-7.3.3 ARCH=arm64 olddefconfig

# 6. 构建
make -C work/linux-7.3.3 O=$PWD/work/build-7.3.3 ARCH=arm64 \
     CROSS_COMPILE=aarch64-linux-gnu- Image dtbs -j8
```

## 三个坑

### 1. `git diff` 默认不含二进制内容

第一次导出的补丁只有 36 MB，应用后 `firmware/novatek/*.bin` 全是空文件，
构建在 `drivers/base/firmware_loader/builtin/` 报：

```
error: empty firmware: .../novatek_ts_fw02.bin
make[7]: *** 没有规则可制作目标 "firmware/novatek/novatek_ts_fw02.bin"
```

加 `--binary` 能生成二进制补丁，但**它并不可靠**——这次加了 `--binary` 之后
文件依然是空的。最终是直接从旧树 `cp` 那 5 个文件解决的。

**结论：跨树移植时二进制文件单独复制，不要依赖补丁。**

### 2. `O=` 外置构建时 `.config` 必须放构建目录

把 `.config` 拷进源码树会让内核报：

```
*** The source tree is not clean, please run 'make ARCH=arm64 mrproper'
```

因为 `outputmakefile` 检查 `$(srctree)/.config` 是否存在。正确做法是放
`work/build-7.3.3/.config`。

### 3. `firmware/.gitignore` 排除 `*.bin`

内核固件目录的 `.gitignore` 里有 `*.bin`，`git add` 会静默跳过这些文件。
必须 `git add -f`。

这个坑很隐蔽：工作区文件在、构建能过，但 `git ls-files` 是空的，
`git diff --stat` 比预期少 5 个文件。

## 验证

```sh
# 文件完整性（必须等于 7.3.0 的 port）
git diff --name-only b536183 HEAD | wc -l        # 1347
git -C git/chopin-linux-7.3.0-backup diff --name-only 8c08b0269..HEAD | wc -l   # 1347

# 构建
make -C git/chopin-linux-7.3 O=$PWD/build/chopin-7.3 ARCH=arm64 \
     CROSS_COMPILE=aarch64-linux-gnu- Image dtbs -j8

# 版本
strings build/chopin-7.3/arch/arm64/boot/Image | grep -m1 "Linux version"
```

## 当前目录布局

```
git/chopin-linux-7.3/              ← 7.3.3 + 17 个 port 提交（在用）
git/chopin-linux-7.3.0-backup/     ← 旧的 7.3.0 + 261 个提交（保留备查）
build/chopin-7.3/                  ← 构建目录（.config、Image、DTB）
work/port-7.3.0.patch              ← 迁移用的补丁（36 MB）
work/linux-7.3.3/                  ← 迁移过程的工作副本
work/build-7.3.3/                  ← 迁移过程的构建目录
```

`port/tools/build-kernel.sh` 的 `src` 默认指向 `git/chopin-linux-7.3`，
替换后无需修改。

## 回退

```sh
mv git/chopin-linux-7.3 git/chopin-linux-7.3.failed
mv git/chopin-linux-7.3.0-backup git/chopin-linux-7.3
```

或者直接刷入 `out-scp-v196-restore/boot-stockdt.img`（7.3.0 的启动镜像）。
