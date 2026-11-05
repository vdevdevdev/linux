.. SPDX-License-Identifier: GPL-2.0

==========================
Adding a new device driver
==========================

This document walks through the steps needed to add a new device driver to
the kernel tree, from choosing where the code lives to sending the patches.
It uses a small, fictional memory-mapped device called "foo" made by the
vendor "acme" as a running example, and links to the detailed documentation
for each topic rather than repeating it.

Before you start
================

* Check that a driver does not already exist.  Many devices are variants of
  existing hardware and only need a new ID table entry or a small amount of
  per-variant data in an existing driver.  Searching the tree for the vendor
  name, the chip name and the register names from the datasheet usually
  answers this quickly.
* Find the subsystem the device belongs to.  A temperature sensor is a hwmon
  or IIO driver, a GPIO expander is a GPIO driver, a network card is a
  netdev driver, and so on.  Writing to the subsystem's framework, rather
  than exposing a custom character device, gets you a stable user-space ABI
  and a lot of code for free.  The subsystem APIs are listed in
  Documentation/driver-api/index.rst.
* Read the subsystem's maintainer entry profile, if it has one (see
  Documentation/process/maintainer-handbooks.rst), and look at a few recent,
  well-reviewed drivers in the same directory.  Copying their structure is
  the fastest way to match what reviewers expect.

Understanding the driver model
==============================

Every driver binds to devices through a *bus*.  The bus knows how to
enumerate devices and how to match them against the ID tables of the
registered drivers.  When a match is found, the driver's ``probe()``
callback is called; when the device goes away or the driver is unbound,
``remove()`` is called.  The details are in
Documentation/driver-api/driver-model/index.rst.

Choose the driver type that matches how the device is connected:

================================  ==========================================
Device is connected via           Register a
================================  ==========================================
PCI / PCIe                        ``struct pci_driver``
USB                               ``struct usb_driver``
I2C                               ``struct i2c_driver``
SPI                               ``struct spi_driver``
Memory-mapped on an SoC, found    ``struct platform_driver``
through Device Tree or ACPI
================================  ==========================================

The example below is a platform driver, but the overall shape (ID table,
``probe()``, optional ``remove()``, registration macro) is the same for
every bus.

Where the code goes
===================

Drivers live under ``drivers/``, in the directory of the subsystem they
implement, e.g. ``drivers/hwmon/``, ``drivers/gpio/`` or
``drivers/net/ethernet/<vendor>/``.  A small driver is a single
``<subsystem>-<name>.c`` or ``<name>.c`` file, following the naming used by
its neighbours.  Larger drivers get their own sub-directory with their own
``Kconfig`` and ``Makefile``.

For the example, we add ``drivers/misc/acme-foo.c``.

Writing the driver
==================

A minimal platform driver looks like this::

  // SPDX-License-Identifier: GPL-2.0-only
  /*
   * Driver for the ACME FOO controller
   *
   * Copyright (C) 2026 Your Name <you@example.com>
   */

  #include <linux/clk.h>
  #include <linux/io.h>
  #include <linux/mod_devicetable.h>
  #include <linux/module.h>
  #include <linux/platform_device.h>
  #include <linux/pm.h>

  #define FOO_CTRL		0x00
  #define FOO_CTRL_ENABLE	BIT(0)

  struct foo_priv {
  	void __iomem *base;
  	struct clk *clk;
  };

  static int foo_probe(struct platform_device *pdev)
  {
  	struct device *dev = &pdev->dev;
  	struct foo_priv *priv;

  	priv = devm_kzalloc(dev, sizeof(*priv), GFP_KERNEL);
  	if (!priv)
  		return -ENOMEM;

  	priv->base = devm_platform_ioremap_resource(pdev, 0);
  	if (IS_ERR(priv->base))
  		return PTR_ERR(priv->base);

  	priv->clk = devm_clk_get_enabled(dev, NULL);
  	if (IS_ERR(priv->clk))
  		return dev_err_probe(dev, PTR_ERR(priv->clk),
  				     "failed to get clock\n");

  	platform_set_drvdata(pdev, priv);

  	writel(FOO_CTRL_ENABLE, priv->base + FOO_CTRL);

  	return 0;
  }

  static int foo_suspend(struct device *dev)
  {
  	struct foo_priv *priv = dev_get_drvdata(dev);

  	writel(0, priv->base + FOO_CTRL);
  	clk_disable_unprepare(priv->clk);

  	return 0;
  }

  static int foo_resume(struct device *dev)
  {
  	struct foo_priv *priv = dev_get_drvdata(dev);
  	int ret;

  	ret = clk_prepare_enable(priv->clk);
  	if (ret)
  		return ret;

  	writel(FOO_CTRL_ENABLE, priv->base + FOO_CTRL);

  	return 0;
  }

  static DEFINE_SIMPLE_DEV_PM_OPS(foo_pm_ops, foo_suspend, foo_resume);

  static const struct of_device_id foo_of_match[] = {
  	{ .compatible = "acme,foo" },
  	{ }
  };
  MODULE_DEVICE_TABLE(of, foo_of_match);

  static struct platform_driver foo_driver = {
  	.probe = foo_probe,
  	.driver = {
  		.name = "acme-foo",
  		.of_match_table = foo_of_match,
  		.pm = pm_sleep_ptr(&foo_pm_ops),
  	},
  };
  module_platform_driver(foo_driver);

  MODULE_AUTHOR("Your Name <you@example.com>");
  MODULE_DESCRIPTION("ACME FOO controller driver");
  MODULE_LICENSE("GPL");

