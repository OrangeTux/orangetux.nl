+++
date = "2017-09-13T13:37:00+02:00"
draft = true
title = "A look inside the Kobo Aura eReader"
description = "Teardown of the eReader Kobo Aura"
tags = [
    "embedded",
    "kobo",
]
categories = [
    "embedded",
]
+++

# Intro

Recently I broke the E-ink screen of my [Kobo
Aura](https://us.kobobooks.com/products/kobo-aura) eReader. Without fear
of damaging it, because it was already broken, I descided to open the device.
As a embedded Linux engineer myself I was curious to see if I could learn
something learn from the way Kobo build the hardware and software of this eReader.
What hardware components did they use? How does the firmware upgrade mechanism
work? Is the device secure? This post is a teardown of the hardware of the
Kobo Aura.

# Components

On the picture below you the PCB of the eReader. The most interesting
components have been numbered.

{{<figure src="/img/kobo.png">}}

1. SoC, [Freescale i.MX6 Sololite](http://www.nxp.com/docs/en/data-sheet/IMX6SLCEC.pdf)
2. RAM, [Nanya NT6TL64M32BG-G1](http://www.nanya.com/en/Product/List/547/2252)
3. Data storage, Sandisk
4. Wifi module, [Realtek 8189 FTV](http://www.realtek.com/products/productsView.aspx?Langid=1&PNid=21&PFid=48&Level=5&Conn=4&ProdID=354)
5. Power management, [Ricoh RC5T619](http://www.e-devices.ricoh.co.jp/en/products/product_pmic/multiple-pmu/rc5t619/rc5t619.pdf)
6. Crystal osscilators, 24 Mhz and 32.768 kHz
7. Serial debug port

## SoC

The haert of the Kobo Aura is the [i.MX6
Sololite](http://www.nxp.com/docs/en/data-sheet/IMX6SLCEC.pdf), produced by
Freescale (Freescale has been acquired by NXP in 2015). This 32-bit SoC runs at
a speed of 1 Ghz and is member of the Cortex ARM 9 family. According to the
datasheet it was designed for devices like eReaders, barcode scanners and even
low end tablets. The SoC is power efficient. I could use my
ereader for weeks without recharging the battery. The voltage and clock speed
can be scaled dynamically to conserve power. The clock speed can be scaled down
to to a frequency of 24 MHz.

The SoC has a coprossesor, the [Cortex-A9 NEON Media Processor
Engine](http://infocenter.arm.com/help/topic/com.arm.doc.ddi0409g/DDI0409G_cortex_a9_neon_mpe_r3p0_trm.pdf).
The coprossesor is used to accelerate the rendering of the things you see on
the screen.

<!--- zie pagine 22 v datasheet-->
<!--kristal is 24 Mhz-->
<!--real time clock 32.768 khz-->

## Data storage

All of the device's data is stored on an SD-card. In my case it has a capacity
of 4 GB SD-card by Sandisk. The SD-card contains 3 partitions:

* rootfs, an 256 MB EXT4 partition with the operating system
* recoveryfs, an 256 MB EXT4 partition with a recovery operating system
* KOBOeReader, a 3.15 GB FAT32 parition to store your ebooks on

A teardown of the rootfs is featured in an upcoming post.

Note the copper dots around the SD-card slot. These are probably tests points.
The manufactorer can connect an SD-card in an automated process, probably for
testing purposes. It's a lot easier than to insert a SD-card manually and to
peel it out when testing is done.

<!--sd zit in SD mode, 4 datal lijngen gevonden. Deze tonen random lijnen-->
<!--1 clock lijn, deze geeft constante golf van 50 Hz-->

## Memory

The memory bank is a [Nanya NT6TL64M32BG-G1](http://www.nanya.com/en/Product/List/547/2252).
I couldn't find this product this exact product of the vendor. But according to
the [part number guide](http://www.nanya.com/en/Support/59/Mobile%20DRAM) it's
LPDDR2 with a capacity of 2 GB. I was suprised to found out it has a capacity
of 2 GB. `cat /proc/meminfo` tells me that Linux thinks it's has only 256 MB
of memory. But more on this in a future post.

## Connectivity

The [Realtek
RTL8189FTV](http://www.realtek.com/products/productsView.aspx?Langid=1&PNid=21&PFid=48&Level=5&Conn=4&ProdID=354)
is responseble for the Wifi connection. It supports the 2.4 GHz band.  Note the
small blue knob at the top of the board. That is the external Wifi antenna.

## Power management

The [Ricoh
RC5T619](http://www.e-devices.ricoh.co.jp/en/products/product_pmic/multiple-pmu/rc5t619/rc5t619.pdf)
is responsible for managing the power for the whole board. It is responsible
for both loading and unloading the battery, switching between between USB and
battery as power source and for distributing the power to all components.

<!--ricoh rc5t519-CSP-->
<!--http://www.e-devices.ricoh.co.jp/en/products/product_pmic/multiple-pmu/rc5t619/rc5t619.pdf-->
<!--power device-->
<!--heeft 5 power rails, oa 1 van 1.2V, 1.8V, 3.3V, 3.5V. Deze bregen spanning naar-->

## Serial debug port

The PCB exposes a serial debug port. Using a USB-to-serial cable you could get
root access to the system.

## Screen

Unfortunately I couldn't find the datasheets or product pages of the screen
components. The E-ink screen is a [Ichia 94V-0 1623](http://www.ichia.com). The
touch screen controller is an [ELAN EKTF2232ALW](http://www.emc.com.tw).

