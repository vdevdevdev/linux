.. SPDX-License-Identifier: GPL-2.0

Adding support for a new USB device
==================================

This guide describes how to choose and implement Linux support for a new
USB device, using a keyboard as an example. Start with the device's USB
interface descriptors and protocol documentation: a new product does not
necessarily need a new driver.

Check existing support
----------------------

Connect the device and inspect the kernel log and USB descriptors::

    dmesg --follow
    lsusb
    lsusb -t
    sudo lsusb -v -d VVVV:PPPP

Run the log command in a separate terminal before connecting the device.
Replace ``VVVV:PPPP`` with the hexadecimal vendor and product IDs reported
by ``lsusb``. The ``lsusb`` utility is provided by usbutils. Record the IDs,
firmware revision, interface numbers, interface classes, and endpoint
addresses and transfer types. ``lsusb -t`` also shows the bound USB drivers.
A composite device can expose several interfaces with different drivers;
inspect the interface descriptors, not just the device class.

For a USB HID keyboard, the usual path is::

    USB interface -> usbhid -> HID device -> hid-generic -> input subsystem

The input subsystem exposes key events through evdev, normally as
``/dev/input/eventX`` when ``CONFIG_INPUT_EVDEV`` is enabled. Enable
``CONFIG_USB``, the appropriate USB host controller driver, ``CONFIG_HID``,
``CONFIG_USB_HID``, and ``CONFIG_HID_GENERIC`` in the test kernel. These
options may be built in or selected as modules where supported.

A standards-compliant HID keyboard normally works without adding its
vendor and product IDs to the kernel. USB HID uses a report descriptor to
describe keys, buttons, and LEDs. Inspect that descriptor and the reports
before deciding that a driver is missing. See
Documentation/hid/hidintro.rst for descriptor decoding and report capture.
If the expected key events already appear in evdev, investigate userspace
key mapping and desktop configuration before changing a kernel driver.

Choose the right subsystem
--------------------------

Use the smallest change that implements the missing functionality:

* If an existing device-specific driver implements the same protocol,
  verify compatibility and extend its device table if necessary. A matching
  vendor name alone does not demonstrate protocol compatibility.
* For a HID keyboard with a broken report descriptor or unusual key
  mappings, extend an appropriate HID driver or add a HID driver under
  ``drivers/hid/``. Keep the USB transport in ``usbhid``. Also consider the
  mechanisms described in Documentation/hid/hid-bpf.rst for HID fixes.
* For a non-HID interface with a vendor-specific protocol, use the relevant
  subsystem, such as input, networking, or serial. A custom keyboard
  protocol still belongs in the input subsystem.
* If an interface is only used for application-specific control and needs
  no kernel subsystem integration, consider a userspace implementation
  using libusb. HID applications can use the interface described in
  Documentation/hid/hidraw.rst.

Do not replace the generic HID stack just to support a new keyboard ID.
The boot-protocol driver in ``drivers/hid/usbhid/usbkbd.c`` is a useful
example of USB interrupt transfers and input events, but it supports the
restricted boot keyboard protocol, rather than general HID reports.
Likewise, ``drivers/usb/usb-skeleton.c`` demonstrates bulk transfers and a
character-device interface, not a keyboard implementation.

Implement a HID-specific change
-------------------------------

Read Documentation/hid/hid-transport.rst to understand the distinction
between a HID device driver and its transport. Look at existing drivers
such as ``drivers/hid/hid-holtek-kbd.c`` for a keyboard descriptor fix.

A device-specific HID driver uses a ``struct hid_driver`` and a terminated
``struct hid_device_id`` table. For example, the following is a matching
fragment, not a complete driver::

    static const struct hid_device_id example_devices[] = {
        { HID_USB_DEVICE(EXAMPLE_VENDOR_ID, EXAMPLE_PRODUCT_ID) },
        { }
    };
    MODULE_DEVICE_TABLE(hid, example_devices);

Define the symbolic IDs using the actual device IDs, following the
conventions in ``drivers/hid/hid-ids.h``. Assign the table to the driver's
``id_table`` and register it with ``module_hid_driver()``. The module table
provides aliases for automatic module loading.

Use the HID callbacks appropriate to the problem: ``report_fixup`` for a
malformed descriptor, ``input_mapping`` for usage-to-key mappings, or
``raw_event`` when report processing is necessary. Check descriptor or
report sizes and identifying bytes before accessing or changing them;
firmware revisions and interfaces may have different formats.

