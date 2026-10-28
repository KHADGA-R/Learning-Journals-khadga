# Learning Journal — Linux Kernel Programming & Yocto Project

**Date:** 2026-10-28  
**Environment:** Ubuntu Linux  
**Level:** Beginner  
**Course progress:** Lessons 1–2 of 12 (topics covered; exercises not yet verified)

## Overview

Studied Linux architecture and the C programming foundations needed for Linux kernel modules, device drivers, and embedded Linux development. Introduced the Yocto Project as a way to build customized embedded Linux images.

## Lesson 1 — Linux Fundamentals

### Concepts
- **Linux architecture:** User applications communicate with the kernel through system calls; the kernel manages hardware, processes, memory, drivers, and filesystems.
- **User space vs. kernel space:** Kernel code has elevated privileges and bugs can affect the entire system.
- **System calls:** Introduced `read()`, `write()`, `fork()`, and `execve()`.
- **Kernel modules:** Extend a running kernel without rebuilding the entire kernel.
- **Virtual filesystems and devices:** Explored `/proc` and `/dev`.
- **Yocto introduction:** Poky, BitBake, recipes, layers, BSPs, and custom images.

### Useful Ubuntu commands

```bash
uname -r               # Running kernel release
uname -a               # Kernel and system details
lsmod                  # Loaded kernel modules
sudo dmesg | tail -20  # Recent kernel messages
cat /proc/version      # Kernel build information
cat /proc/meminfo      # Memory information
free -h                # Human-readable memory summary
```

### First C program

```c
#include <stdio.h>

int main(void)
{
    printf("Hello Embedded Linux!\n");
    return 0;
}
```

```bash
gcc -Wall -Wextra -o hello hello.c
./hello
```

**Takeaway:** Ordinary C applications run in user space and request kernel services through system calls, often via library wrappers.

## Lesson 2 — C Programming for Kernel Development

### Pointers
A pointer stores an address; dereferencing accesses the pointed-to object.

```c
int number = 10;
int *ptr = &number;
*ptr = 50;  // Changes number
```

### Structures
Structures group related device state; `->` accesses members through a structure pointer.

```c
struct device {
    int id;
    int status;
};
struct device led = {1, 0};
struct device *ptr = &led;
ptr->status = 1;
```

### Memory
Compared stack and heap storage and studied allocation and cleanup.

```c
#include <stdlib.h>

int *ptr = malloc(sizeof(*ptr));
if (ptr != NULL) {
    *ptr = 100;
    free(ptr);
}
```

In kernel code, memory APIs include `kmalloc()` and `kfree()`, not user-space `malloc()` and `free()`.

### Bitwise operations
Useful for manipulating hardware control and status bits.

```c
unsigned int reg = 0;
reg |= (1U << 3);   // Set bit 3
reg &= ~(1U << 3);  // Clear bit 3
reg ^= (1U << 3);   // Toggle bit 3
```

### Makefiles
Learned to automate GCC compilation. Recipe command lines must begin with tabs.

```makefile
CC = gcc
CFLAGS = -Wall -Wextra

all: hello

hello: hello.c
	$(CC) $(CFLAGS) -o hello hello.c

clean:
	rm -f hello

.PHONY: all clean
```

### Mini-project — Simulated LED controller
Studied a user-space LED simulation using a device structure, pointer-based functions, and bit masks for on/off state. It models driver logic but does **not** access real hardware.

## Practical Takeaways
1. Pointers, structures, and memory safety are essential for device drivers.
2. Kernel programming uses kernel APIs and has different failure consequences from user-space programming.
3. Bit masks let programs manipulate individual register bits.
4. Makefiles prepare the way for Linux Kbuild.
5. Yocto builds customized embedded Linux distributions.

## Next Steps
- [ ] Complete and verify Lesson 2 coding exercises.
- [ ] Practice GDB and AddressSanitizer.
- [ ] Lesson 3: processes, system calls, and virtual memory.
- [ ] Write programs using `open()`, `read()`, `write()`, `fork()`, and `exec()`.
- [ ] Trace system calls with `strace`.
- [ ] Progress to kernel modules, device drivers, and Yocto image creation.

## References
- [Linux Kernel Documentation](https://docs.kernel.org/)
- [Linux Kernel Coding Style](https://docs.kernel.org/process/coding-style.html)
- [Kernel-doc Format](https://docs.kernel.org/doc-guide/kernel-doc.html)
- [Yocto Project Documentation](https://docs.yoctoproject.org/)

**Documentation preference:** Explain and comment important code sections, not every line.
