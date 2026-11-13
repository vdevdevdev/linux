.. SPDX-License-Identifier: GPL-2.0

.. _adding-usb-device-driver:

====================================
Adding a Driver for a New USB Device
====================================

This document walks through the steps for adding kernel support for a new
USB device, using a USB mouse as the running example. It is meant as a
practical roadmap: it helps you decide *which kind* of driver you need,
shows a minimal implementation of each kind, and lists what has to be
touched in the tree (ids, Kconfig, Makefile) to get the driver built,
loaded and merged.

For a detailed walk-through of a generic USB driver, see
:ref:`writing-usb-driver` and ``drivers/usb/usb-skeleton.c``.

Step 1: Find out what the device is
===================================

Before writing any code, find out how the device describes itself. Plug it
in and run::

    $ lsusb
    Bus 001 Device 007: ID 1234:5678 Example Corp. Example Mouse

    $ lsusb -v -d 1234:5678

The first command gives the vendor ID (``1234``) and product ID (``5678``).
The second dumps the device, configuration, interface and endpoint
descriptors. For a mouse, the interesting part looks like this::

    Interface Descriptor:
      bInterfaceNumber        0
      bNumEndpoints           1
      bInterfaceClass         3 Human Interface Device
      bInterfaceSubClass      1 Boot Interface Subclass
      bInterfaceProtocol      2 Mouse
      Endpoint Descriptor:
        bEndpointAddress     0x81  EP 1 IN
        bmAttributes            3
          Transfer Type            Interrupt
        wMaxPacketSize     0x0008  1x 8 bytes
        bInterval              10

Also check the kernel log (``dmesg``) to see whether some driver already
binds to the device, and look in ``/sys/bus/usb/devices/`` and
``/sys/bus/hid/devices/``. The debugfs files under
``/sys/kernel/debug/hid/<device>/`` show the parsed report descriptor
(``rdesc``) and decoded live events (``events``), which is invaluable when
debugging HID devices.

Step 2: Pick the right kind of driver
=====================================

Almost every mouse, keyboard, joystick, touchpad or similar input device
uses the USB *Human Interface Device* (HID) class (``bInterfaceClass 3``).
Such devices are already handled by the generic ``usbhid`` transport
driver together with ``hid-generic``, which parse the device's *report
descriptor* and create input devices automatically. **A standard USB mouse
needs no new driver at all.**

Use this decision list, cheapest option first:

1. **The device works, but something is slightly off** (wrong button
   count, needs to be polled continuously, hangs on a ``GET_REPORT``
   request, ...): add a *quirk* to ``drivers/hid/hid-quirks.c``. See
   `Option A: add a HID quirk`_.

2. **The device is HID, but its report descriptor is broken, or it sends
   some reports that need special handling**: write a small *HID driver*
   in ``drivers/hid/``. See `Option B: write a HID driver`_.
   For descriptor-only fixes, HID-BPF (Documentation/hid/hid-bpf.rst) can
   also be used without writing a kernel module.

3. **The device is not HID at all** (``bInterfaceClass 0xff``, vendor
   specific) and speaks its own protocol over bulk or interrupt endpoints:
   write a *USB interface driver*. See `Option C: write a USB driver`_.

Options A and B are by far the most common for input devices; new raw USB
drivers for HID-class devices are generally not accepted, since they
duplicate what the HID core already does.

Option A: add a HID quirk
=========================

Quirks are flags attached to a vendor/product pair that tweak how the HID
core treats a device. They are listed in ``include/linux/hid.h``
(``HID_QUIRK_*``). Common ones for mice include:

``HID_QUIRK_ALWAYS_POLL``
    Keep polling the interrupt endpoint even when nobody has the device
    open. Needed by some wireless receivers that otherwise disconnect.

``HID_QUIRK_NOGET``
    Do not issue ``GET_REPORT`` requests during initialization; some
    devices stall or crash on them.

