# Lesson 8 of 12 — Linux Device Model, Device Tree, and Hardware Interfaces

**Course:** Linux Kernel Programming & Yocto Project  
**Prepared:** 2026-10-28  
**Environment:** Ubuntu Linux (use a disposable VM for kernel module experiments)  
**Status:** Study material prepared; exercises and builds not yet completed

---

## Objectives
Learn buses, devices, drivers, probe/remove, sysfs, platform devices, Device Tree, GPIO, I²C, SPI, and driver binding.

## 1. Linux device model
The Linux driver model organizes **devices**, **drivers**, **buses**, and **classes**. When a device matches a driver, the driver may receive a `probe()` callback; the driver must release resources during removal or via managed-resource APIs.

- `/sys/bus/`: bus/device/driver relationships.
- `/sys/class/`: logical device classes.
- `/sys/devices/`: physical and logical device hierarchy.
- `/dev/`: user-visible device nodes where applicable.
- `devm_*` APIs: device-managed resources, usually cleaned up when the device detaches.

## 2. Explore your machine
```bash
ls /sys/bus
ls /sys/class
ls /sys/devices
find /sys/bus -maxdepth 2 -type d | head -30
lsmod | head
```
Available buses depend on the platform; x86 desktops often discover PCI/USB devices, while embedded ARM boards commonly describe onboard devices using Device Tree.

## 3. Platform driver structure (concept snippet)
```c
#include <linux/module.h>
#include <linux/of.h>
#include <linux/platform_device.h>

static int example_probe(struct platform_device *pdev)
{
    // Called when a matching platform device is bound.
    dev_info(&pdev->dev, "example device matched\n");
    return 0;
}

static void example_remove(struct platform_device *pdev)
{
    // Release manually managed resources if any were acquired.
    dev_info(&pdev->dev, "example device removed\n");
}

static const struct of_device_id example_matches[] = {
    { .compatible = "training,example-device" },
    { }
};
MODULE_DEVICE_TABLE(of, example_matches);

static struct platform_driver example_driver = {
    .probe = example_probe,
    .remove = example_remove,
    .driver = {
        .name = "training_example",
        .of_match_table = example_matches,
    },
};
module_platform_driver(example_driver);
MODULE_LICENSE("GPL");
```
This is a **teaching skeleton**: it only binds if a real matching platform device exists. Driver callback signatures can change between kernel releases; verify against the headers for your target kernel.

## 4. Device Tree concepts
Device Tree describes non-discoverable hardware, especially on embedded systems. A node's `compatible` property matches a driver; `reg` describes address ranges; `interrupts`, `clocks`, and GPIO properties describe resources.

Example DTS fragment (**illustrative, not board-ready**):
```dts
example@10000000 {
    compatible = "training,example-device";
    reg = <0x10000000 0x1000>;
    status = "okay";
};
```
Actual `reg` cell counts depend on the parent bus's `#address-cells` and `#size-cells`; a real node also needs correct bindings, addresses, and resources. Do not deploy invented addresses to hardware.

## 5. GPIO, I²C, and SPI
- **GPIO:** digital input/output lines; modern drivers use descriptor-based `gpiod_*` APIs.
- **I²C:** two-wire addressed bus for sensors, EEPROMs, and controllers.
- **SPI:** clocked serial bus with chip-select signals for displays, flash, and sensors.
- **Pin control:** selects pin muxing and electrical configuration.
- **Regulator/clock frameworks:** coordinate power and clock resources.

Driver API snippets:
```c
// Conceptual examples inside a properly matched device's probe().
struct gpio_desc *led = devm_gpiod_get(&pdev->dev, "led", GPIOD_OUT_LOW);
if (IS_ERR(led))
    return PTR_ERR(led);

// Use gpiod_set_value_cansleep() if the GPIO controller may sleep.
gpiod_set_value_cansleep(led, 1);
```
Requires appropriate `<linux/gpio/consumer.h>` and a real firmware GPIO mapping; this is not a complete standalone module.

## 6. Practical exercises
- [ ] Explore `/sys/bus` and `/sys/class`.
- [ ] Explain device/driver matching and `probe()`.
- [ ] Identify Device Tree `compatible` and `reg` properties.
- [ ] Compare GPIO, I²C, and SPI use cases.
- [ ] Sketch a platform driver and matching DTS node for a real board (without deploying).

## 7. Knowledge check
1. What triggers `probe()`?
2. Why do embedded boards use Device Tree?
3. What is the advantage of `devm_*` helpers?
4. When might a GPIO operation sleep?

**Next:** Lesson 9 — Embedded Linux Boot Process and Yocto Foundations.
**References:** https://docs.kernel.org/driver-api/driver-model/overview.html ; https://docs.kernel.org/devicetree/usage-model.html ; https://docs.kernel.org/driver-api/gpio/index.html
