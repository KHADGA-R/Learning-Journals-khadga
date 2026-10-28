# Lesson 7 of 12 — Synchronization, Interrupts, and Deferred Work

**Course:** Linux Kernel Programming & Yocto Project  
**Prepared:** 2026-10-28  
**Environment:** Ubuntu Linux (use a disposable VM for kernel module experiments)  
**Status:** Study material prepared; exercises and builds not yet completed

---

## Objectives
Understand concurrency, races, mutexes, spinlocks, atomics, interrupt context, wait queues, workqueues, and timer-related design.

## 1. Concurrency and races
Multiple CPUs, processes, and interrupt handlers may access shared kernel data. A **race condition** occurs when correctness depends on uncontrolled execution order. A read-modify-write such as `counter++` is not generally atomic.

- **Mutex:** sleeping lock; appropriate for process context, never for hard IRQ context.
- **Spinlock:** busy-wait lock; suitable for short critical sections, including certain atomic contexts.
- **Atomic variables:** operations such as `atomic_inc` for simple counters.
- **Wait queue:** blocks a task until a condition becomes true.
- **Workqueue:** defers work to kernel worker context where sleeping is allowed.
- **RCU:** read-mostly synchronization technique, advanced topic.

## 2. Example: mutex-protected shared state
```c
#include <linux/mutex.h>

static DEFINE_MUTEX(counter_lock);
static unsigned int counter;

static void increment_counter(void)
{
    // Mutex serializes updates in sleepable process context.
    mutex_lock(&counter_lock);
    counter++;
    mutex_unlock(&counter_lock);
}
```
This helper is a **concept snippet**, not a standalone module. A mutex may sleep; do not call it in hard IRQ or other atomic context.

## 3. Example: atomic counter
```c
#include <linux/atomic.h>

static atomic_t events = ATOMIC_INIT(0);

static void note_event(void)
{
    // Atomic increment avoids lost updates for this simple counter.
    atomic_inc(&events);
}
```
Atomic operations do not automatically protect larger multi-field invariants.

## 4. Example: deferred work module
```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/workqueue.h>

static void lesson_work_fn(struct work_struct *work)
{
    // Runs in worker process context, not hard IRQ context.
    pr_info("lesson_work: deferred task executed\n");
}

static DECLARE_WORK(lesson_work, lesson_work_fn);

static int __init lesson_init(void)
{
    // Queue a one-shot task on the system workqueue.
    schedule_work(&lesson_work);
    return 0;
}

static void __exit lesson_exit(void)
{
    // Wait for any pending/running work before module memory disappears.
    cancel_work_sync(&lesson_work);
}

module_init(lesson_init);
module_exit(lesson_exit);
MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Workqueue lifecycle demonstration");
```
Build as `lesson_work.o` with the Lesson 5 external-module Makefile pattern. `cancel_work_sync` is crucial before unloading because a queued callback must never execute from freed module code.

## 5. Interrupt fundamentals
Hardware interrupts notify the CPU about events. An interrupt handler must be brief, must not sleep, and should acknowledge the hardware as required. Depending on device architecture, defer slow work to a threaded IRQ or workqueue. `request_irq` associates a handler with an IRQ; `free_irq` releases it. Do **not** request arbitrary IRQ numbers on a laptop.

Conceptual signature:
```c
// Illustration only: needs a real device, IRQ, and cleanup design.
static irqreturn_t example_irq(int irq, void *dev_id)
{
    /* Check/acknowledge device interrupt; schedule follow-up work. */
    return IRQ_HANDLED;
}
```

## 6. Choosing synchronization
| Situation | Typical tool |
|---|---|
| Short update in process context | Mutex |
| Short shared state update in atomic context | Spinlock |
| Independent numeric counter | Atomic variable |
| Wait until data is ready | Wait queue |
| Slow task triggered by IRQ | Threaded IRQ or workqueue |

Always identify whether the current context may sleep before selecting an API. Protect every access to shared state consistently.

## 7. Practical commands
```bash
cat /proc/interrupts
cat /proc/softirqs
ps -e -o pid,comm | grep -E 'kworker|irq/'
sudo journalctl -k -n 40
```
Output depends on the kernel and system.

## 8. Exercises
- [ ] Explain why `counter++` can lose updates.
- [ ] Compare mutex vs spinlock behavior.
- [ ] Build the workqueue example in a VM and inspect logs.
- [ ] Explain why `cancel_work_sync` belongs in cleanup.
- [ ] Identify which operations may sleep and which must not.

## 9. Knowledge check
1. Can a mutex be taken in a hard IRQ handler?
2. Does an atomic counter protect multiple related variables?
3. Why defer work from an interrupt?
4. What happens if a module unloads while its callback is queued?

**Next:** Lesson 8 — Device Model, Device Tree, and Hardware Interfaces.
**References:** https://docs.kernel.org/locking/index.html ; https://docs.kernel.org/core-api/workqueue.html ; https://docs.kernel.org/core-api/genericirq.html
