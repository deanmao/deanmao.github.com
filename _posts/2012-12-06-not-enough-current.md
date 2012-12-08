---
layout: post
title: "Not enough current"
description: ""
category: 
tags: []
---
{% include JB/setup %}

This is a follow up article to [Making a USB Device](/2012/12/02/making-a-usb-device/)

### Not enough current

Last week when I built my circuit, I falsely assumed that using watch batteries would be enough.  However
when I hooked it up, I saw 8V drop down to 4V using 3 CR2032 batteries in series.  I asked around and I found 
out that those watch batteries only have 200uA.  I looked at the [datasheet](http://www.sony.net/Products/MicroBattery/cr/pdf/cr2032.pdf)
for my batteries and saw that I would probably need to replace it with something better.

<img src="/images/rfid_wired.jpg">

Because of my diodes, I would actually need around 6V, I couldn't just use a 7805, so I wired up the handy MC34063
and made a mini switched regulator that accepted 9V and output 6V.

<img src="/images/rfid_wired2.jpg">
