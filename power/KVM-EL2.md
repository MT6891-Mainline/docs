# KVM on chopin: which firmware decides EL2

## TL;DR

KVM works on this board, but only with the **MIUI 12.5 preloader**.  With the
preloader from the newer firmwares (MIUI 14 / OS2.0) the kernel boots at EL1
and KVM refuses to come up:

```
kvm [1]: HYP mode not available
```

With the 12.5 preloader the same kernel boots at EL2 in VHE mode and KVM works:

```
CPU features: detected: Virtualization Host Extensions
kvm [1]: disabling GICv2 emulation
kvm [1]: GIC system register CPU interface enabled
```

Everything else in the boot chain (LK, TEE, GenieZone, the boot image, the
rootfs) is unchanged.  This matches `display/UEFI-PORT.md` §3.1: LK hands the
payload to GenieZone, which stays at EL2 and starts Linux at EL1h
(`mode=0x3c5`).  A preloader that does not put a hypervisor in the way hands
the payload EL2 directly (`mode=0x3c9`), which is what the 12.5 image does for
us.

## The configuration that works

| Component | Version |
| --- | --- |
| preloader | **MIUI 12.5.5.0 (RKPCNXM, 2021-06-17)** |
| LK / TEE / GenieZone / SCP / SSMP / DPM / SP M | MIUI 14 low-level (the set that boots the port's normal images) |
| boot image | the port's usual image (OS2.0 device tree, `out-scp-*`) |
| rootfs | the port's usual Debian + Denial image |

The device boots to Denial as usual and additionally exposes `/dev/kvm`.

## Evidence

`dmesg` on the working combination:

```
[    0.000000] CPU features: detected: Virtualization Host Extensions
[    0.000000] arch_timer: [Firmware Bug]: VHE-capable CPU without EL2 virtual timer interrupt
[    0.557789] kvm [1]: disabling GICv2 emulation
[    0.558411] kvm [1]: GIC system register CPU interface enabled
```

`/dev/kvm` is created with the right major/minor and an ACL for the desktop
user:

```
crw-rw----+ 1 root kvm 10, 232 /dev/kvm
```

The API answers:

```sh
python3 -c '
import os, fcntl
fd = os.open("/dev/kvm", os.O_RDWR)
print("API", fcntl.ioctl(fd, 0xAE00, 0))   # KVM_GET_API_VERSION -> 12
print("VM ", fcntl.ioctl(fd, 0xAE01, 0))   # KVM_CREATE_VM
'
```

and a real guest boots under hardware acceleration (generic Debian arm64 kernel,
no root device, so it stops at the initramfs prompt):

```sh
qemu-system-aarch64 -accel kvm -M virt -cpu host -smp 4 -m 2048 \
  -kernel /boot/vmlinuz-6.12.115+deb13.1-arm64 \
  -initrd /boot/initrd.img-6.12.115+deb13.1-arm64 \
  -append "console=ttyAMA0" -nographic -monitor none -no-reboot
```

```
smp: Brought up 1 node, 4 CPUs
SMP: Total of 4 processors activated.
virtio-pci 0000:00:01.0: enabling device (0000 -> 0003)
input: gpio-keys as /devices/platform/gpio-keys/input/input0
virtio_net virtio0 enp0s1: renamed from eth0
(initramfs)
```

## Userspace side

`cpu-checker`, `qemu-system-arm` and `qemu-utils` from Debian trixie are enough:

```sh
apt-get install -y qemu-system-arm qemu-utils cpu-checker
kvm-ok        # INFO: /dev/kvm exists / KVM acceleration can be used
```

The generic `linux-image-arm64` package provides a guest kernel that runs on
QEMU's `virt` machine; the port's own kernel is built for MT6893 and has no
PL011/virtio support, so it is not a usable guest.

## Why this matters for the UEFI port

`display/UEFI-PORT.md` already relies on the payload being started at EL2h by a
NoGZ ATF ("打了 `kvm/mtk-mod-tee-nogz` 的 NoGZ ATF 补丁后是 `0x3c9` = EL2h").
The 12.5 preloader achieves the same thing without patching anything: it is the
firmware-side switch that turns the payload into an EL2 payload.  So the KVM
result doubles as proof that the board can be booted at EL2 with an unmodified,
signed vendor preloader.
