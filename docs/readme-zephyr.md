# Building and using MCUboot with Zephyr

MCUboot began its life as the bootloader for Mynewt.  It has since
acquired the ability to be used as a bootloader for Zephyr as well.
There are some pretty significant differences in how apps are built
for Zephyr, and these are documented here.

Please see the [design document](design.md) for documentation on the design
and operation of the bootloader itself. This functionality should be the same
on all supported RTOSs.

The first step required for Zephyr is making sure your board has flash
partitions defined in its device tree. These partitions are:

- `boot_partition`: for MCUboot itself
- `slot0_partition`: the primary slot of Image 0
- `slot1_partition`: the secondary slot of Image 0

It is not recommended to use the swap-using-scratch algorithm of MCUboot, but
if this operating mode is desired then the following flash partition is also
needed (see end of this help file for details on creating a scratch partition
and how to use the swap-using-scratch algorithm):

- `scratch_partition`: the scratch slot

Currently, the two image slots must be contiguous. If you are running
MCUboot as your stage 1 bootloader, `boot_partition` must be configured
so your SoC runs it out of reset. If there are multiple updateable images
then the corresponding primary and secondary partitions must be defined for
the rest of the images too (for example, `slot2_partition` and
`slot3_partition` for Image 1).

The flash partitions are typically defined in the Zephyr boards folder, in a
file named `boards/<arch>/<board>/<board>.dts`. An example `.dts` file with
flash partitions defined is the frdm_k64f's in
`boards/nxp/frdm_k64f/frdm_k64f.dts`. Make sure the DT node labels in your board's
`.dts` file match the ones used there.

## Installing requirements and dependencies

Install additional packages required for development with MCUboot:

```
  cd ~/mcuboot  # or to your directory where MCUboot is cloned
  pip3 install --user -r scripts/requirements.txt
```

## Building the bootloader itself

The bootloader is an ordinary Zephyr application, at least from
Zephyr's point of view.  There is a bit of configuration that needs to
be made before building it.  Most of this can be done as documented in
the `CMakeLists.txt` file in boot/zephyr.  There are comments there for
guidance.  It is important to select a signature algorithm, and decide
if the primary slot should be validated on every boot.

To build MCUboot, create a build directory in boot/zephyr, and build
it as usual:

```
  cd boot/zephyr
  west build -b <board>
```

In addition to the partitions defined in DTS, some additional
information about the flash layout is currently required to build
MCUboot itself. All the needed configuration is collected in
`boot/zephyr/include/target.h`. Depending on the board, this information
may come from board-specific headers, Device Tree, or be configured by
MCUboot on a per-SoC family basis.

After building the bootloader, the binaries should reside in
`build/zephyr/zephyr.{bin,hex,elf}`, where `build` is the build
directory you chose when running `west build`. Use `west flash`
to flash these binaries from the build directory. Depending
on the target and flash tool used, this might erase the whole of the flash
memory (mass erase) or only the sectors where the bootloader resides prior to
programming the bootloader image itself.

## Building applications for the bootloader

In addition to flash partitions in DTS, some additional configuration
is required to build applications for MCUboot.

This is handled internally by the Zephyr configuration system and is wrapped
in the `CONFIG_BOOTLOADER_MCUBOOT` Kconfig variable, which must be enabled in
the application's `prj.conf` file.

The directory `samples/zephyr/hello-world` in the MCUboot tree contains
a simple application with everything you need. You can try it on your
board and then just make a copy of it to get started on your own
application; see samples/zephyr/README.md for a tutorial.