Points worth noting:

License and copyright
  Every file starts with an SPDX license identifier; see
  Documentation/process/license-rules.rst.  ``MODULE_LICENSE("GPL")`` must
  agree with it, and ``MODULE_DESCRIPTION()`` is required, otherwise
  ``modpost`` warns.

ID tables
  The ID table (``of_device_id``, ``acpi_device_id``, ``pci_device_id``,
  ...) is what binds the driver to hardware.  ``MODULE_DEVICE_TABLE()``
  exports it so that user space can load the module automatically when the
  device appears.  Every table ends with an empty sentinel entry.  If
  variants of the device need different handling, attach a per-variant
  structure through the table's ``.data`` field and fetch it with
  ``device_get_match_data()`` instead of comparing compatible strings in
  ``probe()``.

Managed resources
  Prefer the ``devm_*()`` helpers for memory, MMIO mappings, clocks,
  regulators, GPIOs, IRQs and so on.  They are released automatically, in
  reverse order, when ``probe()`` fails or the device is unbound, which
  removes most error-handling code and most ``remove()`` callbacks.  See
  Documentation/driver-api/driver-model/devres.rst.  A ``remove()`` callback
  is only needed for teardown that devres does not cover.

Errors in ``probe()``
  Return a negative errno on failure.  Use ``dev_err_probe()`` when the
  error may be ``-EPROBE_DEFER`` (a supplier such as a clock or regulator is
  not ready yet): it stays silent for deferral, records the reason in
  debugfs, and returns the error code so it can be used directly in the
  ``return`` statement.

Logging
  Use ``dev_err()``, ``dev_warn()``, ``dev_info()`` and ``dev_dbg()`` so
  messages identify the device.  A successfully probed driver should
  normally be silent.

Hardware access
  Use the accessors described in Documentation/driver-api/device-io.rst
  (``readl()``, ``writel()``, ...) and never dereference ``__iomem``
  pointers directly.  For devices with many registers or behind I2C/SPI,
  consider ``regmap`` (``include/linux/regmap.h``), which also provides
  caching and debugfs register dumps.

Power management
  ``DEFINE_SIMPLE_DEV_PM_OPS()`` together with ``pm_sleep_ptr()`` lets the
  compiler drop the callbacks when ``CONFIG_PM_SLEEP`` is disabled without
  any ``#ifdef``.  Runtime PM is described in
  Documentation/power/runtime_pm.rst.

Concurrency
  ``probe()`` may run in parallel with other drivers' probes, and any
  interrupt handler, sysfs attribute or file operation you register can run
  as soon as it is registered.  Fully initialise the device and its private
  data before registering anything user-visible or requesting interrupts.

Coding style
  Follow Documentation/process/coding-style.rst.

Hooking into the build
======================

Add a Kconfig entry next to the related drivers in the directory's
``Kconfig`` file (see Documentation/kbuild/kconfig-language.rst)::

  config ACME_FOO
  	tristate "ACME FOO controller support"
  	depends on ARCH_ACME || COMPILE_TEST
  	depends on HAS_IOMEM
  	help
  	  Say Y here to enable support for the FOO controller found on
  	  ACME SoCs.

  	  To compile this driver as a module, choose M here: the
  	  module will be called acme-foo.

