---
layout: post
title: "RFID cloner"
description: ""
category: 
tags: []
---
{% include JB/setup %}

### RFID cloner

After making the [usb rfid device](/2012/12/02/making-a-usb-device/), I thought it would be pretty cool to make something
to copy & write a rfid key.  I googled around and saw a another project that made an [rfid transponder](http://wiki.smallroom.net/doku.php?id=terd:projects:rfidspoofer) and decided it wouldn't
be too hard to make something similar, but add an rfid reader on it as well.

<img src="/images/rfid_spoofer.jpg">

I'm using Seeedstudio's rfid reader here because it has an external antenna.  I'm reusing the external antenna as the
transponder coil.  When you power up the device, the first 3 seconds are used to read rfid, and we'll transmit that
rfid number after 3 seconds.  You can see the simple code on how to accomplish that here: [my arduino rfid spoofer code](https://github.com/deanmao/rfid_spoofer).

I was surprised at how strong the signal was -- I could transmit my rfid code almost a foot away from one of my smaller
rfid readers.  In my design, one transistor is used to turn the external rfid reader on/off, another transistor is used
in the transponder antenna oscillator, and the third transistor is used to control who can make use of the antenna, since
the antenna is shared by the reader and transponder.  
