# Learning Journal — Linux Kernel Source Code and Compilation

**Date:** 2026-10-28  
**Course:** Linux Kernel Programming & Yocto Project  
**Lesson:** 4 of 12  
**Environment:** Ubuntu Linux  
**Status:** Concepts and commands studied; hands-on kernel compilation pending

---

## 1. Overview

In Lesson 4, I learned about the Linux kernel source tree, kernel configuration, Kbuild, and the compilation process.

I studied how Linux kernel source code is organized, how kernel features are selected, and how source files are compiled into kernel images and modules.

**Learning status:** I studied the concepts, code examples, and commands. I have not yet performed or verified a complete kernel build.

## 2. Key Concepts Learned

### Linux Kernel Source Structure

The Linux kernel source contains implementations of process management, memory management, filesystems, networking, hardware drivers, and architecture-specific functionality.

| Directory | Purpose |
|---|---|
| `arch/` | Architecture-specific code |
| `drivers/` | Device drivers |
| `fs/` | Filesystems |
| `include/` | Header files |
| `init/` | Kernel initialization |
| `kernel/` | Core kernel facilities |
| `mm/` | Memory management |
| `net/` | Networking |
| `scripts/` | Build scripts |
| `Documentation/` | Kernel documentation |

Important source files and directories:

- `kernel/fork.c` — Process creation.
- `kernel/sched/` — Scheduling.
- `mm/` — Memory management.
- `include/linux/sched.h` — Task-related declarations.
- `init/main.c` — Kernel initialization.

### Kernel Configuration: Kconfig and .config

Kconfig defines available kernel configuration options and their dependencies.

The `.config` file records the configuration selected for a particular kernel build.

Common configuration values:

- `y` — Build into the kernel.
- `m` — Build as a loadable module.
- `n` — Disable the feature.

Example:

```text
CONFIG_MODULES=y
CONFIG_EXT4_FS=y
CONFIG_USB_STORAGE=m
```

These are illustrative settings, not an actual configuration from my Ubuntu system.

### Kbuild

Kbuild is the Linux kernel's Makefile-based build system.

```makefile
obj-$(CONFIG_EXAMPLE_DRIVER) += example_driver.o
```

The object is built into the kernel or as a module depending on the corresponding configuration option.

For a standalone kernel module:

```makefile
obj-m += hello_kernel.o
```

This will be useful when learning to write Linux kernel modules.

### Kernel Compilation

The kernel compilation process generally involves:

1. Obtaining the kernel source.
2. Installing development dependencies.
3. Configuring kernel features.
4. Compiling source files.
5. Linking kernel components.
6. Producing a kernel image and optional modules.

Important build artifacts:

| Artifact | Description |
|---|---|
| `vmlinux` | Linked ELF kernel image |
| `bzImage` | Bootable compressed kernel image on x86 |
| `.ko` | Loadable kernel module |
| `.config` | Kernel build configuration |

### Kernel Boot Process

Studied the simplified boot sequence:

```text
Firmware (BIOS/UEFI)
       |
       v
Bootloader
       |
       v
Linux Kernel
       |
       v
Root Filesystem
       |
       v
Initial User-Space Process (PID 1)
       |
       v
System Services and Applications
```

Important kernel initialization functions include `start_kernel()`, `rest_init()`, and `kernel_init()`.

## 3. Commands Studied

### Inspect the Running Kernel

```bash
uname -r
uname -m
uname -a

ls /boot
ls /lib/modules/$(uname -r)
cat /proc/version
```

### Install Kernel Development Tools

```bash
sudo apt update

sudo apt install -y \
    build-essential \
    git \
    bc \
    bison \
    flex \
    libssl-dev \
    libelf-dev \
    libncurses-dev \
    dwarves \
    cpio \
    rsync \
    xz-utils \
    wget
```

### Download and Explore Kernel Source

```bash
mkdir -p ~/kernel-learning
cd ~/kernel-learning

sudo apt install linux-source

ls /usr/src/linux-source-*.tar.*
```

The source archive can be extracted into the learning workspace.

After extraction, important directories to explore include:

```bash
ls drivers
ls kernel
ls mm
ls arch

less kernel/fork.c
```

### Kernel Configuration Commands

```bash
make defconfig
make menuconfig
make olddefconfig
make help
```

**Important:** `make menuconfig` provides an interactive interface for exploring kernel configuration options.

### Kernel Build Commands

For a clean, separate build directory:

```bash
mkdir -p ~/kernel-learning/kernel-build
```

From the kernel source directory, after ensuring the source tree is clean:

```bash
make O=../kernel-build defconfig
make O=../kernel-build -j2
```

The `O=` option places generated build files in a separate directory.

The `-j2` option allows two parallel build jobs.

**Status:** These commands were studied but have not yet been executed and verified as a successful kernel build.

## 4. Code and Build Examples

### Simple User-Space Makefile

```makefile
CC = gcc

hello: hello.c
	$(CC) -o hello hello.c
```

This example demonstrates basic Makefile compilation.

### Kernel Module Build Rule

```makefile
obj-m += hello_kernel.o
```

This Kbuild rule selects `hello_kernel.o` for compilation as an external loadable kernel module.

### Configuration-Dependent Build Rule

```makefile
obj-$(CONFIG_EXAMPLE_DRIVER) += example_driver.o
```

This demonstrates how Kbuild uses configuration values to determine which components to compile.

## 5. Connection to the Yocto Project

Manual kernel compilation and Yocto share important concepts.

| Manual Kernel Build | Yocto Project |
|---|---|
| Obtain kernel source | Fetch source through recipes |
| Configure kernel | Apply configuration and metadata |
| Compile with Make | Execute BitBake build tasks |
| Generate kernel image | Produce deployable build artifacts |

Yocto automates many steps involved in building customized embedded Linux systems.

## 6. Practical Takeaways

1. The Linux kernel source tree separates major subsystems into organized directories.
2. Kconfig controls which kernel features are available and selected.
3. The `.config` file stores build configuration choices.
4. Kbuild uses Makefiles and configuration values to build kernel components.
5. Kernel images and modules are different build artifacts.
6. Building a kernel does not require installing it on the running Ubuntu system.
7. Yocto automates kernel configuration and compilation as part of embedded Linux image creation.

## 7. Next Steps

- [ ] Install the kernel development dependencies on Ubuntu.
- [ ] Download and extract Linux kernel source.
- [ ] Explore `kernel/`, `drivers/`, `mm/`, and `init/`.
- [ ] Run `make menuconfig` and inspect kernel options.
- [ ] Inspect real Kbuild Makefiles.
- [ ] Compile a kernel in a separate output directory.
- [ ] Verify the generated `vmlinux` and architecture-specific kernel image.
- [ ] Begin Lesson 5: Writing My First Linux Kernel Module.
- [ ] Learn `module_init()`, `module_exit()`, `insmod`, `rmmod`, and `dmesg`.

## 8. References

- [Linux Kernel Documentation](https://docs.kernel.org/)
- [Linux Kernel Build System](https://docs.kernel.org/kbuild/index.html)
- [Kconfig Documentation](https://docs.kernel.org/kbuild/kconfig.html)
- [Building External Modules](https://docs.kernel.org/kbuild/modules.html)
- [Yocto Project Linux Kernel Development Manual](https://docs.yoctoproject.org/kernel-dev/)

---

**Course progress:** Lessons 1–4 of 12 studied.

**Practical status:** Lesson 4 kernel build pending.

**Next topic:** Lesson 5 — Writing My First Linux Kernel Module.