``HID_QUIRK_MULTI_INPUT``
    Create a separate input device for each report.

First test a quirk without rebuilding the kernel, using the ``quirks``
parameter of ``usbhid`` (values are hex; the quirk value is the bit of the
flag in ``include/linux/hid.h``)::

    # modprobe usbhid quirks=0x1234:0x5678:0x00000400

or on the kernel command line, ``usbhid.quirks=0x1234:0x5678:0x00000400``.

Once the right quirk is known, make it permanent:

1. Add the vendor and product IDs to ``drivers/hid/hid-ids.h``, keeping
   the file sorted::

       #define USB_VENDOR_ID_EXAMPLE		0x1234
       #define USB_DEVICE_ID_EXAMPLE_MOUSE	0x5678

2. Add an entry to the ``hid_quirks[]`` table in
   ``drivers/hid/hid-quirks.c``, again in sorted order::

       { HID_USB_DEVICE(USB_VENDOR_ID_EXAMPLE, USB_DEVICE_ID_EXAMPLE_MOUSE),
         HID_QUIRK_ALWAYS_POLL },

Option B: write a HID driver
============================

A HID driver is a ``struct hid_driver`` registered on the HID bus. It
binds to devices by vendor/product ID and only overrides the parts of the
HID core's behaviour it needs to; everything else (USB transfers, report
parsing, input device creation) is still done by the core. Specific HID
drivers take precedence over ``hid-generic`` automatically.

The most useful callbacks, all defined in ``include/linux/hid.h``, are:

``report_fixup``
    Called with the raw report descriptor before it is parsed. Return a
    corrected descriptor. This is the usual fix for mice that report the
    wrong number of buttons or wrong axis ranges.

``input_mapping`` / ``input_mapped``
    Change how individual HID usages are mapped to input event codes.

``raw_event``
    Called for each incoming report with the raw bytes, before the HID core
    parses it. Return 0 to let the core continue processing, 1 to tell it
    the report was fully handled, or a negative error code.

``event``
    Called for each parsed usage value.

``probe`` / ``remove``
    Set up and tear down driver state. If omitted, the core performs the
    default ``hid_parse()`` + ``hid_hw_start(hdev, HID_CONNECT_DEFAULT)`` on
    probe and ``hid_hw_stop()`` on remove.

A minimal driver for a mouse whose report descriptor claims three buttons
while the hardware has five could look like this
(``drivers/hid/hid-example.c``)::

    // SPDX-License-Identifier: GPL-2.0
    /*
     * HID driver for the Example Corp. Example Mouse
     */

    #include <linux/hid.h>
    #include <linux/module.h>

    #include "hid-ids.h"

    #define EXAMPLE_RDESC_SIZE	52
    #define EXAMPLE_USAGE_MAX_OFF	17	/* offset of Usage Maximum value */

    static const __u8 *example_report_fixup(struct hid_device *hdev,
                                            __u8 *rdesc, unsigned int *rsize)
    {
            /* Usage Maximum (3) -> Usage Maximum (5) */
            if (*rsize == EXAMPLE_RDESC_SIZE &&
                rdesc[EXAMPLE_USAGE_MAX_OFF - 1] == 0x29 &&
                rdesc[EXAMPLE_USAGE_MAX_OFF] == 0x03) {
                    hid_info(hdev, "fixing up button count\n");
                    rdesc[EXAMPLE_USAGE_MAX_OFF] = 0x05;
            }

            return rdesc;
    }

    static int example_probe(struct hid_device *hdev,
                             const struct hid_device_id *id)
    {
            int ret;

            ret = hid_parse(hdev);
            if (ret) {
                    hid_err(hdev, "parse failed\n");
                    return ret;
            }

            ret = hid_hw_start(hdev, HID_CONNECT_DEFAULT);
            if (ret) {
                    hid_err(hdev, "hw start failed\n");
                    return ret;
            }

            return 0;
    }

    static void example_remove(struct hid_device *hdev)
    {
            hid_hw_stop(hdev);
    }

    static const struct hid_device_id example_devices[] = {
            { HID_USB_DEVICE(USB_VENDOR_ID_EXAMPLE,
                             USB_DEVICE_ID_EXAMPLE_MOUSE) },
            { }
    };
    MODULE_DEVICE_TABLE(hid, example_devices);

    static struct hid_driver example_driver = {
            .name         = "example",
            .id_table     = example_devices,
            .report_fixup = example_report_fixup,
            .probe        = example_probe,
            .remove       = example_remove,
    };
    module_hid_driver(example_driver);

    MODULE_DESCRIPTION("HID driver for the Example Corp. Example Mouse");
    MODULE_LICENSE("GPL");

