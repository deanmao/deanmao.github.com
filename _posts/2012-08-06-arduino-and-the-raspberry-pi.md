---
layout: post
title: "Arduino and the Raspberry Pi"
description: ""
category: 
tags: []
---
{% include JB/setup %}

### I got a PI!

So I got a raspberry pi.  It took almost 10 weeks to arrive, but it was definitely worth the wait.

For those who don't know, the Raspberry Pi is a cheap $35 computer that includes processor, memory,
and a sd card slot for the hard drive.  It also has a few hardware pins that can be used to drive
external devices.  

### Now to interface with a micro-controller...

The most obvious external "thing" I wanted to hook up was the arduino.  The Arduino is kind of like
python in the embedded systems world -- it's not very hard to write something running at the
extremely low power of 20mA, and there's a huge wealth of library code one can pull in to power
motors, lcd panels, etc.  Most beginners of the Arduino would buy a starter kit that 
includes everything you need to get you need to get running assuming you have a computer and some
spare time.  

You plug the arduino into your usb port, upload some code, and suddenly you're 
controlling a motor.  Eventually one migrates off the standard arduino starter kit and will simply
include the atmega328p chip as part of their overall schematic since it isn't exactly cheap to
have an arduino shield for every little project -- it's a $30 module for a $3 chip.  One
can cheaply harness the power of the arduino with a atmega328p IC, a capacitor, resistor, and
ceramic resonator -- a combined cost of around $4.  

Finally, you'll want to interface the arduino with a computer, because it's no fun having a 
microcontroller that can't interface with the internet.  This is where the cheap $35 raspberry pi
comes in.  However, it's not extremely straightforward getting the rpi working with the arduino,
and it's certainly harder than the $30 module arduinos come in.

### Commmand-line arduino?

The raspberry pi only has 256mb of memory so you'll probably want to upload your sketches via
command-line instead of using their fancy UI.  Their UI does nothing for me anyway, as I prefer
editing the code in vim.  Here's what you'll have to do to get the Arduino talking to your 
Raspberry Pi:

We're going to use the serial header exposed by the GPIO pins to communicate with our arduino,
so we'll have to remove it's use on the raspberry pi.  Most distros come configured so that
it acts as another getty session.

Comment out the line in /etc/inittab that references ttyAMA0 (the serial port on the GPIO header):

    #Spawn a getty on Raspberry Pi serial line
    #T0:23:respawn:/sbin/getty -L ttyAMA0 115200 vt100

The entry should be at the bottom of the file.  The serial port is also used as a serial console
so we'll have to disable that as well.  Change the contents of /boot/cmdline.txt to the
following:

    dwc_otg.lpm_enable=0 console=tty1 root=/dev/mmcblk0p2 rootfstype=ext4 elevator=deadline rootwait

Essentially we're removing all references to ttyAMA0.  Depending on what distro you're using,
you may have more references to ttyAMA0.  It might be a good idea to grep all files to see where
else it might be used:

    fgrep -rn ttyAMA0 /etc /boot

Finally, we'll want to disable the magic sysrq key as spurious signals on the serial line can
sometimes trigger the kernel into thinking that the sysrq key was fired, and the kernel will "hang"
because it's in debugging mode.  Add this line to the bottom of your /etc/sysctl.conf:

    kernel.sysrq=0

Finally, we'll install the command-line tools needed to upload your arduino sketches:

    apt-get install arduino-mk

### Wiring up the arduino

We'll cover this in another post soon!