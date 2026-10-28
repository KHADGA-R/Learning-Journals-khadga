# Lesson 6 of 12 — Character Device Drivers and User–Kernel Interfaces

**Course:** Linux Kernel Programming & Yocto Project  
**Prepared:** 2026-10-28  
**Environment:** Ubuntu Linux (use a disposable VM for kernel module experiments)  
**Status:** Study material prepared; exercises and builds not yet completed

---

## Objectives
Understand device numbers, `/dev`, `struct file_operations`, `open/read/write/release`, user-memory boundaries, and safe device cleanup.

## 1. Concepts
A character device transfers a stream of bytes through a device node such as `/dev/example`. The **major** number identifies a driver; the **minor** identifies a device instance. The VFS dispatches file operations through `struct file_operations`. `copy_to_user` and `copy_from_user` handle data crossing the kernel/user boundary; user pointers must not be dereferenced directly.

For a first driver, a **misc device** avoids manual `alloc_chrdev_region`, `cdev_add`, and device-class setup. Learn those APIs after the working example.

## 2. Example: safe read-only misc device
Create `lesson_char.c`:
```c
#include <linux/fs.h>
#include <linux/miscdevice.h>
#include <linux/module.h>
#include <linux/uaccess.h>

static const char message[] = "Hello from kernel device\n";

static ssize_t lesson_read(struct file *file, char __user *buf,
                           size_t count, loff_t *pos)
{
    // simple_read_from_buffer respects *pos, count, and the user boundary.
    return simple_read_from_buffer(buf, count, pos,
                                   message, sizeof(message) - 1);
}

static const struct file_operations lesson_fops = {
    .owner = THIS_MODULE,  // Keep the module loaded during active operations.
    .read = lesson_read,
    .llseek = no_llseek,
};

static struct miscdevice lesson_device = {
    .minor = MISC_DYNAMIC_MINOR,
    .name = "lesson_char",
    .fops = &lesson_fops,
    .mode = 0444,           // Read-only device node permissions.
};

static int __init lesson_init(void)
{
    // Registration creates a device in the misc subsystem.
    return misc_register(&lesson_device);
}

static void __exit lesson_exit(void)
{
    // Undo registration so new opens cannot occur.
    misc_deregister(&lesson_device);
}

module_init(lesson_init);
module_exit(lesson_exit);
MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Read-only misc character device");
```

`simple_read_from_buffer` handles file offsets and copies bytes to userspace, avoiding a common endless-`cat` bug. `struct file_operations` callbacks run in kernel context and must obey error-code and locking conventions.

## 3. Makefile
```makefile
obj-m += lesson_char.o
KDIR := /lib/modules/$(shell uname -r)/build
PWD := $(shell pwd)
all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules
clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

## 4. Build and test in a disposable VM
```bash
sudo apt install -y build-essential linux-headers-$(uname -r)
make
sudo insmod ./lesson_char.ko
ls -l /dev/lesson_char
cat /dev/lesson_char
sudo rmmod lesson_char
make clean
```
If udev has not yet created the node, inspect `/sys/class/misc/lesson_char/dev`; do not guess major/minor numbers.

## 5. Userspace test: device_read.c
```c
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>

int main(void)
{
    char buf[128];
    int fd = open("/dev/lesson_char", O_RDONLY);
    if (fd < 0) { perror("open"); return 1; }

    // read() invokes the driver's .read callback via the VFS.
    ssize_t n = read(fd, buf, sizeof(buf));
    if (n < 0) { perror("read"); close(fd); return 1; }
    if (n > 0 && write(STDOUT_FILENO, buf, (size_t)n) != n)
        perror("write");
    close(fd);
    return 0;
}
```
Compile: `gcc -Wall -Wextra -o device_read device_read.c`. For production code, handle partial `write` results with a loop.

## 6. Deeper concepts
- Traditional character driver: `alloc_chrdev_region` → `cdev_init` → `cdev_add` → device registration, with reverse-order cleanup.
- `copy_to_user` / `copy_from_user` return the number of **uncopied** bytes, not a negative errno.
- `ioctl` supports structured control commands; use stable UAPI definitions and validate sizes.
- `poll` and blocking I/O can notify userspace of readiness.
- Concurrent opens may require mutexes or other synchronization.

## 7. Exercises
- [ ] Explain major and minor device numbers.
- [ ] Build and read the device in a VM.
- [ ] Use `strace` on `device_read`.
- [ ] Design a writable version on paper: size limits, locking, and input validation.
- [ ] Compare misc devices with manually registered `cdev` devices.

## 8. Knowledge check
1. Why can't a driver directly dereference a userspace pointer?
2. What does `simple_read_from_buffer` do with the file offset?
3. Why is `.owner = THIS_MODULE` useful?
4. Which cleanup function pairs with `misc_register`?

**Next:** Lesson 7 — Synchronization, Interrupts, and Deferred Work.
**References:** https://docs.kernel.org/driver-api/misc_devices.html ; https://docs.kernel.org/core-api/uaccess.html