The ``probe`` and ``remove`` callbacks above do exactly what the core does
by default, so they could be left out; they are shown because real drivers
usually need to allocate private data (with ``devm_kzalloc(&hdev->dev,
...)`` and ``hid_set_drvdata()``) or talk to the device before or after
``hid_hw_start()``.

Note that ``report_fixup`` must return a buffer that stays valid for the
lifetime of the device: either the ``rdesc`` buffer passed in (possibly
modified in place, as above), a ``static const`` array holding a complete
replacement descriptor, or memory managed by the driver. Always check the
original descriptor's size and contents before patching it, so that other
firmware revisions are left alone. ``drivers/hid/hid-xiaomi.c`` is a
complete, short real-world example of a mouse descriptor fix.

To hook the driver into the build:

1. Add the IDs to ``drivers/hid/hid-ids.h`` (as in Option A).

2. Add a Kconfig entry to ``drivers/hid/Kconfig``, in alphabetical order
   inside the "Special HID drivers" menu::

       config HID_EXAMPLE
       	tristate "Example Corp. mice"
       	help
       	  Support for Example Corp. Example Mouse, whose report
       	  descriptor reports too few buttons.

3. Add the object to ``drivers/hid/Makefile``::

       obj-$(CONFIG_HID_EXAMPLE)	+= hid-example.o

Option C: write a USB driver
============================

If the device does not implement a standard class, the driver talks to it
directly through the USB core. Such a driver is a ``struct usb_driver``
that binds to a USB *interface* (not to the whole device).

``drivers/hid/usbhid/usbmouse.c`` is a compact, complete example of this
style. It drives boot-protocol mice without going through the HID parser
(it exists for tiny embedded systems; normal systems use ``usbhid``), and
its structure is the same as a driver for a vendor-specific mouse would
have. The essential pieces are:

**The device ID table** tells the USB core which interfaces the driver
handles, and ``MODULE_DEVICE_TABLE()`` exports it so udev can load the
module automatically when the device is plugged in::

    static const struct usb_device_id example_id_table[] = {
            { USB_DEVICE_INTERFACE_CLASS(0x1234, 0x5678,
                                         USB_CLASS_VENDOR_SPEC) },
            { }
    };
    MODULE_DEVICE_TABLE(usb, example_id_table);

Other match macros (``USB_DEVICE()``, ``USB_INTERFACE_INFO()``,
``USB_DEVICE_AND_INTERFACE_INFO()``, ...) are defined in
``include/linux/usb.h``.

