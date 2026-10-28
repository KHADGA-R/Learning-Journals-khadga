# Learning Journal — Linux Kernel Programming & Yocto Project

**Date:** 2026-10-28  
**Lesson:** 3 of 12  
**Topic:** Linux Processes, System Calls, and Memory Management  
**Environment:** Ubuntu Linux  
**Status:** Lesson covered; practical exercises pending verification

---

## 1. Overview

In Lesson 3, I studied how Linux creates and manages processes, how user-space programs communicate with the Linux kernel through system calls, and how virtual memory works.

This builds on the Linux fundamentals and C programming concepts covered in Lessons 1 and 2.

## 2. Concepts Learned

### Linux Processes

A **program** is an executable file on disk; a **process** is a running instance of a program.

- **PID:** Process identifier.
- **PPID:** Parent process identifier.
- **Process states:** Running, runnable, sleeping, stopped, and zombie.
- **Scheduling:** The kernel selects runnable tasks to execute on CPUs.
- **Isolation:** Processes normally have separate virtual address spaces.

```bash
ps
ps aux
echo $$
top
```

### Process Creation with `fork()`

`fork()` creates a child process. The child receives `0`, the parent receives the child's PID, and failure returns `-1`.

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>

int main(void)
{
    pid_t pid = fork();

    if (pid < 0) {
        perror("fork failed");
        return 1;
    }
    if (pid == 0) {
        // Child branch.
        printf("Child PID: %ld\n", (long)getpid());
        return 0;
    }

    // Parent reaps its child to avoid a zombie.
    printf("Parent PID: %ld\n", (long)getpid());
    if (waitpid(pid, NULL, 0) == -1) {
        perror("waitpid failed");
        return 1;
    }
    return 0;
}
```

Compile with `gcc -Wall -Wextra -o fork_example fork_example.c`, then run `./fork_example`. Parent/child output order may vary.

### Program Execution with `exec()`

The `exec` family replaces the current process's program image. A successful call does not create a new process or return to the old program.

```c
execlp("ls", "ls", "-l", (char *)NULL);

// Reached only if execlp() fails.
perror("execlp failed");
```

A typical shell pattern is `fork()` to create a child, `exec()` in the child to run a program, and `waitpid()` in the parent.

### Linux System Calls

System calls are controlled entry points into the kernel.

| Call | Purpose |
|---|---|
| `open()` | Open a file |
| `read()` | Read bytes |
| `write()` | Write bytes |
| `close()` | Close a file descriptor |
| `fork()` | Create a child process |
| `execve()` | Execute a program |
| `waitpid()` | Wait for a child |
| `mmap()` | Create a memory mapping |

The C library provides wrappers for many system calls; functions like `printf()` are not themselves system calls.

### Virtual Memory

Virtual memory enables address translation, memory protection, and process isolation.

- **Text:** Executable code.
- **Data/BSS:** Initialized and zero-initialized globals/statics.
- **Heap:** Dynamic allocations.
- **Stack:** Function call frames and local variables.
- **Mappings:** Shared libraries and other mapped regions.

Actual layouts vary by system and Address Space Layout Randomization (ASLR).

## 3. Code Examples

### Example 1 — Process Information

**File:** `process_info.c`

```c
#include <stdio.h>
#include <unistd.h>

int main(void)
{
    printf("Process ID: %ld\n", (long)getpid());
    printf("Parent Process ID: %ld\n", (long)getppid());
    return 0;
}
```

```bash
gcc -Wall -Wextra -o process_info process_info.c
./process_info
```

**Takeaway:** `getpid()` identifies the running process; `getppid()` identifies its parent.

### Example 2 — File Operations Using System Calls

**File:** `file_read.c`

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>

int main(void)
{
    char buffer[128];
    int fd = open("sample.txt", O_RDONLY);

    if (fd == -1) {
        perror("open failed");
        return 1;
    }

    // Read from the file descriptor into a bounded buffer.
    ssize_t bytes = read(fd, buffer, sizeof(buffer));
    if (bytes < 0) {
        perror("read failed");
        close(fd);
        return 1;
    }

    if (bytes > 0 &&
        write(STDOUT_FILENO, buffer, (size_t)bytes) != bytes) {
        perror("write failed or incomplete");
        close(fd);
        return 1;
    }

    if (close(fd) == -1) {
        perror("close failed");
        return 1;
    }
    return 0;
}
```

