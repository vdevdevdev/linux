.. SPDX-License-Identifier: GPL-2.0

===============================
Adding support for a USB device
===============================

This guide describes how to support a new USB device connected to a Linux
host, using a mass storage device as an example. Start by identifying the
protocol the device implements: a new product often works with an existing
class driver and needs no kernel changes.

If instead you want a Linux system to *appear* as a USB device to another
computer, use the USB gadget framework. See `Making Linux a storage device`_ below.

Identify the device and its interfaces
=====================================

Connect the device and collect its descriptors, the bound drivers, and the
kernel messages. The following commands use tools from usbutils::

    lsusb
    lsusb -t
    sudo lsusb -v -d VVVV:PPPP
    sudo dmesg

Replace ``VVVV:PPPP`` with the hexadecimal vendor and product IDs reported
by ``lsusb``. Record the device revision (``bcdDevice``), interface class,
subclass, protocol, alternate settings, and endpoint descriptors. Obtain
the device's protocol documentation where available.

USB drivers normally bind to interfaces. A composite device can have
several interfaces handled by different drivers, so inspect the interface
descriptors rather than relying only on ``bDeviceClass`` or the product
name. For example, a camera may provide both an imaging interface and a
mass storage interface.

Choose the existing subsystem
=============================

Look for a driver that already implements the device's class or protocol.
For example, storage devices use the SCSI subsystem, keyboards use HID,
and network adapters use the networking subsystem. Search the relevant
drivers for matching interface classes, protocol implementations, and
related products.

For a USB disk, the usual path is::

    USB interface -> usb-storage or uas -> SCSI core -> sd -> block device

A typical Bulk-Only Transport storage interface reports:

* ``bInterfaceClass`` = ``08`` (mass storage).
* ``bInterfaceSubClass`` = ``06`` (SCSI transparent command set).
* ``bInterfaceProtocol`` = ``50`` (Bulk-Only Transport).
* One bulk IN endpoint and one bulk OUT endpoint.

These descriptor values are hexadecimal. USB Attached SCSI (UAS) uses
protocol ``62`` and has different endpoint requirements. A device may
provide alternate settings for both transports; let the existing drivers
select the appropriate one.

Enable ``CONFIG_USB`` and the host controller driver for the machine.
For a typical USB disk, also enable ``CONFIG_SCSI``, ``CONFIG_BLK_DEV_SD``,
and ``CONFIG_USB_STORAGE``. Enable ``CONFIG_USB_UAS`` for UAS support.
See ``drivers/usb/storage/Kconfig`` for dependencies and additional
protocol-specific drivers. Build and install the kernel and modules as
described in Documentation/admin-guide/quickly-build-trimmed-linux.rst.
If built as modules, the storage drivers can be loaded with::

    sudo modprobe usb-storage
    sudo modprobe uas

Reconnect the device and check ``lsusb -t`` and the kernel log. Standard
class matching should bind a compliant device without adding its vendor
and product IDs. Use ``lsblk`` to identify the resulting disk and its
partitions. A missing filesystem driver or an unformatted disk is a
separate issue from USB driver binding.

Extend an existing driver when necessary
=======================================

If the device fails, first establish where it fails: enumeration, driver
binding, transport commands, SCSI discovery, or filesystem access. Compare
the descriptors and behaviour with the protocol specification and a known
working device. Documentation/usb/usbmon.rst explains how to capture USB
traffic to investigate command failures.

For a device that implements an existing vendor protocol, a narrowly
scoped ID table entry may be sufficient. Verify protocol compatibility
before adding an ID; matching a driver does not make an incompatible
protocol work. Check that the match does not claim unrelated interfaces
of a composite device.

Storage devices sometimes require a workaround for a firmware defect:

* ``drivers/usb/storage/unusual_devs.h`` records many usb-storage quirks
  and protocol overrides. Follow the instructions at the top of that file,
  keep entries sorted, and limit the device revision range to the affected
  firmware.
* ``drivers/usb/storage/unusual_uas.h`` records UAS-specific entries.
  Check ``drivers/usb/storage/uas-detect.h`` when investigating transport
  selection.
