---
layout: post
title: "Uploading Sketches to the Arduino on the Pi"
description: ""
category: 
tags: []
---
{% include JB/setup %}

This is a follow up article to [Arduino and the Raspberry Pi](/2012/08/06/arduino-and-the-raspberry-pi/)

### Now that your RPi has all the needed software

You should now wire up your arduino [similar to how it's done here](http://log.liminastudio.com/itp/physical-computing/breadboard-arduino-fast-cheap-and-fun).
You can skip the part about using the 5V linear regulator because we're going to use the ground and the 5V line directly from the 
GPIO pins available on the raspberry pi.  Use the [GPIO pin out documentation](http://elinux.org/RPi_Low-level_peripherals) here for guiding
yourself where the pins are.  It helps to use a multimeter to ensure you have the proper 5V line hooked up.

Alternatively, you can buy a [prototyping shield from adafruit](http://adafruit.com/products/801) where the pins are already broken
out for you.  All you need to do is locate the tx, rx, 5v, and gnd lines.  It also helps to map down at least one digital io pin that
we'll be using as a fake DTR pin.

Finally, you'll need something like a [logic level converter](http://adafruit.com/products/757) before you wire up any pins to the
arduino because most of the pins on the raspberry pi operate at a 3.3v level and your arduino uses 5v.  The logic level converter
makes it really easy to hook up 3.3v lines that will directly convert to 5v lines, and will support the signal in both directions (so 
if your signal goes low on the 5v side, it will also go low on the 3.3v side, and vice versa)

Here's what my board looks like wired up.  The power rails on the left side of the breadboard are 3.3v, and the rails on the right
side are 5v.  B1 on the logic level converter is hooked up to D0 on the arduino, B2 is hooked up to D1, and B3 is hooked up to RST.
A1 on the logic level converter is the 3.3 equivalent of B1.  A1 hooks up to TX on the raspberry pi gpio pins, and in this case it is 
pin 8 on the header.  A2 hooks up to RX which is pin 10 on the header. 

<img src="/images/arduino.png">

### Preparing your environment

Now when you upload code to the arduino on the command line, you should create an empty directory where you'll store your code.  For
our examples, lets call this directory "mysketch".  You'll also need to copy a sample Makefile that contains all the necessary logic
for uploading sketches.  You can find this "Makefile" in the base arduino package that you installed.  We'll run "dpkg -L arduino-mk"
to find out where this file is located:

    root@raspberrypi:~# dpkg -L arduino-mk
    /.
    /usr
    /usr/bin
    /usr/bin/ard-parse-boards
    /usr/share
    /usr/share/arduino
    /usr/share/arduino/Arduino.mk
    /usr/share/man
    /usr/share/man/man1
    /usr/share/man/man1/ard-parse-boards.1.gz
    /usr/share/doc
    /usr/share/doc/arduino-mk
    /usr/share/doc/arduino-mk/copyright
    /usr/share/doc/arduino-mk/changelog.Debian.gz
    /usr/share/doc/arduino-mk/arduino-cli.html
    /usr/share/doc-base
    /usr/share/doc-base/arduino-cli

The file we're looking for is called Arduino.mk.  Copy this file to your mysketch directory and rename it "Makefile"

    cp /usr/share/arduino/Arduino.mk ~/mysketch
    cd ~/mysketch
    mv Arduino.mk Makefile

You'll need a few environment variables set properly so that the tools know where your arduino is hooked up.  You should probably
add these to your .bashrc or .profile file so that they will be there the next time you login:

    export BOARD=uno
    export SERIALDEV=/dev/ttyAMA0
    export ARDUINO_PORT=/dev/ttyAMA0
    export ARDUINO_DIR=/usr/share/arduino

Unless you're using a different distro or arduino bootloader version, these parameters will probably be identical for you.

Next you'll create your arduino code which should have a file that end in ".ino".  When you list the contents of the directory,
you'll have 2 files in there, the Makefile we copied over, and your arduino sketch:

    root@raspberrypi:~/# cd mysketch
    root@raspberrypi:~/mysketch# ls
    Makefile  blink.ino

### Uploading code

Sending code the the arduino is simple.  Simply run the command "make upload" and it will compile your .ino file into a hex file
that will be sent to the arduino using a tool called avrdude.  Since you're running this on the command line, you'll see all the 
commands run by the Makefile when you run "make upload".  

      root@raspberrypi:~/mysketch# make upload
      Makefile:535: build-cli/depends.mk: No such file or directory
      echo '#include <Arduino.h>' > build-cli/blink.cpp
      cat  blink.ino >> build-cli/blink.cpp
      /usr/bin/avr-g++ -MM -mmcu=atmega328p -DF_CPU=16000000L -DARDUINO=100 -I. -I/usr/share/arduino/hardware/arduino/cores/arduino -I/usr/share/arduino/hardware/arduino/variants/standard   -g -Os -w -Wall -ffunction-sections -fdata-sections -fno-exceptions build-cli/blink.cpp -MF build-cli/blink.d -MT build-cli/blink.o
      cat build-cli/blink.d > build-cli/depends.mk
      rm build-cli/blink.cpp
      cat build-cli/blink.d > build-cli/depends.mk
      for STTYF in 'stty -F' 'stty --file' 'stty -f' 'stty <' ; \
                        do $STTYF /dev/tty >/dev/null 2>&1 && break ; \
      		done ; \
      		$STTYF /dev/ttyAMA0  hupcl ; \
      		(sleep 0.1 2>/dev/null || sleep 1) ; \
      		$STTYF /dev/ttyAMA0 -hupcl 
      echo '#include <Arduino.h>' > build-cli/blink.cpp
      cat  blink.ino >> build-cli/blink.cpp
      /usr/bin/avr-gcc -mmcu=atmega328p -Wl,--gc-sections -Os -o build-cli/mysketch.elf build-cli/blink.o build-cli/libcore.a  -lc -lm
      /usr/bin/avr-objcopy -O ihex -R .eeprom build-cli/mysketch.elf build-cli/mysketch.hex
      /usr/bin/avrdude -q -V -p atmega328p -C /etc/avrdude.conf -c arduino -b 115200 -P /dev/ttyAMA0  \
      			-U flash:w:build-cli/mysketch.hex:i
      done with autoreset
      avrdude-original: stk500_recv(): programmer is not responding
      
You might see the error message above:  "programmer is not responding".  This is because one must temporarily flip the reset pin on
the arduino before code can be uploaded.  When we wired up the arduino, it was hooked up to one of our GPIO pins.  It's possible to
manually flip this pin every time you upload code, but you'll need to have great reflexes.


### How do we fix the DTR pin?

Visit the next article in the series for [auto-magically flipping the DTR pin](/2012/08/12/fixing-the-dtr-pin/)

