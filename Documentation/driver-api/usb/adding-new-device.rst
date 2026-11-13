.. SPDX-License-Identifier: GPL-2.0

=====================================
Adding support for a new USB device
=====================================

This guide covers the usual steps for getting a USB device that Linux
does not yet handle to work, from finding out what the device is to
sending the patch.  Most new devices do not need a new driver.  The
usual change is one more line in the ID table of an existing driver,
sometimes with a quirk flag.  Writing a whole new driver is the last
resort; :doc:`writing_usb_driver` covers that case.

The guide uses a "Bluetooth speaker" as its running example, because
the term can mean three different things on Linux:

1. A speaker that you pair over Bluetooth.  It is not a USB device.
   The kernel only drives the Bluetooth *controller* (the USB dongle or
   the built-in chip).  The audio (A2DP, HFP) is done in user space by
   BlueZ and PipeWire or PulseAudio.  If the controller works and
   ``bluetoothctl`` can pair with the speaker, the kernel is done.
   Audio problems at that point are user-space problems.

2. A USB Bluetooth *adapter* that is not detected.  The driver is
   ``btusb`` (``drivers/bluetooth/btusb_main.c``) and the change is
   most likely an ID table entry.  See `Bluetooth adapters (btusb)`_.

3. A speaker with a USB cable (or a dock or headset that shows up as
   a USB sound card).  The driver is ``snd-usb-audio``
   (``sound/usb/``).  Class-compliant devices work without any change.
   Devices that break the spec need a quirk.  See
   `USB audio devices (snd-usb-audio)`_.

Step 1: identify the device
===========================

Plug the device in and collect:

- The vendor and product ID, from ``lsusb``::

    $ lsusb
    Bus 003 Device 007: ID 0bda:b85b Realtek Semiconductor Corp. Bluetooth Radio

- The full descriptors, from ``lsusb -v -d 0bda:b85b``.  The
  ``bDeviceClass``, ``bInterfaceClass``, ``bInterfaceSubClass`` and
  ``bInterfaceProtocol`` fields decide which driver can bind.  Some
  common classes are:

  ======  ==========================================
  0x01    Audio (``snd-usb-audio``)
  0x03    HID (``usbhid``)
  0x08    Mass storage (``usb-storage``, ``uas``)
  0x0e    Video (``uvcvideo``)
  0xe0    Wireless controller; 0xe0/0x01/0x01 is
          Bluetooth (``btusb``)
  0xff    Vendor specific: no generic driver binds
  ======  ==========================================

- The kernel log from the time the device was plugged in
  (``dmesg``), and the binding state from ``lsusb -t`` or
  ``/sys/bus/usb/devices/*/driver``.

If an interface already has a driver bound and the device works, there
is nothing to add.

Step 2: find the right driver
=============================

- If the interface uses a standard class code (audio, HID, storage,
  video, CDC, Bluetooth), the class driver binds it without any
  change.  If it does not work, the device most likely needs a quirk
  in that driver rather than a new ID.

- If the interface is vendor specific (0xff), look for a driver that
  supports other devices with the same chip.  Search for the vendor
  ID::

    $ git grep -i 'USB_DEVICE(0x0bda' drivers/ sound/

  Devices that are rebranded often use the chip maker's VID with a new
  PID, or the brand's own VID with the same chip.  The chip in the
  device is often named in the ``lsusb -v`` strings, in the Windows
  driver ``.inf`` file, or on the board.

Step 3: test without rebuilding
===============================

Most USB drivers let you add an ID at run time through ``new_id``
(see Documentation/ABI/testing/sysfs-bus-usb)::

  # modprobe btusb
  # echo "0bda b85b" > /sys/bus/usb/drivers/btusb/new_id

The format is ``VID PID [class [refVID refPID]]``.  If you give a
reference device, its ``driver_info`` is copied, so the new device gets
the same quirk flags as a supported device.  For a ``btusb`` device
with a Realtek chip, for example, pick a Realtek device from the table
as the reference.

If the device works this way, the patch is one line.  If it does not,
you have to look at the quirks, or the device needs more than a new ID.

Some drivers also take quirks as module parameters:

- ``usbcore.quirks=VID:PID:flags`` for USB core quirks (no LPM, delay
  on init, reset on resume, and so on).  See
  Documentation/admin-guide/kernel-parameters.rst.
- ``snd_usb_audio.quirk_alias`` and ``snd_usb_audio.quirk_flags`` for
  USB audio.  See Documentation/sound/alsa-configuration.rst.

Step 4: add the device ID
=========================

Each USB driver has a ``struct usb_device_id`` table that is exported
with ``MODULE_DEVICE_TABLE(usb, ...)``.  Udev uses it to load the
module on hotplug.  Use the narrowest match macro from
``include/linux/usb.h`` that works:

``USB_DEVICE(vid, pid)``
    Matches all interfaces of the device.

``USB_DEVICE_AND_INTERFACE_INFO(vid, pid, cl, sc, pr)``
    Matches only the interface with these class codes.  Use it for
    composite devices, for example a Wi-Fi and Bluetooth combo chip,
    so that the driver does not take an interface that belongs to
    another driver.

``USB_DEVICE_INTERFACE_NUMBER(vid, pid, num)``
    Matches a fixed interface number.

``USB_INTERFACE_INFO(cl, sc, pr)``
    Matches a class and needs no IDs.  Class drivers use it.

Put the entry next to devices with the same chip.  Keep the order that
the table already uses (usually sorted by VID and PID) and add a
comment that names the product.

Step 5: add quirks if needed
============================

Quirks are usually bit flags in the ``driver_info`` field of the ID
entry, or in a separate table that the driver checks.  Read the driver
to see how it uses them; do not guess.  Add a new flag only if none of
the existing flags describes the behaviour of the device.

USB core quirks that are not tied to one driver (LPM, resume and
enumeration problems) go into ``usb_quirk_list`` in
``drivers/usb/core/quirks.c``, with flags from
``include/linux/usb/quirks.h``.

Bluetooth adapters (btusb)
==========================

``btusb`` has two tables in ``drivers/bluetooth/btusb_main.c``:

``btusb_table``
    Decides which devices ``btusb`` binds.  It includes a generic match
    on the Bluetooth class (0xe0/0x01/0x01), so a class-compliant
    adapter binds without any entry.  Vendor-specific adapters
    (class 0xff) need an entry here.

``quirks_table``
    Checked in ``btusb_probe()`` when the ``btusb_table`` match has no
    ``driver_info``.  Class-compliant adapters that need vendor setup,
    such as firmware download, are listed here.

Most new adapters are a known chip with a new VID/PID.  Copy the entry
of a device with the same chip and keep all of its flags.  For example,
this is an existing entry for a Realtek adapter::

	/* Realtek 8852BE Bluetooth devices */
	{ USB_DEVICE(0x0bda, 0xb85b), .driver_info = BTUSB_REALTEK |
						     BTUSB_WIDEBAND_SPEECH },

