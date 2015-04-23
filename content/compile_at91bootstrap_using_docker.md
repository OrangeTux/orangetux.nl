Title: Compile AT91Bootstrap using Docker
Date: 2015-04-23
Category: docker
Tags: docker
Slug: compile-at91bootstrap-using-docker
Authors: Auke Willem Oosterhoff
Summary: Cross-compile AT91Bootstrap boot loader for Atmel's SMART microprocessor in a Docker container without bloating your system with a tool chain.
Status: draft

A few weeks ago [Tom de Boer][tom_de_boer] helped me to get Linux 4.0 working
on Atmel's [AT91SAM9G25][at91sam9g25]. Cross compiling a bootloader was part of
that process. An open source bootloader for this SoC is available:
[AT91Bootstrap][at91bootstrap]. From its [README][readme]:

> "AT91Bootstrap is the 2nd level bootloader for Atmel SMART microprocessors
> (aka AT91)."

Because I didn't want to bloat my system with a cross-compile toolchain I
decided to do the work inside a Docker container.

## Dockerfile
The Dockerfile for the Docker image is quit simple. Git is required to fetch
the latest version of the bootloader's source from [GitHub][at91bootstrap]. A
cross compiler is required to compile the bootloader and the development
headers of [ncurses][ncurses] are needed to build the configuration menu.

    #!
    RUN apt-get update && apt-get install -y
        binutils-arm-linux-gnueabi \                                                
        build-essential \                                                           
        gcc-arm-linux-gnueabi \                                                     
        git \                                                                       
        libc6-dev-armel-cross \                                                     
        libncurses5-dev    

Next the source of AT91Bootstrap must be fetched.:

    #!
    RUN git clone https://github.com/linux4sam/at91bootstrap.git \
        --depth=1 /root/at91bootstrap

My final Dockerfile looked like this:

    #!
    FROM ubuntu:14.10                                                                  

    RUN apt-get update && apt-get install -y \
        binutils-arm-linux-gnueabi \
        build-essential \
        gcc-arm-linux-gnueabi \
        git \
        libc6-dev-armel-cross \
        libncurses5-dev
                                                                                     
    RUN git clone https://github.com/linux4sam/at91bootstrap.git \
        --depth=1 /root/at91bootstrap

    WORKDIR /root/at91bootstrap

    CMD ["/bin/bash"]

Now a Docker image can be build from this Dockerfile:

    #!
    $ docker build -t "orangetux/at91bootstrap" .

## Compile bootloader
After building the Docker image you're ready to compile AT91Bootstrap. Start
and enter the container:

    #!
    $ docker run -ti orangetux/at91bootstrap 

Compile the configuration.:

    #!
    root@2ab4258fb321:~/at91bootstrap# make menuconfig
    *** End of at91bootstrap configuration.
    *** Execute 'make' to build at91bootstrap or try 'make help'.

    #
    # make dependencies written to .auto.deps
    # ATTENTION at91bootstrap devels!
    # See top of this file before playing with this auto-preprequisites!
    #

And build the bootloader.:

    #!
    root@2ab4258fb321:~/at91bootstrap# make CROSS_COMPILE=arm-linux-gnueabi-
    [...]
    [Succeeded] It's OK to fit into SRAM area

The build products can be found in `/root/at91bootstrap/binaries/`.

## Persist build products
Although above instructions work, the are not very useful on their own. 
The configuration created with `make menuconfig` is saved in the container, but
is not accessible from the host. Because the stateless nature of containers
you've to reconfigure AT91Bootstrap every time the container is started. In
order to save the configuration for future use the configuration must be saved
on the host. This can be achieved using Docker's volumes. 

    #!
    $ docker run -it \
        -v $(pwd)/.config:/root/at91bootstrap/.config \
        orangetux/at91bootstrap

The `-v` flag tells Docker to mount the file `.config` of the current directory
as a [volume][docker-volumes] in the container at `/root/at91bootstrap.config`.
The file is shared between host and container.

**Note**: For some reason `make` won't build the menu when an empty config file
is mounted. [Here][.config] is a 'real' config with all default settings.

The build products in `binaries/` are not saved on the host too, just like the
configuration. In order to save the om the host machine create a directory 
and mount it as a volume at `/root/at91bootstrap/binaries` in the container.

    #!
    $ docker run -it \
        -v $(pwd)/.config:/root/at91bootstrap/.config \
        -v $(pwd)/binaries:/root/at91bootstrap/binaries \
        orangetux/at91bootstrap

## Wrapping up
The Dockerfile and a small script to save are config and build products have
been published in [this][orangetux/docker-at91bootstrap] repository on GitHub. 
A Docker image is available at [Docker Hub][docker-hub].


[tom_de_boer]:http://tomdeboer.nl/
[at91sam9g25]:http://www.atmel.com/devices/SAM9G25.aspx
[readme]:https://github.com/linux4sam/at91bootstrap/blob/master/README.txt
[at91bootstrap]:https://github.com/linux4sam/at91bootstrap
[ncurses]:https://www.gnu.org/software/ncurses/
[docker-volumes]:http://docs.docker.com/userguide/dockervolumes/
[.config]:https://raw.githubusercontent.com/OrangeTux/docker-at91bootstrap/master/.config.default
[orangetux/docker-at91bootstrap]:https://github.com/orangetux/docker-at91bootstrap
[docker-hub]:https://registry.hub.docker.com/u/orangetux/at91bootstrap/