* The ``usb-storage.quirks=`` kernel parameter allows experiments with
  supported flags before making a permanent change. Its syntax and flags
  are documented in Documentation/admin-guide/kernel-parameters.rst.

Explain the observed failure and why the chosen quirk fixes it. Do not
copy flags from a similar product without testing them. If the existing
transport cannot express the device's protocol, implement the missing
protocol support within the appropriate subsystem.

Write a new driver only for a new protocol
=========================================

For a device that cannot use an existing driver, first decide whether a
userspace implementation using usbfs (for example, through libusb) is
sufficient. A kernel driver is appropriate when the device needs kernel
subsystem integration, such as presenting storage through the block layer.

For a new kernel driver:

1. Place it in the appropriate subsystem and add Kconfig and Makefile
   entries. For storage, study ``drivers/usb/storage/`` and its SCSI
   integration before creating a separate driver.
2. Define a suitably scoped ``struct usb_device_id`` table and export it
   with ``MODULE_DEVICE_TABLE(usb, ...)`` for module autoloading.
3. Register a ``struct usb_driver`` with probe and disconnect callbacks,
   normally using ``module_usb_driver()``. In probe, validate the interface,
   alternate setting, and endpoints before allocating resources and
   registering with the target subsystem.
4. Implement transfers, error recovery, and reset handling using the USB
   APIs. Validate device-supplied lengths and descriptors before using
   them. Handle short transfers and stalled endpoints according to the
   protocol.
5. Implement disconnect and power management with correct resource
   lifetimes. Stop new I/O, cancel outstanding URBs, and ensure callbacks
   cannot access freed memory. A device can disappear during any operation.

See Documentation/driver-api/usb/writing_usb_driver.rst for the driver
structure and Documentation/driver-api/usb/usb.rst for the APIs. Consult
``drivers/usb/usb-skeleton.c`` and current drivers for implementation
examples. A storage driver should expose disks through SCSI and the block
layer rather than inventing a private read/write device interface.

Test and submit the support
===========================

Use disposable media for storage write tests. Identify the target with
``lsblk`` before mounting or writing to it; device names can change after
reconnection.

Verify enumeration and automatic driver binding, then test reads and
writes with checksums, flushes, unmounting, and reconnection. Exercise
multiple logical units if supported. Test disconnect during I/O, error
recovery, suspend/resume, and repeated plug/unplug cycles. Where possible,
test different host controllers and USB speeds, both transports for a
device supporting UAS and Bulk-Only Transport, and existing devices
sharing the changed driver. Record any tests you could not perform.

Build the affected drivers and check the patch with
``scripts/checkpatch.pl``. Include the device IDs and revision, relevant
descriptors, failure logs, and before/after test results in the patch
description. Follow Documentation/process/submitting-patches.rst and use
``scripts/get_maintainer.pl`` to find the appropriate reviewers.

Making Linux a storage device
=============================

To expose storage *from* Linux to a USB host, the hardware needs a USB
Device Controller (UDC) and a port operating in peripheral mode. A
host-only port cannot provide this function.

Enable the controller's UDC driver and ``CONFIG_USB_GADGET``. For a gadget
assembled through configfs, enable ``CONFIG_USB_CONFIGFS`` and
``CONFIG_USB_CONFIGFS_MASS_STORAGE``. Follow
Documentation/usb/gadget_configfs.rst to create the gadget, set its device
IDs and strings, create a ``mass_storage`` function, configure its backing
file or block device, link it into a configuration, and bind it to a UDC.
See the mass storage section of Documentation/usb/gadget-testing.rst
for function-specific configuration. Use vendor and product IDs assigned
for your device.

Documentation/usb/mass-storage.rst describes backing storage, logical
units, read-only operation, and the alternative ``g_mass_storage`` gadget.
Never mount or modify backing storage locally while the host can write to
it. The two systems do not coordinate filesystem caches; simultaneous
access can corrupt data. Start with a disposable backing image and
read-only access, then test from a separate USB host.