**probe()** is called for each matching interface. It must validate the
descriptors (never trust a device to have the endpoints you expect),
allocate state, set up the input device and the URB used for receiving
data, and register with the input subsystem::

    static int example_probe(struct usb_interface *intf,
                             const struct usb_device_id *id)
    {
            struct usb_device *udev = interface_to_usbdev(intf);
            struct usb_endpoint_descriptor *ep;
            struct example_mouse *mouse;
            struct input_dev *input;
            size_t len;
            int error;

            error = usb_find_int_in_endpoint(intf->cur_altsetting, &ep);
            if (error)
                    return error;

            mouse = kzalloc(sizeof(*mouse), GFP_KERNEL);
            input = input_allocate_device();
            ...
            mouse->data = usb_alloc_coherent(udev, EXAMPLE_PKT_LEN,
                                             GFP_KERNEL, &mouse->data_dma);
            mouse->irq = usb_alloc_urb(0, GFP_KERNEL);
            ...
            usb_make_path(udev, mouse->phys, sizeof(mouse->phys));
            len = strnlen(mouse->phys, sizeof(mouse->phys));
            strscpy(mouse->phys + len, "/input0", sizeof(mouse->phys) - len);

            input->name = "Example Mouse";
            input->phys = mouse->phys;
            usb_to_input_id(udev, &input->id);
            input->dev.parent = &intf->dev;

            input_set_capability(input, EV_KEY, BTN_LEFT);
            input_set_capability(input, EV_KEY, BTN_RIGHT);
            input_set_capability(input, EV_KEY, BTN_MIDDLE);
            input_set_capability(input, EV_REL, REL_X);
            input_set_capability(input, EV_REL, REL_Y);
            input_set_capability(input, EV_REL, REL_WHEEL);

            input_set_drvdata(input, mouse);
            input->open = example_open;
            input->close = example_close;

            usb_fill_int_urb(mouse->irq, udev,
                             usb_rcvintpipe(udev, ep->bEndpointAddress),
                             mouse->data, EXAMPLE_PKT_LEN,
                             example_irq, mouse, ep->bInterval);
            mouse->irq->transfer_dma = mouse->data_dma;
            mouse->irq->transfer_flags |= URB_NO_TRANSFER_DMA_MAP;

            error = input_register_device(input);
            if (error)
                    goto err_free_urb;

            usb_set_intfdata(intf, mouse);
            return 0;
            ...
    }

**open() and close()** of the input device start and stop the interrupt
URB, so the device is only polled while someone is listening::

    static int example_open(struct input_dev *input)
    {
            struct example_mouse *mouse = input_get_drvdata(input);

            return usb_submit_urb(mouse->irq, GFP_KERNEL) ? -EIO : 0;
    }

    static void example_close(struct input_dev *input)
    {
            struct example_mouse *mouse = input_get_drvdata(input);

            usb_kill_urb(mouse->irq);
    }

**The URB completion handler** runs in interrupt context each time a
packet arrives. It decodes the packet, reports input events, and
resubmits the URB with ``GFP_ATOMIC``. The packet layout is whatever the
device's protocol defines; here it is the common boot-protocol layout::

    static void example_irq(struct urb *urb)
    {
            struct example_mouse *mouse = urb->context;
            struct input_dev *input = mouse->input;
            s8 *data = mouse->data;

            switch (urb->status) {
            case 0:
                    break;
            case -ECONNRESET:   /* URB was killed */
            case -ENOENT:
            case -ESHUTDOWN:
                    return;
            default:            /* transient error, try again */
                    goto resubmit;
            }

            if (urb->actual_length < 4)
                    goto resubmit;

            input_report_key(input, BTN_LEFT,   data[0] & 0x01);
            input_report_key(input, BTN_RIGHT,  data[0] & 0x02);
            input_report_key(input, BTN_MIDDLE, data[0] & 0x04);
            input_report_rel(input, REL_X,      data[1]);
            input_report_rel(input, REL_Y,      data[2]);
            input_report_rel(input, REL_WHEEL,  data[3]);
            input_sync(input);

    resubmit:
            if (usb_submit_urb(urb, GFP_ATOMIC))
                    dev_err(&mouse->udev->dev, "can't resubmit URB\n");
    }

See Documentation/input/input-programming.rst for the input subsystem API
and Documentation/input/event-codes.rst for which event codes a mouse
should report.

