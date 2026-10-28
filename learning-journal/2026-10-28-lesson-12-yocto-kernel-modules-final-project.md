# Lesson 12 of 12 — Integrating Kernel Modules with Yocto, Debugging, and Final Project

**Course:** Linux Kernel Programming & Yocto Project  
**Prepared:** 2026-10-28  
**Environment:** Ubuntu Linux / QEMU  
**Status:** Study material prepared; exercises and builds not yet completed

---

## Objectives
Integrate an external kernel module into Yocto, package it into an image, debug build/runtime problems, understand reproducibility and maintenance, and complete a capstone.

## 1. Integration architecture
```text
meta-learning/
  recipes-kernel/
    hello-kmod/
      hello-kmod_1.0.bb
      files/
        hello_kmod.c
        Makefile
```
The kernel module is built against the **Yocto target kernel**, not the Ubuntu host kernel. Never reuse host `/lib/modules/$(uname -r)/build` inside a Yocto module recipe.

## 2. Minimal kernel module
Create `files/hello_kmod.c`:
```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>

static int __init hello_kmod_init(void)
{
    // Confirm module initialization in the target kernel log.
    pr_info("hello_kmod: initialized\n");
    return 0;
}

static void __exit hello_kmod_exit(void)
{
    // Runs when the module is safely removed.
    pr_info("hello_kmod: removed\n");
}

module_init(hello_kmod_init);
module_exit(hello_kmod_exit);
MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Yocto external module learning example");
```

Create `files/Makefile`:
```makefile
obj-m += hello_kmod.o
```
The `module` class handles invoking the target kernel's build system; this Makefile specifies the module object.

## 3. Yocto recipe
Create `hello-kmod_1.0.bb`:
```bitbake
SUMMARY = "Example external kernel module"
LICENSE = "GPL-2.0-only"
LIC_FILES_CHKSUM = "file://hello_kmod.c;beginline=1;endline=1;md5=REPLACE_WITH_REAL_MD5"

inherit module

SRC_URI = "file://hello_kmod.c file://Makefile"
S = "${WORKDIR}"
```
**Before building:** the license checksum above is deliberately a placeholder, not a runnable value. Add an SPDX license header as the first line of `hello_kmod.c`:
```c
// SPDX-License-Identifier: GPL-2.0-only
```
Then compute the first-line MD5 using `head -n 1 files/hello_kmod.c | md5sum` and replace `REPLACE_WITH_REAL_MD5`. Verify `S` against your Yocto release (`WORKDIR` vs `UNPACKDIR` for unpacked sources). The exact output package name should be checked after the build.

## 4. Build and include
From the initialized Yocto environment:
```bash
bitbake hello-kmod
bitbake -e hello-kmod | grep '^KERNEL_VERSION='
```
External kernel modules are typically packaged under a `kernel-module-...` package name. Inspect the generated package names before editing the image:
```bash
bitbake -e hello-kmod | grep '^PACKAGES='
```
If the package is named `kernel-module-hello-kmod`, add:
```conf
IMAGE_INSTALL:append = " kernel-module-hello-kmod"
```
to `conf/local.conf`, then rebuild `core-image-minimal`. Do not assume the package name without checking.

## 5. Runtime validation in QEMU
```sh
uname -r
find /lib/modules -name 'hello_kmod.ko*'
modprobe hello_kmod
dmesg | tail -n 20
lsmod | grep hello_kmod
modprobe -r hello_kmod
```
Depending on image configuration, `modprobe` and related tools may not be installed by default; add `kmod` to the image if needed. Some kernel/module configurations may prevent unloading.

## 6. Debugging and reproducibility
- **Task logs:** `tmp/work/.../temp/log.do_compile*` and `log.do_install*`.
- **Environment:** `bitbake -e <recipe>` reveals expanded variables.
- **Kernel ABI:** module `vermagic` must match the target kernel build.
- **License compliance:** declare correct licenses, checksums, and notices.
- **Source reproducibility:** pin Git revisions with `SRCREV`; avoid floating branches.
- **Layer compatibility:** use `LAYERSERIES_COMPAT` and test supported release combinations.
- **Security:** minimize image packages, protect signing keys, and keep software updated.
- **SBOM and supply chain:** explore Yocto's SPDX/SBOM generation features.
- **Debug workflow:** isolate whether failure is fetch, configure, compile, package, deploy, or runtime.

## 7. Final capstone project
**Goal:** Build a custom QEMU embedded Linux image that includes both a C application and a kernel module.

Acceptance checklist:
- [ ] Document host OS, Poky release, target MACHINE, and layer revisions.
- [ ] Build `core-image-minimal` for QEMU.
- [ ] Add `meta-learning` and custom `hello` application recipe.
- [ ] Build and package `hello-kmod` for the target kernel.
- [ ] Include both packages in the image.
- [ ] Boot QEMU and execute `hello-learning`.
- [ ] Load/unload the module and capture kernel log messages.
- [ ] Record build commands, output locations, and troubleshooting notes.
- [ ] Explain the separation between host compiler, target compiler, and target kernel headers.
- [ ] Commit reproducible source/recipes and a README; exclude large build outputs, caches, and secrets.

## 8. Further learning
Explore kernel debugging (`dynamic_debug`, ftrace, perf), device-tree overlays, custom BSP layers, systemd service recipes, SDK/eSDK, CI, secure boot, kernel signing, patch management, and Yocto release upgrades.

## 9. Knowledge check
1. Why must an external module target Yocto's kernel rather than Ubuntu's host kernel?
2. What does `inherit module` provide?
3. How do you verify a generated module package name?
4. What evidence proves the final image works?

**Course completion is not claimed:** lessons 5–12 are prepared study guides; complete the exercises and capstone to verify practical mastery.
**References:** https://docs.yoctoproject.org/kernel-dev/common.html ; https://docs.yoctoproject.org/ref-manual/classes.html ; https://docs.yoctoproject.org/dev-manual/debugging.html
