+++
title= "Debricking CORE 9G25: Understanding the boot process"
date= 2018-05-20T11:02:29+02:00
author = "Auke Willem Oosterhoff"
description = "First post about fixing a bricked CORE 9G25 using SAM-BA."
slug = "debricking-core-9g25-understanding-the-boot-process"
+++

At [ACS Buildings][acs] one of our products is build around [CORE
9G25][core9g25]. This System on a Module (SoM) has a 400 MHz ARM CPU
[AT91SAM9G25][at91sam9g25] from Atmel (which has been bought by Microchip, so
you might see also see both names on websites and datasheets). It also
contains 256 MB of RAM and a NAND flash of the same size. The following image
shows the SoM, taken from the website of [CoreWind][corewind].

{{<figure src="/img/core9g25.jpg">}}

The vendor doesn't ship the SoM with an empty NAND. It already contains 2
bootloaders, a Linux kernel and a root filesystem. Because these programs are
quite old we reflashed the NAND and ieplaced the stack with more recent
versions of the programs. Reflashing the NAND is dangerous and can lead to
bricked and therefor unusable devices. In a series of 3 posts I'll demonstrate
how to fix a bricked CORE 9G25. In this first post I'll explain how a normal
reflash procedure and boot procedure of the CORE 9G25 work. The
[second post]({{< relref "posts/debrick-core9g25-using-sam-ba-on-linux.md" >}})
will be about about reflashing the NAND using the tool SAM-BA on Linux. The
third post will be about the same thing, but then Windows is used to fix the
NAND.

## Reflash procedure
Reflashing the content on the NAND is easy when all goes well. During boot, the
bootloader U-Boot looks for an USB drive to be present. When it's found U-Boot
will look for files with specific names, load them into RAM before writing them
to specific segments of the NAND flash. So if you want to replace the Linux
kernel you have to make sure to put a file called `uImage` on the USB drive.
The snippet below shows the output of the U-Boot script that replaces the
content on the NAND.

```
1 Storage Device(s) found

########## Reading at91sam9x5ek-nandflashboot-uboot-3.5.3.bin ###########

#### Read 0x2FB0(12208) bytes ###
Erase NAND flash 0x0 0x40000 OK

########## Reading u-boot.bin ###########
##
#### Read 0x6BD04(441604) bytes ###
Erase NAND flash 0x40000 0x80000 OK

########## Reading at91sam9g25ek.dtb ###########

#### Read 0x5C42(23618) bytes ###
Erase NAND flash 0x180000 0x80000 OK

########## Reading uImage ###########
####################
#### Read 0x304FDA(3166170) bytes ###
Erase NAND flash 0x200000 0x600000 OK

########## Reading rootfs.img ###########
####################
#### Read 0xE40000(14942208) bytes ###
Erase NAND flash 0x800000 0xf800000 OK

### Update OK ### Please RESET the board ###
```

As you can see U-Boot updates the NAND using 5 files.

* `at91sam9x5ek-nandflashboot-uboot-3.5.3.bin` - the bootloader
[AT91Bootstrap][at91bootstrap] (which I've covered a while ago in the post [Compile AT91Bootstrap using Docker]({{< relref "posts/compile_at91bootstrap_using_docker.md" >}}))
* `u-boot.bin` - the bootloader U-Boot
* `at91sam9g25ek.dtb` - the device tree blob
* `uImage` - the Linux kernel
* `rootfs.img` - the root filesystem

First U-Boot loads the file `at91sam9x5ek-nandflashboot-uboot-3.5.3.bin`, th
it ereases a block of `0x40000` bytes on the NAND, starting at `0x0`. It then

writes the file to NAND, aligning the program at address `0x0`.
The next image is a graphical overview of the layout of the NAND, showing the
same addresses as used in the update script.

{{<figure src="/img/nand-layout.jpg">}}

## Risks
Because U-Boot performs the update, updating U-Boot is risky.  What happens if
the binaries for AT91Boostrap or U-Boot aren't compiled correctly? Or what if
the power is lost during the update?  Or what if you've replaced the original
U-Boot with a version that doesn't contain the update script (yes, I've done
that)? Well, your devices are bricked.

Recently my college Marten van Houten and I managed to reflash the NAND using
tools provided by the vendor of the CPU.

In order to fully understand how we did that I'll explain the boot
process of AT91SAM9G25 CPU.

## Boot process
The following snippet is taken from chapter 10 'Boot strategies' of [AT91SAM9G25's
datasheet][datasheet] and it explains the boot process:

> The system always boots at address `0x0`. To ensure maximum boot
> possibilities, the memory layout can be changed with the BMS pin. This allows
> the user to lay out the ROM or an external memory to `0x0`. The sampling of
> the BMS pin is done at reset. If BMS is detected at `0`, the controller boots
> on the memory connected to Chip Select `0` of the External Bus.
>
> [...]
>
> If BMS is detected at `1`, the boot memory is the embedded ROM and the Boot
> Program described below is executed.

Upon receiving power the CPU starts the program at address `0x0`.  Normally,
when the BMS pin is `0`, the NAND is laid out to this address. So
when the CPU starts the program that is located at `0x0` on the NAND.

And what program is at this address? Well, that is AT91Bootstrap.

## SAM-BA to the rescue
But if you set the BMS pin to `1`, the memory lay out is modified.
Instead of the NAND, the ROM is now laid out at address `0x0`. The program
that is found on `0x0` on the ROM will be executed.

The AT91SAM9G25 CPU comes with a 64 kB ROM and contains a program called
[SAM boot agent (SAM-BA)][sam-ba]. This program allows one to reprogram the
NAND flash.

In the next posts post I'll demonstrate how to do this. The
series = ["Debrick CORE 9G25"]
[second post]({{< relref "posts/debrick-core9g25-using-sam-ba-on-linux.md" >}})
of this series will be about reflashing the NAND using a Linux machine. In the
third post I'll show how to use a Windows machine to debrick the CORE 9G25.

[acs]: https://www.acs-buildings.com/
[corewind]: http://www.armdevs.com/picture/CORE%209G25.html
[at91bootstrap]: https://github.com/linux4sam/at91bootstrap
[core9g25]:http://www.armdevs.com/CORE%209G25.html
[at91sam9g25]: http://www.microchip.com/wwwproducts/en/AT91sam9g25
[datasheet]: http://ww1.microchip.com/downloads/en/DeviceDoc/Atmel-11032-32-bit-ARM926EJ-S-Microcontroller-SAM9G25_Datasheet.pdf
[sam-ba]: http://www.microchip.com/DevelopmentTools/ProductDetails.aspx?PartNO=Atmel%20SAM-BA%20In-system%20Programmer
