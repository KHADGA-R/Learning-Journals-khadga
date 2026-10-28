# Lesson 10 of 12 — Building Your First Yocto Image with QEMU

**Course:** Linux Kernel Programming & Yocto Project  
**Prepared:** 2026-10-28  
**Environment:** Ubuntu Linux / QEMU  
**Status:** Study material prepared; exercises and builds not yet completed

---

## Objectives
Prepare a supported Ubuntu host, obtain a compatible Poky release, initialize a build, configure MACHINE, run BitBake, locate images, and boot a QEMU target.

## 1. Prerequisites and planning
Yocto builds are resource intensive. Plan for roughly 100 GB or more of free disk space and 16 GB RAM if possible; actual needs vary by release, image, and parallelism. Prefer a native Linux filesystem (not a Windows-mounted filesystem) and a supported host distribution. **Check the current release's supported distributions and required packages before building**: https://docs.yoctoproject.org/ref-manual/system-requirements.html

## 2. Install common host tools (Ubuntu)
```bash
sudo apt update
sudo apt install -y gawk wget git diffstat unzip texinfo gcc build-essential \
    chrpath socat cpio python3 python3-pip python3-pexpect xz-utils \
    debianutils iputils-ping file locales zstd liblz4-tool
sudo locale-gen en_US.UTF-8
```
Package names vary with Ubuntu/Yocto release; follow the selected release's documentation if a dependency differs.

## 3. Obtain Poky
```bash
mkdir -p ~/yocto-learning && cd ~/yocto-learning
git clone https://git.yoctoproject.org/poky
cd poky
git branch -r | head
```
**Pin a supported stable release branch** instead of building a moving development branch. Example placeholder:
```bash
# Replace RELEASE_BRANCH with a real supported branch from the release notes.
git checkout RELEASE_BRANCH
```
Do not run that literal placeholder; choose a release from https://docs.yoctoproject.org/migration-guides/release-notes.html and check out its exact branch.

## 4. Initialize and configure
```bash
source oe-init-build-env build-qemu
```
This changes the current shell's working directory to the build directory. Edit `conf/local.conf` and set:
```conf
MACHINE ??= "qemux86-64"
```
If an existing `MACHINE` assignment is present, edit it rather than appending a conflicting value. `qemux86-64` is a common QEMU machine target.

## 5. Build a minimal image
```bash
bitbake core-image-minimal
```
The first build may take hours and download substantial source archives. Do not delete `downloads/` or `sstate-cache/` casually; they accelerate rebuilds.

## 6. Locate outputs
```bash
ls tmp/deploy/images/qemux86-64/
bitbake -e core-image-minimal | grep '^MACHINE=' | head -1
```
Artifacts depend on the selected release and image type.

## 7. Boot in QEMU
```bash
runqemu qemux86-64 core-image-minimal nographic
```
Use the QEMU launcher included with your initialized Poky environment. Quit according to the QEMU console instructions; `Ctrl+A`, then `X` is common for nographic QEMU serial consoles. Login behavior varies by image and configuration.

## 8. Useful BitBake commands
```bash
bitbake -s                       # List recipes and versions.
bitbake -c listtasks core-image-minimal
bitbake -e core-image-minimal | less
bitbake -g core-image-minimal    # Generate dependency graphs.
```
Use `bitbake -c clean <recipe>` with care; it removes task outputs. Avoid `cleansstate` unless necessary.

## 9. Common problems
- Missing host package: consult release-specific requirements.
- Low disk space: check `df -h` and build directory growth.
- Fetch failures: verify network/proxy and upstream availability.
- Unsupported host distro: use a supported Ubuntu VM/container configuration per official guidance.
- Slow build: limit `BB_NUMBER_THREADS` and `PARALLEL_MAKE` according to RAM/CPU.

## 10. Exercises
- [ ] Verify host meets requirements.
- [ ] Choose and record a stable Poky release.
- [ ] Initialize build and inspect `conf/local.conf`.
- [ ] Build `core-image-minimal` (pending until run).
- [ ] Boot QEMU and record kernel version.
- [ ] Find image files and inspect build metadata.

## 11. Knowledge check
1. What does `oe-init-build-env` set up?
2. What does `MACHINE` select?
3. What is `core-image-minimal`?
4. Why pin a Yocto release branch?

**Next:** Lesson 11 — Custom Recipes, Layers, and Application Integration.
**References:** https://docs.yoctoproject.org/brief-yoctoprojectqs/index.html ; https://docs.yoctoproject.org/dev-manual/start.html
