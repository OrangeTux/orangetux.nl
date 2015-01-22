Title: Automate Android app development with Docker
Date: 2015-01-22
Category: docker
Tags: docker, android
Slug: automate-android-app-development-with-docker
Authors: Auke Willem Oosterhoff
Summary: Automate build and installation of Android apps, speeding up your develepment.

For a small project for school me and my team mate [Maurits van
Mastrigt][mauvm] had to create an Android app. We decided to use
[Cordova][cordova]. This post demonstrates how to build and install an Android
app created with Cordova, but you can use any other tool you like. We als
wanted a Docker container which would take care of the building and
installation process. It turned out to be quite simple to do (on a Linux
machine). 

## Start container
We used [this image][base_image] for our container. The image comes with
Cordova and some Android build tools.

Create a folder where you want your code to be and start the container like
this:

    $ docker run -it --priviliged -v $(pwd):/app -v /dev/bus/usb:/dev/bus/usb \
        ugnb/ubuntu-cordova-android-build /bin/bash

This command will start a container and mounts your (empty) code directory in
it. It also mounts the USB device nodes from your host machine into the
container. The `--priviliged` flag gives the container access to all devices.
This way the container can access your Android phone via the USB debugger. 

## Create app
First we need some code to build our app from. 

    $ cordova create /app com.github.orangetux SampleApp

This command will genere the code for your app in `/app` directory of your
container. Because this directory has been mounted from your host machine this
code should also be visible on your host. Now add support for Android:

    $ cordova platform add android

## Debug USB
Before you can install the app on your phone the USB ports of your host should
be available in the conainter. Install `usbutils` in your container, connect
your Android phone with your host and check if you you can 'see' your phone by
running `lsusb`. Our phone is connected on bus 003 device 003.

    root@0520d37b082e:/# lsusb
    Bus 004 Device 003: ID 2232:1054  
    Bus 004 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
    Bus 006 Device 002: ID 0cf3:3004 Atheros Communications, Inc. 
    Bus 006 Device 001: ID 1d6b:0001 Linux Foundation 1.1 root hub
    Bus 003 Device 002: ID 0bda:0129 Realtek Semiconductor Corp. 
        RTS5129 Card Reader Controller
    Bus 003 Device 003: ID 04e8:6860 Samsung Electronics Co., 
        Ltd GT-I9100 Phone [Galaxy S II], GT-I9300 Phone [Galaxy S III], 
        GT-P7500 [Galaxy Tab 10.1]
    Bus 003 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
    Bus 005 Device 001: ID 1d6b:0001 Linux Foundation 1.1 root hub
    Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
    Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub

When youre phone is connected start the [Android Debugger Bridge][adb] and
check if `adb` is able to to detect your phone as well:

    root@c547f9b34347:/# adb devices
    * daemon not running. starting it now on port 5037 *
    * daemon started successfully *
    List of devices attached 
    4df792ad34de1187       unauthorized

This command will force a pop up on your phone asking for permission to use the
USB debugger. Accept it. When you run `adb devices` again you'll see that your
device has been authorized.

    root@c547f9b34347:/# adb devices   
    List of devices attached 
    4df792ad34de1187    device

## Install app
Now you are ready to install the app on your phone. Assuming that you are in
the root directory of your app, build and install your application:

    root@c547f9b34347:/app# cordova run --device
    Running command: /app/platforms/android/cordova/run --device
    Buildfile: /app/platforms/android/build.xml
    ...
    BUILD SUCCESSFUL
    Total time: 17 seconds
    Built the following apk(s):
        /app/platforms/android/ant-build/CordovaApp-debug.apk
    Using apk: /app/platforms/android/ant-build/CordovaApp-debug.apk
    Installing app on device...
    12Launching application...
    LAUNCH SUCCESS

Great, you build and installed your app from a Docker conainer!

## Automate
Now let's automate this stuff. We created a small `Dockerfile` and put the
commands used above in a script, both on your host system, of course.

    #!
    FROM ugnb/ubuntu-cordova-android-build
                                                                                         
    # Volume where code of app will be mounted on.
    VOLUME ["/app"]
    WORKDIR /app
                                                                                         
    CMD ["/var/tools/build.sh"]

Create `build.sh` and put it somewhere in your project, we put in in a `tools/`
directory. Note line 13, this command removes the previously installed
application. Replace the package name with the name of your package.

    #!bash
    #!/usr/bin/env bash
    set -e
    adb start-server
    
    # This will force a pop up on your phone asking for permission 
    # to USB debugging.
    adb devices                                                                        
                                                                                         
    # Let's wait a few seconds so user has a chance to allow the 
    # USB debugging.
    sleep 5                                                                              
                                                                                       
    # Delete current installation of app if present.
    echo ">> Deleting app..."                                                            
    adb shell pm uninstall com.github.orangetux
                                                                                       
    cordova run --device                 

Build an image from `Dockerfile` once:

    $ docker build -t="android"

And start the container:

    $ docker run -it --priviliged -v $(pwd):/app -v /dev/bus/usb:/dev/bus/usb \
        -v $(pwd)/tools:/var/tools android

## Troubleshooting
The process as described above works fine on a Linux machine. We had problems
on OSX. Docker runs on OSX inside VirtualBox which could cause problems with
mounting the USB device nodes through the VirtualBox layer in your docker
container.

Every time the container starts the "Allow USB debugging" pop up on your
Android phone pops op. Even if you checked "Always allow from this computer".
We don't exactly know how to fix this.

Sometimes the Android phone isn't visible inside the container or is
`unauthorized`. Unplug and plug your phone and restarting the container might 
help.

[cordova]:http://cordova.apache.org
[base_image]:https://registry.hub.docker.com/u/ugnb/ubuntu-cordova-android-build/
[adb]:http://developer.android.com/tools/help/adb.html
[mauvm]:http://mauvm.nl
