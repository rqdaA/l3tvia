+++
title = "How to Build Android 15 arm64 for QEMU"
description = "Simple guide to building Android 15 (AOSP) for arm64 architecture on QEMU emulator"
date = 2025-02-19T12:00:00Z
draft = false

[taxonomies]
tags = ["Build","Setup"]
[extra]
toc = true
series = "Setup"
+++

# ソースコードを取得

```sh
 mkdir android && cd android
 repo init -u https://android.googlesource.com/kernel/manifest -b common-android15-6.6-lts
 repo sync -c -d -j`nproc`
```

# ソースコード改変

必要なLKMをkernel本体にくっつける作業

```diff
diff --git a/arch/arm64/configs/gki_defconfig b/arch/arm64/configs/gki_defconfig
index e4a6cfe6fa5b..260082894c00 100644
--- a/arch/arm64/configs/gki_defconfig
+++ b/arch/arm64/configs/gki_defconfig
@@ -785,3 +785,19 @@ CONFIG_KUNIT_TEST=m
 CONFIG_KUNIT_EXAMPLE_TEST=m
 # CONFIG_KUNIT_DEFAULT_ENABLED is not set
 # CONFIG_RUNTIME_TESTING_MENU is not set
+
+CONFIG_PTP_1588_CLOCK=y
+CONFIG_E1000=y
+CONFIG_E1000E=y
+CONFIG_VIRTIO_PCI=y
+CONFIG_VIRTIO_BALLOON=y
+CONFIG_VIRTIO_BLK=y
+CONFIG_VIRTIO_NET=y
+CONFIG_9P_FS_SECURITY=y
+CONFIG_NET_9P=y
+CONFIG_NET_9P_VIRTIO=y
+CONFIG_NET_9P_DEBUG=y
+CONFIG_9P_FS=y
+CONFIG_9P_FS_POSIX_ACL=y
+CONFIG_DEVTMPFS=y
+CONFIG_DEVTMPFS_MOUNT=y

diff --git a/build.config.gki b/build.config.gki
index 4b931d9eb333..da87a30d24e8 100644
--- a/build.config.gki
+++ b/build.config.gki
@@ -1,2 +1,2 @@
 DEFCONFIG=gki_defconfig
-POST_DEFCONFIG_CMDS="check_defconfig"
+###POST_DEFCONFIG_CMDS="check_defconfig"

diff --git a/modules.bzl b/modules.bzl
index c93be156738d..9ad3e7624c74 100644
--- a/modules.bzl
+++ b/modules.bzl
@@ -8,7 +8,7 @@ This module contains a full list of kernel modules

 _COMMON_GKI_MODULES_LIST = [
     # keep sorted
-    "drivers/block/virtio_blk.ko",
+    # "drivers/block/virtio_blk.ko",
     "drivers/block/zram/zram.ko",
     "drivers/bluetooth/btbcm.ko",
     "drivers/bluetooth/btqca.ko",
@@ -39,16 +39,16 @@ _COMMON_GKI_MODULES_LIST = [
     "drivers/net/usb/rtl8150.ko",
     "drivers/net/usb/usbnet.ko",
     "drivers/net/wwan/wwan.ko",
-    "drivers/pps/pps_core.ko",
-    "drivers/ptp/ptp.ko",
+    # "drivers/pps/pps_core.ko",
+    # "drivers/ptp/ptp.ko",
     "drivers/usb/class/cdc-acm.ko",
     "drivers/usb/mon/usbmon.ko",
     "drivers/usb/serial/ftdi_sio.ko",
     "drivers/usb/serial/usbserial.ko",
-    "drivers/virtio/virtio_balloon.ko",
-    "drivers/virtio/virtio_pci.ko",
-    "drivers/virtio/virtio_pci_legacy_dev.ko",
-    "drivers/virtio/virtio_pci_modern_dev.ko",
+    # "drivers/virtio/virtio_balloon.ko",
+    # "drivers/virtio/virtio_pci.ko",
+    # "drivers/virtio/virtio_pci_legacy_dev.ko",
+    # "drivers/virtio/virtio_pci_modern_dev.ko",
     "kernel/kheaders.ko",
     "lib/crypto/libarc4.ko",
     "mm/zsmalloc.ko",
@@ -61,8 +61,8 @@ _COMMON_GKI_MODULES_LIST = [
     "net/6lowpan/nhc_routing.ko",
     "net/6lowpan/nhc_udp.ko",
     "net/8021q/8021q.ko",
-    "net/9p/9pnet.ko",
-    "net/9p/9pnet_fd.ko",
+    # "net/9p/9pnet.ko",
+    # "net/9p/9pnet_fd.ko",
     "net/bluetooth/bluetooth.ko",
     "net/bluetooth/hidp/hidp.ko",
     "net/bluetooth/rfcomm/rfcomm.ko",
@@ -97,7 +97,7 @@ _ARM64_GKI_MODULES_LIST = [
     "arch/arm64/geniezone/gzvm.ko",
     "drivers/char/hw_random/cctrng.ko",
     "drivers/misc/open-dice.ko",
-    "drivers/ptp/ptp_kvm.ko",
+    # "drivers/ptp/ptp_kvm.ko",
```

# Build

```sh
tools/bazel run //common:kernel_aarch64_dist
```

# Create rootfs

**WIP**

# Boot

```sh
qemu-system-aarch64 \
        -m 2048 \
        -smp 2 \
        -machine virt \
        -cpu cortex-a57 \
        -display none \
        -serial stdio \
        -no-reboot \
        -device virtio-rng-pci \
        -cpu coretex-a57 \
        -device e1000,netdev=net0 \
        -netdev user,id=net0,restrict=on,hostfwd=tcp::20022-:22 \
        -drive file=./rootfs.ext2,if=virtio,format=raw,cache=none \
        -snapshot \
        -kernel ./Image \
        -append 'rdinit=/linuxrc console=ttyAMA0 root=/dev/vda verbose'
```
