+++
date = "2017-04-23T12:02:53+02:00"
draft = false
title = "Writing a SPI driver for the MCP3008 in Go"
description = "Use Go to create a simple driver to interact with the MCP3008 using SPI."
tags = [
    "go",
    "mcp3008",
    "spi",
]
catogories = [
    "spi",
    "go"
]
slug = "writing-spi-driver-for-mcp3008-in-go"
+++

Using SPI in Go is easy using the experimental
[golang.org/x/exp/io/spi][go_spi] package. In this post I'll show how to read
analog values using the [MCP3008][mcp3008] over SPI. The MCP3008 is an Analog
Digital Converter (ADC) with 8 analog input channels. It has a resolution of 10
bits which means it can read up to `2^10 = 1024` different values.

## SPI
Serial Peripheral Interface (SPI) is a master-slave protocol and it's used a
lot for communications between chips in embedded systems. One master can
communicate with one or more slaves.

SPI is a so-called four-wire protocol because it requires 4 wires:

* Serial Clock (SLCK or SCK)
* Master Input Slave Output (MISO)
* Master Output Slave Input (MOSI)
* Slave Select (SS)


The master controls the SLCK line. One bit of data is shifted on every pulse on
the SLCK line.  The MISO line is used to transfer data from the client to the
master and the MOSI line is used to transfer data the other way around. SPI is
a synchronous protocol, so for every byte you write on the MOSI line, you read
1 byte from the MISO line. The SS line is used to select a slave. A slave only
communicates if it's SS line is pulled low.

{{<figure src="/img/spi.png">}}

For more information about SPI read the [Linux kernel documentation][kernel]
about this topic. The image above comes from this great article about SPI from
[Sparkfun][sparkfun].

## Open device
Note that to interact with SPI devices from userspace the [spidev][spidev]
kernel driver needs to be loaded.

The following code creates a driver and uses that to open a SPI device.

``` go
import (
    "log"

    "golang.org/x/exp/io/spi"
)

conn, err := spi.Open(&spi.Devfs{
        Dev:      "/dev/spidev32766.0",
        Mode:     spi.Mode0,
        MaxSpeed: 3600000,
})

if err != nil {
    log.Fatalf("failed to open SPI device: %s", err)
}

defer conn.Close()
```

The device is located at `/dev/spidev32766.0` and it's configured with a
maximum clock speed of 3600000 Hz. This exact number depends on the on the
slave and its reference input voltage. You should look it up in the datasheet
of the slave. With a reference input of 5V the maximum frequency for the
MCP3008 is 3.6 Mhz.

{{<figure src="/img/mcp3008_frequency.png">}}

The mode of the device is set to `spi.Mode0`. Other options are `spi.Mode1`,
`spi.Mode2` and `spi.Mode3`. I won't go in to detail what these modes are. Mode
1 and mode 3 are most commonly used. But if you are in doubt, check the
datasheet.

## Read channel
The ADC measures the voltage difference between 2 pins. A single-ended input
samples its input in the range from the ground (0V) to Vref, that is the refence
input. A 10-bits ADC with a reference input of 5V has a precision of `(5 - 0) /
1024 = 0.0049V = 4.9mV` on single-ended inputs.

A (pseudo-)differential input samples its input between voltage of a
second pin and Vref, allowing measurements with higher precision. Assume a
voltage on that second pin is 3V and Vref is 5V. That gives a precision of
`(5 - 3) / 1024 = 0.0020V = 2.0mV` for measurements between 3V and 5V. Of
course, values between 0V and 3V cannot be measured in this case.

To read a digital output code of a channel the master starts with a start bit,
followed by a bit to indicating single-ended (1) or pseudo-differential mode
(0). The next 3 bytes are used to select 1 the 8 channels. The MCP3008 will
"answer" with a 10 digital output code. The following figure from
[MCP3008's datasheet][mcp3008_datasheet] visualize the process.
The datasheet names the lines differently, but

* CS is Chip Select
* CLK is Serial Clock
* Din is MOSI
* Dout is MISO

Note the null bit between the request on the Din line and the the response on
the Dout line.

{{<figure src="/img/mcp3008_communication.png">}}

The code below shows how to read digital output code of channel 5 in
single-ended mode:

``` go
//        |------------------- 1 start bit
//        | |----------------- 1 bit to select single-ended/pseudo-differential input
//        | ||||-------------- 3 bits to select channel
// 00000001 11010000 00000000
// Out is te data send over the MOSI line.
channel := 5
out := []byte{1, (8 + channel) << 4, 0}
// In is the data received over the MISO line.
in := make([]byte, 3)

if err := conn.Tx(out, in); err != nil {
        return 0, fmt.Errorf("failed to read channel %d: %v", channel, err)
}

// The 10-bits digital output code is at the end of the 3 byte response.
//
// 11111111 11111010 10110111
//                ^^ ^^^^^^^^
// To get the base10 value of the channel the second byte is masked
// with 3:
//
//	    11111010
//          00000011
//          -------- &
//          00000010
//
// The byte is shifted 8 bits and the last byte is added:
// 00000010 00000000
//          10110111
//          -------- +
// 00000010 10110111
//
// 00000010 10110111 is 696 in base10.
code := int(in[1]&3)<<8 + int(in[2])
```

The complete driver for the MCP3008 can be found in the package
[github.com/advancedclimate/io/spi/adc][github]. It also implements drivers
for the [MCP3004][mcp3004], [MCP3204][mcp3204] and [MCP3208][mcp3208].

[github]: https://github.com/AdvancedClimateSystems/io/tree/master/spi/microchip
[go_spi]: https://godoc.org/golang.org/x/exp/io/spi
[kernel]: https://www.kernel.org/doc/Documentation/spi/spi-summary
[mcp3004]: http://www.microchip.com/wwwproducts/en/MCP3003
[mcp3008]: http://www.microchip.com/wwwproducts/en/MCP3008
[mcp3204]: http://www.microchip.com/wwwproducts/en/MCP3204
[mcp3208]: http://www.microchip.com/wwwproducts/en/MCP3208
[mcp3008_datasheet]: http://ww1.microchip.com/downloads/en/DeviceDoc/21295d.pdf
[sparkfun]: https://learn.sparkfun.com/tutorials/serial-peripheral-interface-spi
[spidev]: https://www.kernel.org/doc/Documentation/spi/spidev