If a custom ``probe`` is needed, the usual initialization calls are
``hid_parse()`` followed by ``hid_hw_start()`` with the required connection
flags. Propagate errors and unwind any resources acquired by the driver.
Pair successful hardware startup with ``hid_hw_stop()`` in a custom remove
path. Drivers that omit these callbacks can use the HID core's default
probe and remove handling. Let HID and input core handle standard key
reporting and LEDs where possible.

Implement a USB interface driver when needed
-------------------------------------------

For a non-HID protocol, read :doc:`writing_usb_driver` and study an
existing driver in the target subsystem. Obtain the protocol specification
or document observed transfers using Documentation/usb/usbmon.rst. USB
endpoint descriptors identify how to transfer bytes; they do not define
what those bytes mean.

A USB interface driver needs the following pieces:

1. A terminated ``struct usb_device_id`` table, a
   ``MODULE_DEVICE_TABLE(usb, ...)`` declaration, and a ``struct usb_driver``
   with ``probe``, ``disconnect``, and ``id_table``. Register it using
   ``module_usb_driver()``. Restrict matching to the supported interfaces
   of a composite device; a device-wide ID match may select unrelated
   interfaces. Validate the interface in ``probe`` as well.
2. A ``probe`` implementation that checks the interface and endpoints,
   allocates private state and transfer buffers, and registers with the
   target subsystem. Use endpoint descriptors and helpers such as
   ``usb_endpoint_is_int_in()`` instead of assuming endpoint numbers.
   Store per-interface state using ``usb_set_intfdata()`` and unwind all
   allocations on failure.
3. Transfer handling appropriate to the protocol. For an interrupt-IN
   endpoint, initialize an URB with ``usb_fill_int_urb()`` using the endpoint's
   interval, then submit it with ``usb_submit_urb()``. A completion callback
   must check the URB status and actual received length before decoding
   data. It must not sleep. Resubmit only while the device is running;
   cancellation and disconnect statuses must terminate the transfer loop.
4. Input integration for a custom keyboard: allocate an ``input_dev``, set
   its identity and supported key capabilities, then register it with
   ``input_register_device()``. Translate the device protocol into Linux
   key codes using ``input_report_key()``, and end each event packet with
   ``input_sync()``. Report releases as well as presses. Support LED output
   through the input subsystem if the protocol provides it. See
   Documentation/input/input-programming.rst.
5. Disconnect and power-management handling. Prevent new submissions,
   synchronously stop outstanding URBs with ``usb_kill_urb()`` or the
   corresponding anchored-URB helpers, and cancel deferred work before
   freeing state. Coordinate this with open, close, and output callbacks
   so none can access freed memory or restart I/O. Suspend must stop I/O;
   resume and reset paths must restore the device's required state.

Do not assume every keyboard sends an eight-byte boot-protocol report.
Report IDs, lengths, rollover behavior, and output commands depend on the
protocol. Treat all received bytes and descriptors as untrusted input.

Build and test
--------------

Add a configuration option and a Makefile entry in the chosen subsystem,
following nearby drivers. For example, a HID module named ``hid-example.c``
would have an entry like::

    obj-$(CONFIG_HID_EXAMPLE) += hid-example.o

Define ``HID_EXAMPLE`` in ``drivers/hid/Kconfig`` with the dependencies
required by the implementation. Enable it in a configured kernel tree,
then build the module, for example::

    make M=drivers/hid W=1

This requires a prepared build tree with the dependencies already built;
see Documentation/kbuild/modules.rst. Build and install the test kernel
and modules as described in
Documentation/admin-guide/quickly-build-trimmed-linux.rst, boot it, and load
the module with ``modprobe`` if it was built as a module. Keep another
keyboard or a remote console available when testing keyboard support.

Check the bound driver and kernel log, then test:

* Key presses and releases with ``evtest`` on the correct event device,
  including modifiers, simultaneous keys, repeat, and any special keys.
* Caps Lock and other supported LEDs, including repeated output changes.
* Unplugging while keys are held or output is pending, reconnecting,
  module unload and reload, and multiple devices connected at once.
* Suspend and resume, runtime power management if supported, and recovery
  after a USB reset. Confirm that no keys remain stuck and no kernel
  warnings or memory errors occur.

For HID changes, retain descriptor dumps and report captures that explain
both the failure and the corrected behavior. Test other interfaces and
supported models when modifying a shared driver.

Prepare the patch
-----------------

Follow Documentation/process/coding-style.rst and
Documentation/process/submitting-patches.rst. Describe the device, protocol
or descriptor problem, chosen subsystem, and testing performed. Run
``scripts/checkpatch.pl`` on the patch and use ``scripts/get_maintainer.pl``
to identify the relevant maintainers and lists. Include Kconfig, Makefile,
and device-table changes in the same submission as the driver.