**disconnect()** is called when the device is unplugged or the driver is
unbound. It must stop all I/O and free everything allocated in probe::

    static void example_disconnect(struct usb_interface *intf)
    {
            struct example_mouse *mouse = usb_get_intfdata(intf);

            usb_kill_urb(mouse->irq);
            input_unregister_device(mouse->input);
            usb_free_urb(mouse->irq);
            usb_free_coherent(mouse->udev, EXAMPLE_PKT_LEN,
                              mouse->data, mouse->data_dma);
            kfree(mouse);
    }

**Registration** ties everything together::

    static struct usb_driver example_driver = {
            .name       = "example_mouse",
            .probe      = example_probe,
            .disconnect = example_disconnect,
            .id_table   = example_id_table,
    };
    module_usb_driver(example_driver);

    MODULE_DESCRIPTION("Example Corp. vendor-specific mouse driver");
    MODULE_LICENSE("GPL");

Add a Kconfig entry and Makefile line in the matching subsystem directory
(for a mouse, ``drivers/input/mouse/``), depending on ``USB`` and
``INPUT``. If a HID-class driver such as ``usbhid`` would also match the
interface, the device must be added to ``hid_ignore_list[]`` in
``drivers/hid/hid-quirks.c`` so the two drivers do not compete.

Things to keep in mind:

* The device is untrusted input. Check endpoint types, descriptor sizes
  and ``urb->actual_length`` before using the data.
* Buffers used for USB transfers must be DMA-capable: allocate them with
  ``kmalloc()`` or ``usb_alloc_coherent()``, never on the stack or as
  static data.
* The device can disappear at any time. After ``disconnect()`` returns,
  the driver must not touch the device again; make sure no URB is still in
  flight and no work item is pending.
* Consider power management (``suspend``/``resume``), see
  Documentation/driver-api/usb/power-management.rst.

Step 3: Build, load and test
============================

Enable the new option (``make menuconfig``, or ``scripts/config -m
HID_EXAMPLE``), then build and install just the module::

    $ make M=drivers/hid
    # insmod drivers/hid/hid-example.ko

If the device is already bound to another driver, re-plug it, or unbind
and rebind it through sysfs (``/sys/bus/hid/drivers/*/unbind`` or
``/sys/bus/usb/drivers/*/unbind``). Then check that:

* ``dmesg`` shows the new driver binding to the device;
* ``evtest`` (or ``libinput debug-events``) shows the expected events when
  moving the mouse and pressing every button;
* unplugging and replugging, and unloading the module while the device is
  in use, do not produce warnings or crashes. Building with ``KASAN`` and
  ``PROVE_LOCKING`` enabled helps catch bugs here.

Without real hardware, a HID device can be emulated from user space with
``uhid`` (Documentation/hid/uhid.rst), which is also what the HID selftests
in ``tools/testing/selftests/hid/`` use.

Step 4: Submit the patch
========================

Run ``scripts/checkpatch.pl`` on the patch and find the maintainers and
mailing lists with ``scripts/get_maintainer.pl``. HID drivers and quirks go
to the HID maintainers and ``linux-input@vger.kernel.org``; input drivers
in ``drivers/input/`` go to the input maintainer on the same list; other
USB drivers go to ``linux-usb@vger.kernel.org``. Include the ``lsusb -v``
output and, for HID devices, the report descriptor (from debugfs
``rdesc``) in the commit message or cover letter, so reviewers can check
the fix. See Documentation/process/submitting-patches.rst for the general
process.

Further reading
===============

* :ref:`writing-usb-driver` and ``drivers/usb/usb-skeleton.c``
* Documentation/driver-api/usb/usb.rst -- the USB core API
* Documentation/driver-api/usb/URB.rst -- USB Request Blocks
* Documentation/hid/hidintro.rst -- introduction to HID and report
  descriptors
* Documentation/hid/hid-transport.rst -- the HID core and transport drivers
* Documentation/hid/hid-bpf.rst -- fixing HID devices with BPF
* Documentation/input/input-programming.rst -- the input subsystem
