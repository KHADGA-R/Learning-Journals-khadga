# Lesson 5 of 12 — Writing Your First Linux Kernel Module

**Course:** Linux Kernel Programming & Yocto Project  
**Prepared:** 2026-10-28  
**Environment:** Ubuntu Linux (use a disposable VM for kernel module experiments)  
**Status:** Study material prepared; exercises and builds not yet completed

---

## Objectives
Understand loadable kernel modules (LKMs), kernel vs user-space APIs, module lifecycle, Kbuild, loading/unloading, kernel logs, and common build errors.

## 1. Concepts
A loadable kernel module is compiled code that can be inserted into a running kernel without rebuilding the whole kernel. Modules execute with kernel privileges: a bug can crash or compromise the system. Practice in a disposable Ubuntu VM with a snapshot. Never test untrusted modules on a production host.

- `module_init()` registers the function invoked at module insertion.
- `module_exit()` registers cleanup for module removal.
- `pr_info()` writes to the kernel log, not to standard output.
- `MODULE_LICENSE("GPL")` declares the module's license; it does not itself grant a license.
- `MODULE_AUTHOR` and `MODULE_DESCRIPTION` provide metadata.
- `insmod` inserts a specific .ko file; `modprobe` resolves dependencies by module name.
- `rmmod` removes a module, if it is safe to unload.

## 2. Prerequisites
```bash
uname -r
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r) kmod
test -d /lib/modules/$(uname -r)/build && echo "Headers available"
mkdir -p ~/kernel-learning/lesson05 && cd ~/kernel-learning/lesson05
```
If headers are unavailable for the running kernel, use a supported Ubuntu kernel/VM; do not compile against arbitrary mismatched headers.

## 3. Complete example: hello_kernel.c
```c
#include <linux/init.h>
#include <linux/kernel.h>
#include <linux/module.h>

static int __init hello_init(void)
{
    // Runs when the module loads; return 0 to accept insertion.
    pr_info("hello_kernel: loaded\n");
    return 0;
}

static void __exit hello_exit(void)
{
    // Release resources here before the module is unloaded.
    pr_info("hello_kernel: unloaded\n");
}

module_init(hello_init);
module_exit(hello_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Learning Journal");
MODULE_DESCRIPTION("Minimal kernel module for learning lifecycle");
```

## 4. Kbuild Makefile
Save the following as `Makefile` (recipe lines must begin with a TAB):
```makefile
obj-m += hello_kernel.o

KDIR := /lib/modules/$(shell uname -r)/build
PWD := $(shell pwd)

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```
`obj-m` requests an external module. `-C` enters the running kernel's build tree; `M=` tells Kbuild where the external source lives.

## 5. Build and inspect
```bash
make
file hello_kernel.ko
modinfo ./hello_kernel.ko
sudo insmod ./hello_kernel.ko
lsmod | grep hello_kernel
sudo dmesg | tail -n 20
sudo rmmod hello_kernel
sudo dmesg | tail -n 20
make clean
```
Some Ubuntu configurations restrict `dmesg`; try `sudo journalctl -k -n 30`. Secure Boot lockdown can reject unsigned modules: prefer a disposable VM with a suitable test configuration, or follow your distribution's official module-signing guidance; do not disable protections on your main machine casually.

## 6. Debugging
- `Invalid module format`: check `uname -r` and `modinfo -F vermagic hello_kernel.ko`.
- `Required key not available`: module signature policy/Secure Boot.
- `Unknown symbol`: unresolved dependency or unavailable exported symbol.
- `Operation not permitted`: insufficient privileges or kernel security policy.
- `make: ... build: No such file`: missing matching headers.

## 7. Exercises
- [ ] Explain why kernel code uses `pr_info` instead of `printf`.
- [ ] Build the module in a VM and record `modinfo`.
- [ ] Confirm load and unload messages in kernel logs.
- [ ] Add `module_param` for a read-only integer setting and document permissions.
- [ ] Explain why all acquired resources must be released on failure/unload.

## 8. Knowledge check
1. What does `obj-m` mean?
2. Why must module headers match the running kernel?
3. What is the difference between `insmod` and `modprobe`?
4. Why is kernel module testing higher risk than ordinary C programs?

**Next:** Lesson 6 — Character Device Drivers and User–Kernel Interfaces.
**References:** https://docs.kernel.org/kbuild/modules.html ; https://docs.kernel.org/core-api/printk-basics.html
