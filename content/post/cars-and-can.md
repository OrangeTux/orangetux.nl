+++
date = "2017-10-21T13:37:00+01:00"
description = "Use a cars CAN bus to read live data."
title = "TED2: Cars and CAN"
series = ["Ted"]
slug = "cars-and-can"
author = "Auke Willem Oosterhoff"
+++

In [previous post]({{< relref "post/obd_ii_elm327_and_python.md">}}) of this
series I demonstrated how to obtain live data of a car by using an OBD-II
adapter. I was a bit disappointed about the fact that I could only read a
handful of parameters. Together with Wisse Hooghiem I went a step further by
hooking up in the CAN network of my car. In this post I'll demonstrate how to
interact with a car's CAN bus in Linux with help of a an Arduino Uno.

## CAN bus

First, what is CAN? Every car contains electronic chips to control parts of the
car. The are called Electronic Control Units (ECUs). I identified a few ECUs in
my car from 2006. Modern cars have a lot more chips on board.

To list a few of the ECUs:

* Motor Control Unit (MCU) - responsible for things like fuel injection and
  engine speed monitoring
* Door Control Unit (DCU) - handles the (un)locking of the doors and the
  windows
* Tranmission Control Unit (TCU) - controlls the operations related to the tranmsission

Most ECUs require input from other ECUs to operate. For example the Airbag
Control Unit requires information about speed, acceleration and occupant
status. Therefore, in most cars the ECUs are connected using a CAN bus.
CAN is an acronym for Controller Area Network and it has been developed by the
German company Bosch. CAN is a simple 2-wire bus. Its
first use in cars dates back to 1990.

## Data frames

CAN has 4 packet types. But in this post we're only interested in 1 type: the
data frame. A data frame is used to send data, for example the motor
temperature. These frames are broadcasted over the bus. The other packet types
are used for transmitting errors, to ask for data or to add a small delay
between other frames.

A data frame constists of 4 parts:

* Arbitration ID
* Identifier Exentions
* Data Length
* Data

All parts are marked in the image below.

{{<figure src="/img/can_data_frame_layout.png">}}

The arbitration id defines the meaning of the data. For example in my car the
data frames with id 0x2F1 contain the steering angle. Normally this id is 11
bits in length. This is called a base frame. In case of a base frame the
Identifier Extenstion bit is 0. When this bit is 1 it's called an extened
frame. With these type the id is 29 bits long.

The data length part holds amount of bytes of data to follow. The data can be 8
bytes long at max.

This intro into CAN is fairly brief. If you want to learn more about this topic
I recommend reading [Wikipedia page about CAN][wikipedia] or [chapter
2][bus-protocols] of the [Car Hacker's Handbook][car-hackers-handbook] by Craig
Smith.

## MCP2515

Your Linux machine probably doesn't have a CAN bus. In order to receive the
data frames on your machine additional hardware is required. I used a [CAN bus
shield][can-bus-shield] mounted on top of an Arduino Uno.

This shield will act as a CAN proxy between a Linux device and your car. [This
sketch][sketch] must be loaded on the Arduino.

The shield must be connected to the CAN bus. If your car has a CAN bus it's
very likely that the bus is wired to the OBD-II connector. Pins 6 and 14 are
reserved for the CAN bus as shown in the image below.

{{<figure src="/img/obd-ii-pin-out.jpg">}}

The main component of the shield is the [MCP2515][mcp2515]. This is a
CAN-controller with a SPI interface. I haven't tried, but I guess any other
shield with the MCP2515 that fits on a Arduino Uno will work as well.

## SocketCAN

Linux has native support for CAN starting from Linux 2.6.25. The patches where
provided by car maker VolksWagen and are called SocketCAN. The VW engineers did
a clever job and extended the Berkley Socket API. This allowed one to use a CAN
socket in the same way one works with a TCP/IP socket. For example you can use
the `ip` command to create a CAN socket.

As mentioned before, your laptop probably doesn't have a CAN port. Often
external CAN devices, like our Arduino, are connected via serial and use their
own protocol to communictie: slcan, which stands for serial can. slcan is an
ASCII representation of CAN. The code on the Arduino does nothing more than
translating CAN to slcan and vice verca.

## can-utils

`can-utils` is a set of utilities and tools to configure and use SocketCAN.
Install the `can-utils` from your favorite package manager or compile them
yourself:

``` bash
$ git clone https://github.com/linux-can/can-utils
$ cd can-utils
$ make
$ sudo make install
```
Connect your Linux machine with the Arduino using a USB A cable. The device
should show up as `/dev/ttyACMx` or `/dev/ttyUSBx`. When in doubt run `dmesg`
and look for lines like this:

``` bash
[32573.775684] cdc_acm 1-1:1.0: ttyACM0: USB ACM device
```

Initialize `slcan0` using these commands:

``` bash
$ sudo slcand -o -t hw -S 1000000 /dev/ttyACM0 slcan0
$ sudo ip link set up slcan0
```

The first command configures the `slcan0` interface with a baudrate of
1000000. The latter comand brings the `slcan0` interface up. Just as you would
do with a 'normal' network interface.

Assuming that the shield is connected to the your car's CAN bus you now should
be able to read and send data!

``` bash
$ candump slcan0
```

## Summary

In this post, you learned about the CAN protocol and how to connect to the CAN
bus in your car. You also learned how to read data from the CAN bus using the
Linux CAN subsystem, also known as SocketCAN. In the next post I'll show how
to find meaning in the flood of messages that is send over the CAN bus.

## Further reading

* [Hacking Cars wit Python - Eric Evenchick](https://www.youtube.com/watch?v=3bZNhMcv4Y8)
* [Vehicle Hacking Village - Eric Evenchick (video)](https://www.youtube.com/watch?v=Ym8xFGO0llY)
* [Hopping on the CAN bus - Eric Evenchick (video)](https://www.youtube.com/watch?v=U1yecKUmnFo)
* [Hacking into a Vehicle CAN bus - Fabio Baltieri](https://fabiobaltieri.com/2013/07/23/hacking-into-a-vehicle-can-bus-toyothack-and-socketcan/)
* [CAN bus - Wikipedia][wikipedia]
* [Car Hacker's Handbook - Craig Smith][car-hackers-handbook]

[bus-protocols]: http://opengarages.org/handbook/ebook/#calibre_link-261
[can-bus-shield]: https://www.tinytronics.nl/shop/nl/arduino/shields/can-bus-shield-mcp2515?search=can
[car-hackers-handbook]: http://opengarages.org/handbook/
[mcp2515]: http://www.microchip.com/wwwproducts/en/en010406
[wikipedia]: https://en.wikipedia.org/wiki/CAN_bus
