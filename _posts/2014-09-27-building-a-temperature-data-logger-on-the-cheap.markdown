---
layout: post
status: publish
published: true
title: Building a temperature data logger on the cheap
author_login: tim
author_email: tim@tim-smith.us
wordpress_id: 142
wordpress_url: http://blog.tim-smith.us/?p=142
date: !binary |-
  MjAxNC0wOS0yNyAxOToyMjoxOCAtMDcwMA==
date_gmt: !binary |-
  MjAxNC0wOS0yOCAwMjoyMjoxOCAtMDcwMA==
categories:
- my research
tags: []
comments: []
---
I needed an accurate and inexpensive temperature logger to calibrate our stage incubator for live cell microscopy and make sure that it was stable over time, so I&nbsp;built one. This was a very quick project, using a $30 <a href="http://www.dataq.com/products/di-145/">Dataq DI-145</a>&nbsp;data logger and a $4 <a href="https://www.adafruit.com/products/372">epoxy-coated thermistor</a>&nbsp;from Adafruit. (To power the thermistor from USB, which is safe and easy, add a <a href="https://www.adafruit.com/product/1833">micro-USB breakout board</a>, a matching USB cable, and a small solderless breadboard for the complete package!)

<p>Thermistors have a resistance that depend on temperature. By placing a thermistor in a voltage divider configuration with a known resistance, you can measure a voltage, determine the resistance, and then use a lookup table or a formula to calculate the temperature. I used the 10 Kohm resistor Adafruit included as my reference resistor.</p>
<p>We wanted to calculate and display the temperature in real time. The WinDAQ software that Dataq offers for free doesn't&nbsp;offer this feature (you can apply a linear scale but not an arbitrary function) and doesn't run on OS X or Linux.&nbsp;Happily, the DI-145 has a <a href="http://www.dataq.com/resources/techinfo/di-145-protocol-rev1-07.html">very friendly</a>&nbsp;ASCII serial protocol. A&nbsp;<a href="https://github.com/tdsmith/labmisc/blob/master/twiddle.py">short Python script</a>&nbsp;using pyserial lets us read the voltage divider voltage and the value of Vcc off the DI-145, compute the resistance, and then calculate the&nbsp;temperature using a numpy spline interpolation function against <a href="https://github.com/tdsmith/labmisc/blob/master/therm_lookup.csv">the lookup table</a>&nbsp;from Adafruit.</p>
<p>We've observed that we can measure temperature to about 0.25 &deg;C, which is good enough for us. Using a higher Vcc or a data logger with higher resolution would improve our precision but would have added cost and complexity.</p>
<p>I tested the logger&nbsp;by sticking the probe in a bucket of melting ice, which is an easy temperature reference for 0 &deg;C. Here's what it looks like:<br />
<img class="alignnone" src="https://dl.dropboxusercontent.com/u/3720/junk/templogger.png" alt="" width="731" height="387" /></p>
<p>So the system&nbsp;looks pretty accurate! I probably should have tested boiling water as well, but I didn't get around to it.</p>
<p>To actually run my <a href="https://github.com/tdsmith/labmisc/blob/master/twiddle.py">twiddle.py</a> code, hook Vcc to input 1, the voltage divider output (I put the thermistor on top and connected the 10K reference to ground) to input 2, and give it the name of the serial port on the command line. On Windows, this looks like "python twiddle.py COM6"; on Linux (and OS X, I think), you'll use something from /dev instead of COM6. We like to run it with tee so that the temperature is both displayed on-screen and written to a file, like "python twiddle.py COM6 | tee templog.txt".</p>
<p>Happy hacking!</p>
