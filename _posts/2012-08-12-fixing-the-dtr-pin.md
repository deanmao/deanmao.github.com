---
layout: post
title: "Fixing the DTR pin"
description: ""
category: 
tags: []
---
{% include JB/setup %}

This is a follow up article to [Uploading Sketches to the Arduino on the Pi](/2012/08/10/uploading-sketches-to-the-arduino-on-the-pi/)

### What actually happens when you upload code to the arduino

If you were ever curious, you might look into avrdude's source code to see how it sends the data to the arduino.  There's really
nothing too fancy -- it just sends the data from the hex file in a serial stream of data.  However, there it has code to fire
the DTR pin briefly before sending the data.  This pin typically connects to a capacitor which connects to the reset pin
on your arduino.  Bouncing the pin would usually reboot the arduino, but it also announces to the optiboot system to use
the D0 and D1 pins as serial RX/TX briefly to receive commands from somewhere else.  During this short time, you can emit
the hex code that will run on the optiboot system.

If you peek into the source, you'll see something that looks like this:

    /* Clear DTR and RTS to unload the RESET capacitor 
     * (for example in Arduino) */
    serial_set_dtr_rts(&pgm->fd, 0);
    usleep(250*1000);
    /* Set DTR and RTS back to high */
    serial_set_dtr_rts(&pgm->fd, 1);
    usleep(50*1000);

It flags the pin for about 250ms.  The Raspberry Pi does not have the RTS/DTR pins that typically 9 pin serial ports have.  We need
to emulate this pin using one of the other available GPIO pins, and we need to flag the pin exactly when avrdude requests of it or
else we will be unable to upload the code.

### How the DTR pin is set

The underlying implementation of serial_set_dtr_rts in avrdude actually makes the ioctl call in a normal posix system like this:

    if (is_on) {
      /* Set DTR and RTS */
      ctl |= (TIOCM_DTR | TIOCM_RTS);
    }
    else {
      /* Clear DTR and RTS */
      ctl &= ~(TIOCM_DTR | TIOCM_RTS);
    }

    r = ioctl(fdp->ifd, TIOCMSET, &ctl);

We'd like to fire one of Raspberry Pi's GPIO pins exactly when they call ioctl.

### The hack

There's a little debugging tool called "strace" that lets you listen in on system calls for any binary.  For example, if we were to
run this command on avrdude like this:

    strace -eioctl avrdude -q -V -p atmega328p -C /etc/avrdude.conf \
           -c arduino -b 115200 -P /dev/ttyAMA0 -U flash:w:build-cli/arduino.hex:i

We'd get an output that looks like this:

    ioctl(0, SNDCTL_TMR_TIMEBASE or TCGETS, {B38400 opost isig icanon echo ...}) = 0
    ioctl(3, SNDCTL_TMR_TIMEBASE or TCGETS, 0xbe8962d4) = -1 ENOTTY (Inappropriate ioctl for device)
    ioctl(3, SNDCTL_TMR_TIMEBASE or TCGETS, 0xbe8962a4) = -1 ENOTTY (Inappropriate ioctl for device)
    ioctl(3, SNDCTL_TMR_TIMEBASE or TCGETS, {B9600 opost isig icanon echo ...}) = 0
    ioctl(3, SNDCTL_TMR_TIMEBASE or TCGETS, {B9600 opost isig icanon echo ...}) = 0
    ioctl(3, SNDCTL_TMR_TIMEBASE or TCGETS, {B9600 opost isig icanon echo ...}) = 0
    ioctl(3, SNDCTL_TMR_START or TCSETS, {B115200 -opost -isig -icanon -echo ...}) = 0
    ioctl(3, SNDCTL_TMR_TIMEBASE or TCGETS, {B115200 -opost -isig -icanon -echo ...}) = 0
    ioctl(3, TIOCMGET, [TIOCM_DTR|TIOCM_RTS|TIOCM_CTS]) = 0
    ioctl(3, TIOCMSET, [TIOCM_CTS])         = 0
    ioctl(3, TIOCMGET, [TIOCM_CTS])         = 0
    ioctl(3, TIOCMSET, [TIOCM_DTR|TIOCM_RTS|TIOCM_CTS]) = 0

We can actually see when avrdude performs ioctl on the DTR pin and now we can act on it.  Using some simple hackery, we will rename avrdude to
avrdude-original, and run a python script to emulate this behavior as if it were the actual avrdude program.  To see full instructions
on how this is accomplished, visit my github repository for this example: [https://github.com/deanmao/avrdude-rpi](https://github.com/deanmao/avrdude-rpi).

When you flash your arduino now, the output should look like this:

    root@raspberrypi:~/arduino# make upload
    done with autoreset

    avrdude-original: AVR device initialized and ready to accept instructions
    avrdude-original: Device signature = 0x1e950f
    avrdude-original: NOTE: FLASH memory has been specified, an erase cycle will be performed
                      To disable this feature, specify the -D option.
    avrdude-original: erasing chip
    avrdude-original: reading input file "build-cli/arduino.hex"
    avrdude-original: writing flash (1078 bytes):
    avrdude-original: 1078 bytes of flash written

    avrdude-original: safemode: Fuses OK

Mission accomplished!
