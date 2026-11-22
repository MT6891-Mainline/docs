# op6893 development checkpoint

The original kernel-tree directory snapshotted the board bring-up reports previously maintained at
/data/src/mainline/mt6893-direct. The source tree is experimental, not an
upstream-ready port. Validated image06374ef46937 predates this commit; changing
Git HEAD does not establish that a newly built image was tested on hardware.

See [peripheral status](../PERIPHERAL-STATUS.md), [Wi-Fi/Bluetooth bring-up](../CONNECTIVITY-STATUS.md),
[hardware roadmap](HARDWARE-PLAN.md), and [binary-analysis handoff](WIFI-REVERSE-ENGINEERING.md). Earlier report links may refer to large external captures/tools that
remain in mt6893-direct; boot images, rootfs, proprietary firmware and device
private data are not included here. The external build/initramfs workspace is
not a Git repository and is not made reproducible by this kernel checkpoint.

For the current modem architecture, lifecycle, userspace tools and test limits,
see [MODEM-CONTROL-STACK.md](../MODEM-CONTROL-STACK.md).

Fourteen existing host tests passed at checkpoint time; output is recorded in
[commit-tests.log](evidence/attempt-50/commit-tests.log). No new hardware test occurred during
this checkpoint. Historical experiments and retained inherited-resource paths
must not be interpreted as full power management or thermal protection.

AI assistance: Codex, Devin and Claude assisted source changes, source
audits, test execution and documentation across the iterative board bring-up. Human hardware tests
are distinguished in the reports. No DCO certification is asserted.
