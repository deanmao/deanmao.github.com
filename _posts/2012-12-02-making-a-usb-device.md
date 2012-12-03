---
layout: post
title: "Making a USB device"
description: ""
category: 
tags: []
---
{% include JB/setup %}

### Making a USB device

I know I haven't been making blog posts recently, but I figured I would make one on a hardware/software device since
it's the first one where I had to build everything from the desktop software to physical hardware.  It was also the
first time I had to make heavy use of my logic analyzer.

As some of you may be aware, I'm involved in a local hackerspace called the Hacker Dojo.  In an email thread, someone
suggested that we have a handheld rfid scanner that could be used by security personnel to determine if someone was
a member or not.  I mentioned in an email that it could probably be built cheaply and it initiated my side project. 

    > It could be about the size of an iphone and flash green/red if the person is a member or not, and have a usb jack for uploading the rfid numbers.  I'm guessing it would be around $30 in parts to make one.

    That is really cheap.  I'm interested. We should build more than one for redundancy.

Obviously the costs involved are actually the man-hours developing it, but I figured if I learned something, then it
was probably worth it.  I also like to self-impose constraints to the problem so that the solution doesn't become
something stupid-easy like slapping a rfid reader on top of a laptop and calling it a day.  I wanted the device to 
be relatively low power, use as few components as possible, cheap, easy to use, and self maintenance.

I decided upon using a $10 rfid scanner from seeedstudio, an ATtiny 85, and powered with some watch batteries.  I figured
the final cost could be around $15.  All the code for my project can be found on my github here:

The software talked about is here ---> [the code](https://github.com/deanmao/dojo_member_tool) <---

### Data storage, bloom filter?

Initially I thought I would use the tiny85's 512 byte eeprom as a bloom filterbit array, but upon calculating how many bits I would need, it turned out I would still have around 1% false positives.  In the following equation, n = number of 
items in the filter, p = probability of false positive, m = number of bits in the filter, k = number of hash functions.

    m = ceil((n * log(p)) / log(1.0 / (pow(2.0, log(2.0)))));
    k = round(log(2.0) * m / n);

If I were to store 350 objects and wanted at least a 0.01% chance of false positive, I would need 1k of storage.  I had
some spare 24LC256 chips lying around so I used one of those.  Since the chip had 32k on board, I figured I would just
load all the raw data instead of implementing a bloom filter.  I only need 6 bytes to store 1 rfid key.  The Hacker
Dojo only has around 350 members currently, which means I can store all the raw data in 2kb.  

### Choosing USB

I chose usb as the data transfer mechanism since everyone has a usb port.  There's a little known project called 
[V-USB](http://www.obdev.at/products/vusb/index.html) that is a software solution for making any AVR chip into something
that can talk usb 1.1.  I chose to base my project on top of another existing V-USB project called
[micronucleus](https://github.com/bluebie/micronucleus-t85) (Hi Jenna!).  It's a micro bootloader that can receive the
data via usb.  I modified the bootloader so that I could also use the V-USB parts to send/receive data from the computer
for my project's purposes.

<img src="/images/rfid_breadboard.jpg">

As you can see, it is a fairly simple design.  I only have 5 pins to work with on the tiny85, and I'm actually reusing 2
of them when the device is powered externally (not connected to usb).  Pins 2 & 3 are used for the usb D-/D+, pin 5 is 
for the rfid serial rx, and pins 6 & 7 are the I2C SDA/SCL.  When usb isn't connected, pins 2 & 3 are output pins for 
the green & red LED that you see in the picture.  I use a small npn transistor & diode for switching them.  

### Debugging

Debugging was actually fairly difficult since the tiny85 doesn't have many pins, so sometimes I would output serial on the 
LED pin just so that I could verify if my data was correct or not.  In the breadboard image, you can see that I have the
SDA/SCL & rfid rx line hooked up to my trusty logic analyzer.

<img src="/images/saleae_logic.png">
 
When connected to usb, we have to use a 16.5 Mhz clock because the internal oscillator is only precise up to 8 Mhz.  Anything
higher may require some trickery.  The 16.5 Mhz speed was built specifically to target the AVR chips with the 64 Mhz PLL. 
(the ATTinyX5 line of chips).  The software serial library was only configured to use 8/16/20 Mhz, not the 16.5 Mhz that I
had.  The frame errors were visible in the logic analyzer so I had to reduce the speed for serial to work properly.

    CLKPR = 1 << CLKPCE;
    CLKPR = _BV(CLKPS0);

The above will reduce speed to 8mhz, allowing your time sensitive libraries to operate correctly. 

### The Software

The bootloader was modified to add additional usb functions that allowed my desktop software to load in the member rfid
keys.  The loaded firmware would contain the code to check if a rfid key existed or not and flash the appropriate red/green
led.  My code uses libusb to communicate with the device, which is relatively cross platform.

<img src="/images/desktop_software.png">

The desktop software is relatively simple.  It allows me to update the firmware on the device remotely.  I would upload the
new firmware to github, then have the user push the "Update Firmware" button and plug in their device when prompted to.
They could also update the member keys stored on the device by pushing the "Upload Members" button.

### Concluding Remarks

Although this side project isn't yet complete, as I still have to create the physical form in CAD and have it made on our
MakerBot Replicator, it feels relatively polished since the end user won't have to muck with the internals to update
functionality.  The hardware used is pretty fairly cheap, around $15 for the final design.  If the device were to 
transform into something with more functionality, it might be worth getting the board etched properly in bulk.  