Use ``tristate`` so the driver can be built as a module, depend on the
platform or bus the hardware is found on so the option is not offered where
it is useless, and add ``|| COMPILE_TEST`` so the driver gets build coverage
on all architectures.  Use ``depends on`` for other subsystems; ``select``
should only be used for library-like symbols that have no dependencies of
their own.

Then add the object to the directory's ``Makefile`` (see
Documentation/kbuild/makefiles.rst)::

  obj-$(CONFIG_ACME_FOO)		+= acme-foo.o

Device Tree bindings
====================

If the device is described in Device Tree, a binding must document the
``compatible`` string and the properties the driver uses.  Bindings are
written in YAML schema under ``Documentation/devicetree/bindings/`` and are
sent as a separate patch, before the driver patch, to the Device Tree
maintainers as well as the subsystem maintainers.  Read
Documentation/devicetree/bindings/writing-bindings.rst and
Documentation/devicetree/bindings/writing-schema.rst, then validate the
binding with::

  make dt_binding_check DT_SCHEMA_FILES=acme,foo.yaml

If the vendor prefix (``acme`` here) is new, it must be added to
``Documentation/devicetree/bindings/vendor-prefixes.yaml``.

ACPI-enumerated devices use an ``acpi_device_id`` table instead; see
Documentation/firmware-guide/acpi/enumeration.rst.  Using the
``device_property_*()`` / ``fwnode`` API instead of ``of_*()`` calls lets a
single driver support both.

User-space interfaces and documentation
=======================================

* Any new sysfs attribute must be documented under ``Documentation/ABI/``;
  see Documentation/ABI/README.  New ioctls are covered in
  Documentation/driver-api/ioctl.rst.  Prefer the interfaces offered by the
  subsystem framework over new driver-specific ones.
* If the driver has user-visible behaviour, module parameters, or
  hardware-specific notes, document them in the subsystem's directory under
  ``Documentation/`` and add the file to the matching ``index.rst``.

Adding a MAINTAINERS entry
==========================

New drivers should come with an entry in the ``MAINTAINERS`` file, kept in
alphabetical order, that names who looks after the code and which files it
covers::

  ACME FOO CONTROLLER DRIVER
  M:	Your Name <you@example.com>
  L:	linux-kernel@vger.kernel.org
  S:	Maintained
  F:	Documentation/devicetree/bindings/misc/acme,foo.yaml
  F:	drivers/misc/acme-foo.c

The fields are described at the top of ``MAINTAINERS`` and in
Documentation/process/maintainers.rst.  Run
``./scripts/get_maintainer.pl --self-test=patterns`` to check that the
``F:`` patterns match existing files.

Building and testing
====================

Before sending anything:

* Build the driver both built-in and as a module, with ``W=1``, on your
  target architecture and, thanks to ``COMPILE_TEST``, on at least one
  other (for example a 64-bit and a 32-bit one)::

    make W=1 drivers/misc/acme-foo.o

* Run the static checkers described in Documentation/dev-tools/index.rst,
  such as ``sparse`` (``make C=1``) and ``smatch``.
* Test on real hardware, including probe failure and deferral, unbinding
  and rebinding through sysfs, module unload and reload, and suspend and
  resume.  Enabling ``CONFIG_DEBUG_KMEMLEAK``, ``CONFIG_KASAN``,
  ``CONFIG_PROVE_LOCKING`` and ``CONFIG_DEBUG_DEVRES`` while testing
  catches many common mistakes.
* Run ``./scripts/checkpatch.pl --strict`` on your patches (see
  Documentation/dev-tools/checkpatch.rst) and go through
  Documentation/process/submit-checklist.rst.

Submitting the driver
=====================

Split the work into a logical series, typically:

1. the Device Tree binding (if any),
2. the driver itself, with its Kconfig, Makefile and MAINTAINERS changes,
3. Device Tree source changes that add the device to board files, which
   usually go through the platform maintainers' tree.

The cover letter should say what the hardware is, where it is used and how
the driver was tested.  Use ``./scripts/get_maintainer.pl`` to find the
recipients, and follow Documentation/process/submitting-patches.rst for the
rest of the process, including the Developer Certificate of Origin.  Expect
review comments and several revisions; keep a changelog of what changed
between versions below the ``---`` line of each patch.

Further reading
===============

* Documentation/driver-api/driver-model/index.rst
* Documentation/driver-api/driver-model/devres.rst
* Documentation/driver-api/basics.rst
* Documentation/driver-api/infrastructure.rst
* Documentation/process/development-process.rst
* Documentation/process/submitting-patches.rst