```bash
echo "Learning Linux Kernel Programming" > sample.txt
gcc -Wall -Wextra -o file_read file_read.c
./file_read
```

Expected output:

```text
Learning Linux Kernel Programming
```

**Takeaway:** File descriptors connect user programs to kernel-managed resources. Production programs should loop to handle partial reads and writes.

### Example 3 — Memory Addresses

**File:** `memory_example.c`

```c
#include <stdio.h>
#include <stdlib.h>

int global_value = 100;

int main(void)
{
    int local_value = 200;
    int *heap_value = malloc(sizeof(*heap_value));
    if (heap_value == NULL) {
        perror("malloc failed");
        return 1;
    }

    *heap_value = 300;
    printf("Global: %p\n", (void *)&global_value);
    printf("Local:  %p\n", (void *)&local_value);
    printf("Heap:   %p\n", (void *)heap_value);

    free(heap_value);
    return 0;
}
```

```bash
gcc -Wall -Wextra -o memory_example memory_example.c
./memory_example
```

**Takeaway:** Different storage classes typically occupy different virtual memory regions; ASLR can change addresses between runs.

## 4. Practical Tools

### `strace`

Inspect the system calls a process makes:

```bash
sudo apt install strace
strace ./file_read
strace -e trace=openat,read,write,close ./file_read
strace -f ./fork_example
```

A C `open()` wrapper may invoke the `openat` system call on Linux.

### `/proc` Filesystem

Inspect process memory mappings:

```bash
cat /proc/self/maps
```

## 5. Connection to Kernel Development

- `task_struct` represents kernel tasks.
- `mm_struct` is involved in process memory management.
- The scheduler selects tasks for CPU execution.
- Device drivers can expose file operations to user programs.
- System calls provide a controlled route from user space into kernel services.

## 6. Practical Takeaways

1. A process is a running instance of a program.
2. `fork()` creates a child; `exec()` replaces the running program.
3. System calls connect user applications to kernel functionality.
4. File descriptors represent access to kernel-managed resources.
5. Virtual memory provides translation, protection, and isolation.
6. `strace` and `/proc` are useful debugging and inspection tools.
7. Process and memory concepts underpin kernel modules and device drivers.

## 7. Exercises and Next Steps

- [ ] Compile and run `process_info.c`.
- [ ] Practice `fork()` and `waitpid()`.
- [ ] Modify an `exec()` example to run `pwd`.
- [ ] Trace file operations with `strace`.
- [ ] Compare memory addresses across executions.
- [ ] Inspect mappings using `/proc/PID/maps`.
- [ ] Start Lesson 4: Linux Kernel Source Code and Compilation.
- [ ] Explore the kernel source tree, configuration, and Kbuild.

## 8. References

- [Linux Kernel Documentation](https://docs.kernel.org/)
- [fork(2)](https://man7.org/linux/man-pages/man2/fork.2.html)
- [exec(3)](https://man7.org/linux/man-pages/man3/exec.3.html)
- [read(2)](https://man7.org/linux/man-pages/man2/read.2.html)
- [/proc/PID/maps](https://man7.org/linux/man-pages/man5/proc_pid_maps.5.html)
- [Yocto Project Documentation](https://docs.yoctoproject.org/)

---

**Progress:** Lesson 3 of 12 covered; hands-on exercises not yet confirmed complete.

**Next lesson:** Linux Kernel Source Code and Compilation.
