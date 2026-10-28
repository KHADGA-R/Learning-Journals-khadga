# Lesson 11 of 12 — Yocto Custom Recipes, Layers, and Application Integration

**Course:** Linux Kernel Programming & Yocto Project  
**Prepared:** 2026-10-28  
**Environment:** Ubuntu Linux / QEMU  
**Status:** Study material prepared; exercises and builds not yet completed

---

## Objectives
Create a custom layer, write a recipe for a C application, explain `SRC_URI`, `S`, `do_compile`, `do_install`, packaging, layer priorities, overrides, and image integration.

## 1. Create a custom layer
Run from an initialized Poky build environment (Lesson 10):
```bash
cd ~/yocto-learning/poky
source oe-init-build-env build-qemu
bitbake-layers create-layer ../meta-learning
bitbake-layers add-layer ../meta-learning
bitbake-layers show-layers
```
Layer paths and generated templates may differ slightly by release.

## 2. Add a small C application
Create `../meta-learning/recipes-learning/hello/files/hello.c`:
```c
#include <stdio.h>

int main(void)
{
    // Simple target-side smoke test for our custom Yocto package.
    puts("Hello from my Yocto image!");
    return 0;
}
```

Create `../meta-learning/recipes-learning/hello/hello_1.0.bb`:
```bitbake
SUMMARY = "Small learning application"
DESCRIPTION = "Demonstrates compiling and packaging a C program with Yocto"
LICENSE = "CLOSED"

SRC_URI = "file://hello.c"
S = "${WORKDIR}"

do_compile() {
    # Use Yocto's target compiler and target-specific flags.
    ${CC} ${CFLAGS} ${LDFLAGS} hello.c -o hello-learning
}

do_install() {
    # Install into the recipe staging tree, not the host filesystem.
    install -d ${D}${bindir}
    install -m 0755 hello-learning ${D}${bindir}/hello-learning
}
```
`LICENSE = "CLOSED"` is used only for this original learning example; real third-party software must declare its actual license and appropriate checksums. Depending on the release, `S = "${WORKDIR}"` may be replaced by `S = "${UNPACKDIR}"`; follow the selected branch's migration guidance. `CC`, `CFLAGS`, and `LDFLAGS` are supplied by the build environment.

## 3. Build and add to an image
```bash
bitbake hello
bitbake-layers show-recipes hello
```
Add this line to `conf/local.conf`:
```conf
IMAGE_INSTALL:append = " hello"
```
The leading space inside the quotes matters. Rebuild:
```bash
bitbake core-image-minimal
runqemu qemux86-64 core-image-minimal nographic
```
Inside QEMU:
```sh
hello-learning
```

## 4. Recipe fundamentals
- `SUMMARY` / `DESCRIPTION`: package metadata.
- `LICENSE` / `LIC_FILES_CHKSUM`: license declaration and validation when required.
- `SRC_URI`: where sources or patches are obtained.
- `S`: directory containing source during build.
- `B`: build directory.
- `D`: temporary destination directory for install.
- `DEPENDS`: build-time recipe dependencies.
- `RDEPENDS:${PN}`: runtime package dependencies.
- `FILES:${PN}`: files assigned to the main package.
- `.bbappend`: extends an existing recipe from another layer.
- `PACKAGECONFIG`: optional build features.

## 5. Debugging
```bash
bitbake -e hello | grep '^S='
bitbake -c listtasks hello
bitbake -c compile -f hello
bitbake -c install -f hello
```
Force-running tasks is useful for investigation but can affect task signatures; use a normal `bitbake hello` afterward.

## 6. Exercises
- [ ] Create `meta-learning` and confirm it appears in `show-layers`.
- [ ] Write and build `hello_1.0.bb`.
- [ ] Add the package to an image and run it in QEMU.
- [ ] Explain the difference between `DEPENDS` and `RDEPENDS`.
- [ ] Inspect the recipe's compile/install logs.
- [ ] Explore a `.bbappend` in an existing layer.

## 7. Knowledge check
1. Why use ${CC} instead of hard-coded `gcc`?
2. What is the role of ${D}?
3. Why does `IMAGE_INSTALL:append` include a leading space?
4. What is the difference between a recipe and a layer?

**Next:** Lesson 12 — Integrating Kernel Modules with Yocto, Debugging, and Final Project.
**References:** https://docs.yoctoproject.org/dev-manual/layers.html ; https://docs.yoctoproject.org/dev-manual/new-recipe.html ; https://docs.yoctoproject.org/ref-manual/variables.html
