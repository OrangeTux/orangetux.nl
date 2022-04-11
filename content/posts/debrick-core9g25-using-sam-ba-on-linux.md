+++
title= "Debricking CORE 9G25: Using SAM-BA on Linux"
date= 2018-05-31T11:02:29+02:00
author = "Auke Willem Oosterhoff"
description = "Fix a bricked CORE 9G25 using SAM-BA on Linux."
slug = "debricking-core-9g25-understanding-using-sam-ba-on-linux"
+++

In this post I'll explain how to debrick the CORE 9G25 on a Linux machine. This
post is the second in a series of series posts. I highly recommend to read the
[previous post]({{< relref "posts/debrick-core9g25-intro.md" >}}) before
continuing. In that post I explained how the boot process works, how the
update procedure works and the layout of the NAND. This post will be about
debricking the NAND using SAM-BA on a Linux machine.

## SAM-BA
In the previous post I explained that the ROM code of the CPU contains a
program called SAM Boot Agent (SAM-BA). This program can be used to reflash
the NAND.

In order to interact with this program in ROM from your Linux machine you need
a tool. This tool is also called SAM-BA and you can download it from
[Microchip's][sam-ba] website. For this post I used SAM-BA 3.2.1. If I refer to
SAM-BA in this post I mean the Linux program unless stated otherwise.

## Applets
SAM-BA comes with little programs, called applets, to interact with external
flash or memory. SAM-BA supports different pieces of hardware. The applets
available depends on the board. You can list all supported boards by running
`sam-ba -d <board> -a help`.

``` bash
$ ./sam-ba -b sam9xx5-ek -a help
No port option on command-line, using 'serial'.
Known applets: lowlevel, extram, sdmmc, serialflash, nandflash
```

An applet can be loaded using the `-a` flag and most applets will print an help
text if you run the applet with the `help` parameter. Below you find the
documentation of the `nandflash` applet.

```bash
$ ./sam-ba -b sam9xx5-ek -a nandflash:help
Syntax: nandflash:[<ioset>]:[<bus_width>]:[<header>]
Parameters:
    ioset      I/O set
    bus_width  NAND bus width (8/16)
    header     NAND header value
Examples:
    nandflash                 use default board settings
    nandflash:2:8:0xc0098da5  use fully custom settings (IOSET2, 8-bit bus, header is 0xc0098da5)
    nandflash:::0xc0098da5    use default board settings but force header to 0xc0098da5
For information on NAND header values, please refer to SAM9xx5 datasheet section "10.4.4 Detailed Memory Boot Procedures".
```

## Commands
Most applets can be used in conjunction with zero or more commands using the
`-c` flag. To read the first 1024 (or 0x400 hexadecimal) bytes of the NAND, for
example, you can use the `read` command like below.

``` bash
$ ./sam-ba -p serial -b sam9xx5-ek -a nandflash -c read:file.bin:0x0:0x400
```

All supported commands of an applet can be listed by issueing the `help`
command. The next snippet shows the command that implemented by the `nandflash`
applet.

``` bash
$ ./sam-ba -b sam9xx5-ek -a nandflash -c help
No port option on command-line, using 'serial'.
* erase - erase all or part of the memory
Syntax:
    erase:[<addr>]:[<length>]
Examples:
    erase                 erase all
    erase:4096            erase from 4096 to end
    erase:0x1000:0x10000  erase from 0x1000 to 0x11000
    erase::0x1000         erase from 0 to 0x1000

* read - read from memory to a file
Syntax:
    read:<filename>:[<addr>]:[<length>]
Examples:
    read:firmware.bin              read all to firmware.bin
    read:firmware.bin:0x1000       read from 0x1000 to end into firmware.bin
    read:firmware.bin:0x1000:1024  read 1024 bytes from 0x1000 into firmware.bin
    read:firmware.bin::1024        read 1024 bytes from start of memory into firmware.bin

* verify - verify memory from a file
Syntax:
    verify:<filename>:[<addr>]
Examples:
    verify:firmware.bin         verify that start of memory matches firmware.bin
    verify:firmware.bin:0x1000  verify that memory at offset 0x1000 matches firmware.bin

* verifyboot - verify boot program from a file
Syntax:
    verifyboot:<filename>
Example:
    verify:firmware.bin  verify that start of memory matches boot program firmware.bin

* write - write to memory from a file
Syntax:
    write:<filename>:[<addr>]
Examples:
    write:firmware.bin         write firmware.bin to start of memory
    write:firmware.bin:0x1000  write firmware.bin at offset 0x1000

* writeboot - write boot program to memory from a file
Syntax:
    writeboot:<filename>
Example:
    writeboot:firmware.bin  write firmware.bin as boot program at start of memory
```

## Serial connection
Connect your Linux machine to the serial debug port of the device. I've used
[this USB-to-serial cable][usb-to-serial].

Before I continue I often use Minicom to validate if the serial connection is
working. Start Minicom and configure the serial port with the settings below.
You can open the settings menu by pressing `Ctrl-a` and then `o` (the letter,
not the number).

* baudrate: 115200
* parity: none
* bytesize: 8
* stopbits: 1

{{<figure src="/img/configure_minicom.png">}}

Reboot the CPU. When you see human readable data in Minicom the serial port
has been configured correctly. You probably should at least see the string
"romboot".

If you don't see anything, you probably need to switch your Rx and Tx lines.

When Minicom prints garbage it's likely that the settings on either end of the
serial line doesn't match. You probably have to change Minicom's configuration.

## Boot from ROM
If your serial connection has been set up it's time to fix debrick the NAND.
The ROM of the AT91SAM9G25 contains a program which is also called SAM-BA,
SAM-BA on your Linux machine interacts with SAM-BA on the ROM via the serial
connection.

In order to boot from ROM the BMS pin must be `1`. The BMS pin can be brought
`1` by shortening the marked 2 islands on the picture below.

{{<figure src="/img/core9g25_with_marked_bms.jpg">}}

Before you continue make sure that you've closed Minicom! Only 1 program at the
time can use serial line.

## Initialize RAM
Enable the BMS pin while rebooting the device; the device will boot SAM-BA
program that is from ROM.

Now bring the BMS pin back to `0`. This is important! I took me 2 days to
discover that the BMS pin must set back to `0` in order to execute most of the
applets.

Validate that SAM-BA detects the serial port by running:

``` bash
$ ./sam-ba --port help
Known ports: serial, j-link
```

Before you can write anything to the NAND flash the RAM must be initialized.
This can be done by using the `extram` applet:

``` sam-ba
$ ./sam-ba -p serial -b sam9xx5-ek -a extram
Opening serial port 'ttyUSB0'
Connection opened.
Compatible device detected: SAM9G25.
Connection closed.
```

## Debrick NAND
Now it's time to to use the `nandflash` applet. Before writing a segment of the
NAND you should erase the existing content. This can be done using the `erase`
command. Then you can write a file to the NAND. You can use the either the
`writeboot` or the `write` command. The first command is for writing
a bootloader to the first segment of the NAND. The latter is for writing other
files.

The command in the next snippet loads the `nandflash` applet, erases the first
0x40000 bytes from the NAND and then uses the `writeboot` command to write the
file `at91sam9x5ek-nandflashboot-uboot-3.5.3.bin` to the NAND. No offset is
given, so AT91Bootstrap is written to address `0x0`.

The next snippet shows to write AT91Bootstrap to the NAND using the
`writeboot` command.

```
$ ./sam-ba -p serial -b sam9xx5-ek -a nandflash -c erase:0x0:0x40000 -c writeboot:at91sam9x5ek-nandflashboot-uboot-3.5.3.bin
Opening serial port 'ttyUSB0'
Connection opened.
Compatible device detected: SAM9X25.
Detected memory size is 268435456 bytes.
Page size is 2048 bytes.
Buffer is 131072 bytes (64 pages) at address 0x2000a000.
NAND header value is 0xc0c00405.
Supported erase block sizes: 128KB
Executing command 'erase:0x0:0x40000'
Erased 131072 bytes at address 0x00000000 (50.00%)
Erased 131072 bytes at address 0x00020000 (100.00%)
Executing command 'writeboot:at91sam9x5ek-nandflashboot-uboot-3.5.3.bin'
Prepended NAND header prefix (0xc0c00405)
Appending 1776 bytes of padding to fill the last written page
Wrote 16384 bytes at address 0x00000000 (100.00%)
Connection closed.
```

This next snippet shows how U-Boot is written to the NAND. Again, the `nandflash`
applet is loaded and a segment of the NAND is cleared. This time the segment
has an offset of `0x40000` bytes and it's size is `0x80000` bytes.  The file
`u-boot.bin` is written to the NAND with using that same offset of `0x40000`
bytes. The offset and size of the segment corresponds with the numbers found in
the output of the update that is part of the previous post.

``` bash
$ ./sam-ba -p serial -b sam9xx5-ek -a nandflash -c erase:0x40000:0x80000 -c write:u-boot.bin:0x40000
Opening serial port 'ttyUSB0'
Connection opened.
Compatible device detected: SAM9G25.
Detected memory size is 268435456 bytes.
Page size is 2048 bytes.
Buffer is 131072 bytes (64 pages) at address 0x2000a000.
NAND header value is 0xc0c00405.
Supported erase block sizes: 128KB
Executing command 'erase:0x40000:0x80000'
Erased 131072 bytes at address 0x00040000 (25.00%)
Erased 131072 bytes at address 0x00060000 (50.00%)
Erased 131072 bytes at address 0x00080000 (75.00%)
Erased 131072 bytes at address 0x000a0000 (100.00%)
Executing command 'write:u-boot.bin:0x40000'
Appending 940 bytes of padding to fill the last written page
Wrote 131072 bytes at address 0x00040000 (29.63%)
Wrote 131072 bytes at address 0x00060000 (59.26%)
Wrote 131072 bytes at address 0x00080000 (88.89%)
Wrote 49152 bytes at address 0x000a0000 (100.00%)
Connection closed.
```

Now you only have to write the device tree, Linux kernel and root filesystem
and your CORE 9G25 is working again.

## Conclusion
It took me days to figure it all out, but in the end the debrick procedure of
the CORE 9G25 turned out to be quite simple. I learned a lot while figuring it
out and writing these posts. I hope you learned something too. And I hopefully
saved you a little bit of time.

[sam-ba]: https://www.microchip.com/DevelopmentTools/ProductDetails.aspx?PartNO=Atmel%20SAM-BA%20In-system%20Programmer
[usb-to-serial]: https://www.conrad.nl/p/olimex-usb-serial-cable-f-developmentboard-1195115
