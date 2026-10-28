# Lesson 9 of 12 — Embedded Linux Boot Process and Yocto Foundations

**Course:** Linux Kernel Programming & Yocto Project  
**Prepared:** 2026-10-28  
**Environment:** Ubuntu Linux / QEMU  
**Status:** Study material prepared; exercises and builds not yet completed

---

## Objectives
Explain firmware/ROM, bootloaders, U-Boot, Linux image, Device Tree, initramfs, root filesystem, PID 1, and how Yocto builds embedded Linux components.

## 1. Embedded Linux boot chain
Typical ARM board flow: SoC boot ROM → first-stage loader/SPL → U-Boot → Linux image + DTB (+ optional initramfs) → mount root filesystem → start PID 1 (often systemd). The exact sequence depends on the board and firmware.

- **SPL:** small early bootloader for constrained memory.
- **U-Boot:** bootloader that can load kernel/DTB and pass kernel command-line arguments.
- **DTB:** compiled Device Tree describing hardware.
- **initramfs:** temporary early userspace, sometimes used to locate/mount the real root.
- **rootfs:** persistent userspace containing libraries, binaries, and configuration.
- **PID 1:** initial userspace process responsible for service startup and process reaping.

## 2. Inspect an Ubuntu boot
```bash
uname -a
cat /proc/cmdline
cat /proc/1/comm
findmnt /
ls -lh /boot
journalctl -b -k --no-pager | head -40
```
Your Ubuntu PC may use UEFI and GRUB rather than U-Boot; the concepts still transfer.

## 3. Yocto vocabulary
- **Poky:** reference Yocto distribution combining BitBake, metadata, and example configuration.
- **BitBake:** task scheduler/build engine.
- **Recipe (.bb):** instructions to fetch, configure, compile, install, and package software.
- **Layer:** collection of related metadata, recipes, and configuration.
- **BSP layer:** board support metadata for a machine.
- **MACHINE:** target board/emulated machine selection.
- **DISTRO:** distribution policies and defaults.
- **Image recipe:** selects packages/features to assemble a root filesystem and deployable artifacts.
- **SDK:** cross-development toolchain and target libraries.
- **sstate cache:** reusable task outputs.

## 4. Task flow
A simplified recipe flow: `do_fetch` → `do_unpack` → `do_patch` → `do_configure` → `do_compile` → `do_install` → packaging and image assembly. Actual dependencies include additional tasks.

## 5. Example: illustrative boot configuration
```text
console=ttyS0,115200 root=/dev/mmcblk0p2 rootwait
```
This is **not** a universal boot argument. Serial console names, root devices, and partition layouts are board-specific; wrong arguments can prevent booting.

## 6. Inspect a Device Tree when available
```bash
ls /sys/firmware/devicetree/base 2>/dev/null || echo "No live DT exposed"
command -v dtc || echo "Install device-tree-compiler if needed"
```
Device Tree is common on embedded ARM; a standard x86 Ubuntu machine may not expose it.

## 7. Practical exercises
- [ ] Draw the boot flow for an ARM board and compare with Ubuntu PC boot.
- [ ] Explain kernel image vs DTB vs rootfs.
- [ ] Identify PID 1 on your machine.
- [ ] Define Poky, BitBake, recipe, layer, BSP, MACHINE, and DISTRO.
- [ ] Trace a package from recipe tasks to the final image.

## 8. Knowledge check
1. What does the bootloader pass to Linux?
2. What is the role of PID 1?
3. How does a Yocto image differ from the Linux kernel?
4. Why is MACHINE configuration important?

**Next:** Lesson 10 — Building Your First Yocto Image with QEMU.
**References:** https://docs.yoctoproject.org/overview-manual/concepts.html ; https://docs.u-boot.org/en/latest/ ; https://docs.kernel.org/admin-guide/init.html