The vendor flags (``BTUSB_REALTEK``, ``BTUSB_MEDIATEK``,
``BTUSB_INTEL_COMBINED``, ``BTUSB_QCA_WCN6855`` and others) choose the
setup code and the firmware.  ``BTUSB_WIDEBAND_SPEECH`` enables mSBC
for headsets.  Make sure that the firmware the driver asks for (see
``dmesg``) is in linux-firmware.

Test it: ``btmgmt info`` or ``bluetoothctl show`` lists the controller,
and you can scan, pair and play audio to a Bluetooth speaker.

By convention, btusb patches include the
``/sys/kernel/debug/usb/devices`` entry of the device in the commit
message.

USB audio devices (snd-usb-audio)
=================================

USB Audio Class 1, 2 and 3 devices work without driver changes.  When a
device breaks the spec (wrong sample rates, broken volume controls,
problems at interface setup), add a quirk.  From the least to the most
work:

1. A flag in ``quirk_flags_table`` in ``sound/usb/quirks.c``::

	DEVICE_FLG(0x1234, 0x5678, /* Example USB Speaker */
		   QUIRK_FLAG_GET_SAMPLE_RATE),

   The flags are described in ``sound/usb/usbaudio.h``.  Test them first
   with the ``quirk_flags`` module parameter.

2. Mixer fixes (volume range, control names) in
   ``sound/usb/mixer_maps.c`` or ``sound/usb/mixer_quirks.c``.

3. A full entry in ``sound/usb/quirks-table.h`` that describes the
   interfaces and endpoints by hand.  This is for devices that are not
   class compliant at all.

Test playback with ``aplay -D hw:CARD=...`` at each supported rate and
format, and test the volume control with ``alsamixer``.

Sending the patch
=================

- Make one patch for each driver, and only the change that is needed:
  usually one ID entry or one quirk.
- Write the device name and its VID/PID in the subject and the commit
  message, and say how you tested it.  Add the ``lsusb -v`` output or
  the ``/sys/kernel/debug/usb/devices`` entry if the subsystem asks for
  it.
- Find the maintainers and lists with ``scripts/get_maintainer.pl``,
  and run ``scripts/checkpatch.pl`` before you send.  New device IDs are
  usually accepted for stable kernels.  Add ``Cc: stable@vger.kernel.org``
  (see Documentation/process/stable-kernel-rules.rst).
- Follow Documentation/process/submitting-patches.rst.

Writing a new driver
====================

If no existing driver handles the device and the protocol is not a
standard class, a new driver is needed.  First check whether user space
can handle it with libusb or usbfs.  Many vendor-specific devices do
not need kernel code.  If kernel code is needed, see
:doc:`writing_usb_driver`, :doc:`URB`, :doc:`power-management` and
:doc:`hotplug`, and look at a small driver such as
``drivers/usb/misc/usblcd.c`` as an example.
