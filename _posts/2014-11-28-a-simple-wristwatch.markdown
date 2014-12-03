---
layout: article
title: "A Simple Wristwatch"
date: 2014-11-28T01:57:01-05:00
excerpt: "A simple wristwatch, in progress. Keeps time with a real-time clock,
displays it with lots of little LED's."
image:
  teaser: wristwatch_teaser.jpg

---

In an attempt to master Eagle and surface-mount soldering, I decided to
design a simple LED wristwatch with as small a footprint as I could manage.
As you can see in this [schematic]({{site.url}}/files/wristwatch.sch) (board files
  [here]({{site.url}}/files/wristwatch1.brd)), the
watch runs on an ATMega328P.

![schematic]({{site.url}}/images/wristwatchschematic.png)

I chose this chip because it's the same chip
in an Arduino Uno, and there are many tutorials online for how to upload
code to it via an [Arduino ISP](http://arduino.cc/en/Tutorial/ArduinoISP). I
also decided to use a real time clock with a built-in crystal to keep time. This
RTC is pretty expensive ($8 on mouser?), but keeps time extremely accurately. On
V2, I might use a cheaper RTC or forgo the RTC and add a 16Mhz crystal instead.
At the present, the Arduino is designed to use its own internal 8Hz oscillator as a clock.

I meticulously reviewed the schematics and printed 3 boards for around $10 from
OSHPark.

![oshpark]({{site.url}}/images/wristwatchboards.jpg)
Nonetheless, I found after soldering 90% of the components that **the schematic is broken**!
There's a whole row of LED's that are connected to nothing,
and there's no easy way to get at the clock pin. I tried bootloading the thing
with an Arduino, but haven't had success yet.  News to come with V2.


![bootloading]({{site.url}}/images/bootloading.jpg)
