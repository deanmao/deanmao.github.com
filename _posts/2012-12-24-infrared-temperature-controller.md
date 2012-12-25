---
layout: post
title: "infrared temperature controller"
description: ""
category: 
tags: []
---
{% include JB/setup %}

### Infrared Temperature Controller

[This is a link to my project repository referenced in this blog post](https://github.com/deanmao/infrared_temp_controller)

For the times that it does get cold in California, it does help to have a trusty space heater that can be used on occasion.  The one I have has a temperature control on it, but I didn't find it to be very useful or accurate.  This post details on how I went about creating my own external temperature controller.

<img src="/images/ir_heater.jpg">

#### The Remote

<img src="/images/ir_remote.jpg">

My heater has a little remote that transmits IR signals to control functions of the heater.  I planned on reverse engineering
the signals so that I could replay them using my microcontroller.  I found that I didn't even have to unscrew the device
to find the IR leads exposed and just hooked up my trusty logic analyzer to the LED pins.

<img src="/images/ir_hookup.jpg">

I fired up Saleae Logic and pressed the power button to record the signal.  It generates a bunch of pulses of a carrier
frequency around 38khz, but I would have to analyze the data to figure out the exact timings needed to send to the heater.

<img src="/images/ir_logic.png">

Here's the frequency info panel:

<img src="/images/ir_freq.jpg">

First, I had to export this data in a more useful format so that I could analyze it.  Saleae has the ability to export the
signal data into csv.  I was only interested in the signals & timings.  

<img src="/images/ir_export.jpg">

It creates an output that looks approximately like this:

    1.6386505, 1
    1.6386638125, 0
    1.6386769375, 1
    1.6386901875, 0
    1.6387033125, 1
    1.638716625, 0
    1.63872975, 1
    1.638743, 0

The next step was to write a small ruby script to parse the csv and convert it into frequency pulse timings.  You can find 
all the sample data and code in my [github repository](https://github.com/deanmao/infrared_temp_controller).  I would output
the analysis in a sequence of tuples where the first index indicated the number of carrier cycles and the second index 
indicated the time in microseconds that the IR led should be turned on.  Here's the example output:

47, 443, 47, 443, 15, 1292, 47, 446, 47, 454, 15, 1301, 15, 1301, 15, 1301, 15, 1301, 15, 1301, 15, 1299, 47, 7154, 47, 443, 47, 443, 15, 1292, 47, 446, 47, 454, 15, 1301, 15, 1301, 15, 1301, 15, 1301, 15, 1301, 15, 1299, 47, 7186, 47, 443, 47, 443, 15, 1292, 47, 446, 47, 454, 15, 1301, 15, 1301, 15, 1301, 15, 1301, 15, 1301, 15, 1299, 47, 7169, 47, 443, 47, 443, 15, 1292, 47, 446, 47, 454, 15, 1301, 15, 1301, 15, 1301, 15, 1301, 15, 1301, 15, 1299, 47, 7155, 47, 443, 47, 443, 15, 1292, 47, 446, 47, 454, 15, 1301, 15, 1301, 15, 1301, 15, 1301, 15, 1301, 15, 1299, 47, 7154, 47, 443, 47, 443, 15, 1292, 47, 446, 47, 454, 15, 1301, 15, 1301, 15, 1301, 15, 1301, 15, 1301, 15, 1299, 47, 7185, 47, 443, 47, 443, 15, 1292, 47, 446, 47, 454, 15, 1301, 15, 1301, 15, 1301, 15, 1301, 15, 1301, 15, 1299, 47, 7170, 47, 443, 47, 443, 15, 1292, 47, 446, 47, 454, 15, 1301, 15, 1301, 15, 1301, 15, 1301, 15, 1301, 15, 1299, 47, 7155, 47, 443, 47, 443, 15, 1292, 47, 446, 47, 454, 15, 1301, 15, 1301, 15, 1301, 15, 1301, 15, 1301, 15, 1299, 47, 7152, 47, 443, 47, 443, 15, 1292, 47, 446, 47, 454, 15, 1301, 15, 1301, 15, 1301, 15, 1301, 15, 1301, 15, 1299, 47, 7182, 47, 443, 47, 443, 15, 1292, 47, 446, 47, 454, 15, 1301, 15, 1301, 15, 1301, 15, 1301, 15, 1301, 15, 1299

You can get this output if you run my example ruby script against the csv file like this:  `ruby lasko_ir.rb power_button.csv`.
The output contains many repetitions of the signal, so it's wise to glance through it to see where the pattern starts repeating.
I found a sequence of 24 that seemed to repeat, so I used this as my IR code sequence which can be found at the top of the
main.cpp file in my [github repository](https://github.com/deanmao/infrared_temp_controller).  

    uint16_t power_button[] = {47, 443, 47, 443, 15, 1292, 47, 446, 47, 454, 15, 1301, 15, 1301, 15, 1301, 15, 1301, 15, 1301, 15, 1299, 47, 7154};

I also recalled that the [TV-B-Gone](http://www.tvbgone.com/) also happened to use the ATtiny85 microcontroller so I figured I'd save myself some time to see what they used to output their carrier frequency.  I found that they essentially modified the pwm frequency to fit the carrier frequency, then used a series of delays to structure the pulses.  The code is quite short and if you're interested, you can see it all in the git repo.  

I hacked in a simple watchdog timer to wake up every 2 seconds to check the temperature of the room, and if it was found to be
too cold, I'd crank on the heater.  Here's the final product all wired up on the breadboard:

<img src="/images/ir_breadboard.jpg">

If you also have a Lasko remote controlled heater, you might be able to make use of this project!

