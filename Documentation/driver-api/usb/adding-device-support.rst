.. SPDX-License-Identifier: GPL-2.0

==============================
Adding support for USB devices
==============================

This guide describes how to add Linux host-side support for a USB device,
using headphones, headsets and microphones as examples. It also applies to
other functions, such as keyboards, cameras and network adapters. To make
Linux act as a USB peripheral instead, see :doc:`gadget`.

A product type is not necessarily a new USB device class. Many products
implement a standard class protocol and work with an existing driver.
Start by identifying the interfaces and the missing functionality before
writing a driver or adding a device ID.

Identify the device and its interfaces
======================================

Connect the device and collect the kernel log and USB descriptors. The
following commands use ``lsusb`` from usbutils; access to full descriptors
and kernel logs may require elevated privileges::

    uname -r
    dmesg --follow
    lsusb
    lsusb -t
    lsusb -v -d VVVV:PPPP

Replace ``VVVV:PPPP`` with the hexadecimal vendor and product IDs reported
by ``lsusb``. Record the product name, firmware or device revision
(``bcdDevice``), and whether the problem occurs on a recent mainline kernel.
If several identical devices are attached, select a bus and device address
with ``lsusb -s BUS:DEVICE -v`` instead.

Inspect each interface's class, subclass, protocol, alternate settings and
endpoints. A device-level class of zero means that class information is
provided at the interface level. A composite headset can expose audio
playback, audio capture and HID buttons on separate interfaces. USB
interface drivers bind to interfaces, so one product can use several
drivers. ``lsusb -t`` shows the driver bound to each interface.

Distinguish enumeration failures from driver failures. If the host cannot
read the device descriptors, first investigate cables, power, hubs and the
host controller. Adding an ID to a function driver will not fix enumeration.

Choose the support path
=======================

Use the interface descriptors and the protocol documentation to choose
among these approaches:

* **An existing class driver:** USB Audio Class devices normally use
  ``snd-usb-audio``, HID devices use the HID subsystem, and USB Video Class
  cameras use ``uvcvideo``. Class drivers often match interface classes, so
  a new product does not necessarily need a vendor/product ID entry.
* **An existing driver with a device-specific change:** A compatible device
  may need an ID entry, while a device that violates the class specification
  may need a narrowly scoped quirk. Check the driver's match table and
  probe path before deciding which change is needed.
* **A new protocol implementation:** If no driver implements the protocol,
  obtain the vendor's protocol documentation or investigate the transfers.
  See :doc:`/usb/usbmon` for USB tracing. Integrate with the appropriate
  subsystem, such as ALSA, HID, networking or media, so applications can use
  its established interfaces.

Do not infer compatibility from the product name or similar IDs alone.
Matching an interface only causes a probe attempt; the driver still needs
an implementation of the device's protocol. Avoid broad matches that
claim unrelated interfaces on a composite device.

Example: USB headphones or a microphone
=======================================

First check that the kernel has USB and ALSA support, including
``CONFIG_SND_USB`` and ``CONFIG_SND_USB_AUDIO`` and their dependencies.
``CONFIG_SND_USB_AUDIO=m`` builds the ``snd-usb-audio`` module. With that
module installed for the running kernel, it can be loaded explicitly::

    modprobe snd-usb-audio

For a class-compliant device, check whether ALSA exposes playback or capture
before changing the kernel. These commands require alsa-utils::

    cat /proc/asound/cards
    aplay -l
    arecord -l
    amixer -c N contents

Replace ``N`` with the device's ALSA card number. Headphones normally need
playback, microphones need capture, and headsets may need both. Check mixer
mute, volume and capture controls. If ALSA exposes and operates the device
but a desktop application does not, investigate sound-server profiles,
routing and application settings.

Test a known playback file and make a short recording directly through
ALSA. For example, for a device that supports stereo 48 kHz, 16-bit PCM::

    aplay -D plughw:N,0 known-good.wav
    arecord -D hw:N,0 -f S16_LE -r 48000 -c 2 -d 5 capture.wav
    aplay capture.wav

