# Minimal Linux System with QEMU

This project demonstrates how to build and boot a minimal, self-contained Linux system using a custom kernel, BusyBox, and QEMU. It is useful for learning kernel internals, early user space, and embedded Linux principles.

---
## 1. Install dependencies
```bash
sudo apt install -y qemu-system-x86 cpio build-essential
```
## 2. Prepare the Directory Structure
Create the root directory for the initramfs environment:

```bash
mkdir -p initramfs/{bin,sbin,etc,proc,sys,usr/bin,usr/sbin}
```
Resulting structure:
```bash
initramfs/
├── bin/
├── etc/
├── init         # Init script (PID 1)
├── proc/
├── sbin/
├── sys/
└── usr/
```
## 3. Update submodules
```bash
git submodule update --init --recursive
```
## 3. Build Kernel
```bash
cd externel/linux
git checkout v6.12.23
make defconfig
make -j$(nproc)
```
## 4. Build Busybox
Build static binary (no shared libs)

Disable tc in Networking Utilities (to avoid build errors)

If `menuconfig` fails due to `ncurses`, see this fix:
[Stack Overflow workaround](https://stackoverflow.com/questions/78491346/busybox-build-fails-with-ncurses-header-not-found-in-archlinux-spoiler-i-alrea)
```bash
cd external/busybox
make defconfig
make menuconfig
make -j$(nproc)
```
## 5. Install
```bash
cp external/busybox/busybox initramfs/bin/
cd initramfs/bin

./busybox --list | while read applet; do
  [ "$applet" = "busybox" ] && continue
  ln -sf busybox "$applet"
done
```
## 6. Create the `init` script
```bash
nano /initramfs/init
```
Content
```bash
#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
echo "Welcome to minimal initramfs"
exec /bin/sh
```
Make it executable:
```bash
chmod +x initramfs/init
```
## 7. Build the Initramfs Image
```bash
cd initramfs
find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../initramfs.cpio.gz
```
## 8. Boot the System with QEMU
```bash
qemu-system-x86_64 \
  -kernel external/linux/arch/x86/boot/bzImage \
  -initrd initramfs.cpio.gz \
  -nographic -m 512 \
  -append "console=ttyS0"
```
## 9. Exit the system
```bash
Ctrl + A  then  X
```