The Zephyr `CONFIG_BOOTLOADER_MCUBOOT` configuration option
[documentation](https://docs.zephyrproject.org/latest/kconfig.html#CONFIG_BOOTLOADER_MCUBOOT)
provides additional details regarding the changes it makes to the image
placement and generation in order for an application to be bootable by MCUboot.

With this, build the application as your normally would.

### Signing the application

In order to upgrade to an image (or even boot it, if
`MCUBOOT_VALIDATE_PRIMARY_SLOT` is enabled), the images must be signed.
To make development easier, MCUboot is distributed with some example
keys.  It is important to stress that these should never be used for
production, since the private key is publicly available in this
repository.  See below on how to make your own signatures.

Images can be signed with the `scripts/imgtool.py` script.  It is best
to look at `samples/zephyr/Makefile` for examples on how to use this.

### Flashing the application

The application itself can flashed with regular flash tools, but will
need to be programmed at the offset of the primary slot for this particular
target. Depending on the platform and flash tool you might need to manually
specify a flash offset corresponding to the primary slot starting address. This
is usually not relevant for flash tools that use Intel Hex images (.hex) instead
of raw binary images (.bin) since the former include destination address
information. Additionally you will need to make sure that the flash tool does
not perform a mass erase (erasing the whole of the flash) or else you would be
deleting MCUboot.
These images can also be marked for upgrade, and loaded into the secondary slot,
at which point the bootloader should perform an upgrade.  It is up to
the image to mark the primary slot as "image ok" before the next reboot,
otherwise the bootloader will revert the application.

## Managing signing keys

The signing keys used by MCUboot are represented in standard formats,
and can be generated and processed using conventional tools.  However,
`scripts/imgtool.py` is able to generate key pairs in all of the
supported formats.  See [the docs](imgtool.md) for more details on
this tool.

### Generating a new keypair

Generating a keypair with imgtool is a matter of running the keygen
subcommand:

```
    $ ./scripts/imgtool.py keygen -k mykey.pem -t rsa-2048
```

The argument to `-t` should be the desired key type.  See the
[the docs](imgtool.md) for more details on the possible key types.

### Extracting the public key

The generated keypair above contains both the public and the private
key.  It is necessary to extract the public key and insert it into the
bootloader.  Use the ``CONFIG_BOOT_SIGNATURE_KEY_FILE`` Kconfig option to
provide the path to the key file so the build system can extract
the public key in a format usable by the C compiler.
The generated public key is saved in `build/zephyr/autogen-pubkey.h`, which is included
by the `boot/zephyr/keys.c`.

Currently, the Zephyr RTOS port limits its support to one keypair at the time,
although MCUboot's key management infrastructure supports multiple keypairs.

Once MCUboot is built, this new keypair file (`mykey.pem` in this
example) can be used to sign images.

## Using swap-using-scratch flash algorithm

To use the swap-using-scratch flash algorithm, a scratch partition needs to be
present for the target board which is used for holding the data being swapped
from both slots, this section must be at least as big as the largest sector
size of the 2 partitions (e.g. if a device has a primary slot in main flash
with a sector size of 512 bytes and secondar slot in external off-chip flash
with a sector size of 4KB then the scratch area must be at least 4KB in size).
The number of sectors must also be evenly divisable by this sector size, e.g.
4KB, 8KB, 12KB, 16KB are allowed, 7KB, 7.5KB are not. This scratch partition
needs adding to the .dts file for the board, e.g. for the nrf52dk_nrf52832
board thus would involve updating
`<zephyr>/boards/nordic/nrf52dk/nrf52dk_nrf52832.dts` with:

```
    boot_partition: partition@0 {
        label = "mcuboot";
        reg = <0x00000000 0xc000>;
    };
    slot0_partition: partition@c000 {
        label = "image-0";
        reg = <0x0000C000 0x37000>;
    };
    slot1_partition: partition@43000 {
        label = "image-1";
        reg = <0x00043000 0x37000>;
    };
    scratch_partition: partition@7a000 {
        label = "image-scratch";
        reg = <0x0007a000 0x00006000>;
    };
```

Which would make the application size 220KB and scratch size 24KB (the nRF52832
has a 4KB sector size so the size of the scratch partition can be reduced at
the cost of vastly reducing flash lifespan, e.g. for a 32KB firmware update
with an 8KB scratch area, the scratch area would be erased and programmed 8
times per image upgrade/revert). To configure MCUboot to work in
swap-using-scratch mode, the Kconfig value must be set when building it:
`CONFIG_BOOT_SWAP_USING_SCRATCH=y`.

Note that it is possible for an application to get into a stuck state when
swap-using-scratch is used whereby an application has loaded a firmware update
and marked it as test/confirmed but MCUboot will not swap the images and
erasing the secondary slot from the zephyr application returns an error
because the slot is marked for upgrade.

## Serial recovery

### Interface selection

A serial recovery protocol is available over either a hardware serial port or a USB CDC ACM virtual serial port.
The SMP server implementation can be enabled by the ``CONFIG_MCUBOOT_SERIAL=y`` Kconfig option.
To set a type of an interface, use the ``BOOT_SERIAL_DEVICE`` Kconfig choice, and select either the ``CONFIG_BOOT_SERIAL_UART`` or the ``CONFIG_BOOT_SERIAL_CDC_ACM`` value.
Which interface belongs to the protocol shall be set by the devicetree-chosen node:
- `zephyr,console` - If a hardware serial port is used.
- `zephyr,cdc-acm-uart` - If a virtual serial port is used.

### Entering the serial recovery mode

To enter the serial recovery mode, the device has to initiate rebooting, and a triggering event has to occur (for example, pressing a button).

By default, the serial recovery GPIO pin active state enters the serial recovery mode.
Use the ``mcuboot_button0`` devicetree button alias to assign the GPIO pin to the MCUboot.

Alternatively, MCUboot can wait for a limited time to check if DFU is invoked by receiving an MCUmgr command.
Select ``CONFIG_BOOT_SERIAL_WAIT_FOR_DFU=y`` to use this mode. ``CONFIG_BOOT_SERIAL_WAIT_FOR_DFU_TIMEOUT`` option defines
the amount of time in milliseconds the device will wait for the trigger.

### Direct image upload

By default, the SMP server implementation will only use the first slot.
To change it, invoke the `image upload` MCUmgr command with a selected image number, and make sure the ``CONFIG_MCUBOOT_SERIAL_DIRECT_IMAGE_UPLOAD=y`` Kconfig option is enabled.
Note that the ``CONFIG_UPDATEABLE_IMAGE_NUMBER`` Kconfig option adjusts the number of image-pairs supported by the MCUboot.

The mapping of image number to partition is as follows:
* 0 and 1 - image-0, the primary slot of the first image.
* 2 - image-1, the secondary slot of the first image.
* 3 - image-2.
* 4 - image-3.

0 is a default upload target when no explicit selection is done.

### System-specific commands

Use the ``CONFIG_ENABLE_MGMT_PERUSER=y`` Kconfig option to enable the following additional commands:
* Storage erase - This command allows erasing the storage partition (enable with ``CONFIG_BOOT_MGMT_CUSTOM_STORAGE_ERASE=y``).

### More configuration

For details on other available configuration options for the serial recovery protocol, check the Kconfig options  (for example by using ``menuconfig``).