Replace the card and PCM device numbers with those from ``aplay -l`` or
``arecord -l``, and choose a capture format the device advertises. A mono
microphone may require ``-c 1``. ``plughw`` permits format conversion;
``hw`` tests the hardware parameters directly. Where available,
``/proc/asound/cardN/stream*`` describes USB audio streams and their formats.
Start playback at a low volume.

For kernel changes, inspect the existing implementation in ``sound/usb/``:

* ``card.c`` contains driver registration, matching and probe logic.
* ``quirks-table.h`` describes device-specific interface handling.
* ``quirks.c`` implements quirks and device-specific quirk flags.
* ``mixer.c`` and ``mixer_quirks.c`` handle mixer controls and their quirks.
* ``stream.c``, ``format.c`` and ``endpoint.c`` handle stream descriptions,
  formats and endpoints.

Identify the exact descriptor or protocol behavior that prevents operation.
Reuse an existing quirk only if its behavior fits the device, and restrict
it to the affected IDs, interfaces or revisions as appropriate. Document
why the device needs the exception. Do not copy an unrelated product's
quirk merely to make the driver bind.

Headset volume or mute buttons may use HID independently of USB audio.
Inspect those interfaces separately; see :doc:`/hid/hidintro`. A functioning
audio stream does not establish that the buttons or LEDs work.

Implementing a new driver
=========================

Read :doc:`writing_usb_driver` and examine a current driver for a similar
protocol. ``drivers/usb/usb-skeleton.c`` illustrates USB driver lifetime
management, but its bulk transfers are not a template for isochronous
audio streaming. For audio subsystem interfaces, see
:doc:`/sound/kernel-api/writing-an-alsa-driver`.

A new driver needs to:

* Describe its configuration dependencies in Kconfig and add a Makefile
  entry in the relevant subsystem.
* Provide a suitable ``struct usb_device_id`` table and
  ``MODULE_DEVICE_TABLE(usb, ...)`` for module aliases. Prefer interface
  matches where necessary to avoid claiming other device functions.
* Validate descriptors, endpoint types, packet sizes and alternate settings
  in probe. Treat device-provided lengths and values as untrusted input.
* Use the transfer types required by the protocol. Streaming audio commonly
  uses isochronous transfers; do not substitute bulk transfers for them.
* Register the function with its subsystem, with complete failure cleanup.
* Stop outstanding I/O and prevent new submissions during disconnect.
  Ensure completion handlers, work items and open users cannot access freed
  memory. Handle reset, suspend and resume as required.

See :doc:`URB`, :doc:`anchors` and :doc:`power-management` for the USB APIs
and lifetime rules. If firmware is required, use the firmware loader and
explain how users obtain the required firmware.

Build, test and submit
======================

Build and install a kernel or modules with the affected configuration
using :doc:`/admin-guide/quickly-build-trimmed-linux`. Boot that kernel and
verify its version before testing. Test the supported operations rather
than just successful probe:

* Plug, unplug and reconnect, including unplugging during active I/O.
* Exercise playback, capture and simultaneous operation where supported.
  Check advertised rates, formats and channel counts, mixer controls,
  buttons and mute behavior.
* Exercise suspend/resume and runtime power management. Check the kernel
  log for warnings, transfer failures or resource leaks.
* Check another device using the same driver where available, especially
  when changing a shared class-driver path.

For a documentation-only patch, build the affected documentation instead.
For example, from the kernel source root::

    make SPHINXDIRS=driver-api/usb htmldocs

Follow :doc:`/process/submitting-patches`. Include the device IDs and
revision, relevant descriptors and logs, the reason for the chosen support
path, and the tests performed. State which hardware or scenarios were not
tested. Run ``scripts/checkpatch.pl`` on the patch and use
``scripts/get_maintainer.pl`` to identify the relevant subsystem maintainers
and mailing lists.
