+++
title= "Debricking CORE 9G25: Intro"
date= 2018-05-20T11:02:29+02:00
series = ["Unbrick CORE 9G25"]
author = "Auke Willem Oosterhoff"
draft= true
+++

At [ACS Buildings][acs] one of our products is build around [CORE
9G25][core9g25]. The heart of this System on a Module is the 400 MHz ARM CPU
[AT91SAM9G25][at91sam9g25] from Atmel. It also contains 256 MB of RAM and a
Nand flash of the same size.

The SoM is shipped with 2 bootloaders, a Linux kernel and a root filesystem
on the Nand. Because this software is quite old we reflash the Nand and replace
the stack with more recent firmware. Reflashing the Nand wrongly can lead to
bricked and unusable devices. In a series of 3 posts I'll demonstrate how to
fix a bricked CORE 9G25. In this post I'll explain how normal reflash procudere
and boot procedure of the CORE 9G25 work. In the next 2 posts I'll explain how
to unbrick the module using SAM-BA monitor on [Linux]({{< relref
"post/debrick_core9g25_using_sam_ba_on_linux.md" >}}) or [Windows]({{< relref
"post/debrick_core9g25_using_sam_ba_on_windows.md" >}}).

## Reflash procedure
Reflashing the content on the Nand can be easy.  During boot, the bootloader
U-boot checks if an USB device is present. If so, it'll look for certain files
on that USB drive to write to the Nand. If you want to replace the Linux
kernel, for example, you've to put a file called `uImage` on the USB device. In
the snippet below you can see how U-boot replaces everything on the Nand.

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

First U-Boot loads the file `at91sam9x5ek-nandflashboot-uboot-3.5.3.bin`, than
it ereases a block of `0x40000` bytes, starting at `0x0`. It then writes the
file to Nand, aligning the program at address `0x0`. The file
`at91sam9x5ek-nandflashboot-uboot-3.5.3.bin` contains the bootloader
[AT91Bootstrap][at91bootstrap].

The other files are flashed the same way, only their addresses are different.

## Risks
Because U-Boot performs the update, replacing the kernel or the root filesystem
like this can be done without risks. When it goes wrong you just try it another
time.

But what if you want to update U-Boot itself? That is a risky operation. When
it goes wrong the device is unable to boot and it is bricked. Over the years
I've bricked a few dozen CORE 9G25's. Only recently I discovered how to unbrick
a the CORE 9G25. In order to understand fully how I managed to do that I'll
explain the boot process of AT91SAM9G25 CPU.

## Boot process
The following snippet is taken from chapter 10 Boot strategies of [SAM9G25's
datasheet][datasheet] and explains the boot process:

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

Upon receiving power the CPU starts the program at address `0x0`.
Normally, that is when the BMS pin is `0`, the Nand is layed out to this address.
So the CPU starts the program that is located at `0x0` on the Nand.

And which program is at this address? Well, according to the output of the
update mechanism that is `at91sam9x5ek-nandflashboot-uboot-3.5.3.bin`, which
is the bootloader AT91Boostrap.

## SAM-Ba to the rescue
if your cannot boot from your Nand anymore, you can enable the BMS pin. Now the
CPU starts the program that is located at address `0x0` on the ROM. And that
program is SAM-BA. SAM-BA allows users to reprogram the Nand en thus debricking
Nand using the a Linux or Windows program which is also called
[SAM-BA][sam-ba].

You can find the instructions on how to reflash your nand in the next posts.

[acs]: https://www.acs-buildings.com/
[at91bootstrap]: https://github.com/linux4sam/at91bootstrap
[core9g25]:http://www.armdevs.com/CORE%209G25.html
[at91sam9g25]: http://www.microchip.com/wwwproducts/en/AT91sam9g25
[datasheet]: http://ww1.microchip.com/downloads/en/DeviceDoc/Atmel-11032-32-bit-ARM926EJ-S-Microcontroller-SAM9G25_Datasheet.pdf
[sam-ba]: http://www.microchip.com/DevelopmentTools/ProductDetails.aspx?PartNO=Atmel%20SAM-BA%20In-system%20Programmer
