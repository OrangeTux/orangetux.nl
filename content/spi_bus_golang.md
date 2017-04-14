Using SPI in Go is easy using the experimenal SPI package. In this post I'll
show how to read analog values using the MCP3002 over SPI. The MCP3002 is an
Analog Digital Converter (ADC) with 8 analog input channels. It has a
resolution of 10 bits which means it can read values up to 2^10 = 1024 values.

# SPI

Serial Periphiral Interface (SPI) is a master-slave protocol and it is being
used a lot for communications between chips in embedded systems. One master can
communicate with one or more slaves.

SPI is a so-called four-wire procotol because it requires (at least) 4 wires:

* Serial Clock (SLCK)
* Master Input Slave Output (MISO)
* Master Output Slave Input (MOSI)
* Slave Select (SS)

The master controls the SLCK line. One bit of data is shifted on every pulse
on the SLCK line.  The MISO line is used to transfer data from the client to
the master and the MOSI line is used to transfer data the other way around.

The SS line is used to select a slave. This is useful if a master controls
multiple slaves, but only want to write to a few of them.

For more information about SPI read the [Linux kernel documentation][kernel]
about this topic. I found another great article on [Sparkfun][sparkfun].

To interact with SPI devices from userspace the [spidev][spidev] needs to be
loaded. Normally SPI devices appear at /dev/spi<bus>.<chip> on the filesystem.

# Go

Go has an the expiremental package [x/exp/io/spi][go] which allows one to read
and write to an SPI device.

The folowing code creates a driver and uses that to open a SPI device.

``` go
        dev, err := spi.Open(&spi.Devfs{
		Dev:      "/dev/spidev32766.0",
		Mode:     spi.Mode0,
		MaxSpeed: 5000000,
	})

        if err != nil {
            log.Fatalf("failed to open SPI device: %s", err)
        }

        defer dev.Close()
```

The device is located at "/dev/spidev32766.0" and it's configured with a
maximum clock speed of 5000000 Hz. This exact number depends on the on the
slave and you should look it up in the datasheet of the slave.

The mode of the device is set to spi.Mode0. Other options are
spi.Mode1, spi.Mode2 and spi.Mode3. I won't go in to detail what these modes
are. Mode 1 and mode 3 are most commonly used. But if you are in doubt, check
the datasheet.

The clock speed of the MCP3008 depends on the power supply: it is 3.2 MHz when 5V is
applied and 1.2 MHz when 2.7V is applied.

SPI is a synchronous protocol, so for every byte you write on the
MOSI line, you read 1 byte from the MISO line. The following snippet shows how
to write 1 byte with value 0 to the slave and read 1 byte.

        out := []byte{1, byte(cmd), 0}
	in := make([]byte, len(out))

	if err := dev.Tx(out, in); err != nil {
		log.Fatalf("failed to read channel %s", err)
	}

# MCP3002



[kernel]: https://www.kernel.org/doc/Documentation/spi/spi-summary
[sparkfun]: https://learn.sparkfun.com/tutorials/serial-peripheral-interface-spi
