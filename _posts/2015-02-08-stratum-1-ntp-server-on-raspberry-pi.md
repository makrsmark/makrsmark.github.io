***NOTE***: *this blog post was originally published at <https://insideidt.wordpress.com/2015/02/08/stratum-1-ntp-server-on-raspberry-pi/> and was manually converted to markdown*

The web is full of blog posts about how someone with a GPS and a Raspberry Pi created their own Stratum 1 NTP server.
The problem with them is they’re all old.
I found even the ones written because of all of the blog posts were old out of date.
So here’s another post on how I did it.
There’s quite a few advantages to using this guide over others:

* No kernel compiling
* NTP as a Debian package
* Coarse GPS and PPS used

Some may disagree with these benefits, but I wanted them.

Parts List
==========

I bought everything from Adafruit. I could have spared a few bucks by using parts around the house but I wanted everything in one place.

1. [SMA to uFL/u.FL/IPX/IPEX RF Adapter Cable](https://www.adafruit.com/product/851) PID: 851
1. [GPS Antenna &#8211; External Active Antenna &#8211; 3-5V 28dB 5 Meter SMA]("https://www.adafruit.com/product/960) PID: 960
1. [CR1220 12mm Diameter &#8211; 3V Lithium Coin Cell Battery &#8211; CR1220](https://www.adafruit.com/product/380) PID: 380
1. [Brass M2.5 Standoffs for Pi HATs &#8211; Black Plated &#8211; Pack of 2](https://www.adafruit.com/product/2336) PID: 2336
1. [Adafruit Ultimate GPS HAT for Raspberry Pi A+ or B+ &#8211; Mini Kit](https://www.adafruit.com/product/2324) PID: 2324
1. (optional) [Adafruit Raspberry Pi B+ Case &#8211; Smoke Base w/ Clear Top](https://www.adafruit.com/product/2258) PID: 2258
1. [5V 2A Switching Power Supply w/ 20AWG 6&#8242; MicroUSB Cable](https://www.adafruit.com/product/1995) PID: 1995
1. [Raspberry Pi Model B+ 512MB RAM](https://www.adafruit.com/product/1914) PID: 1914
1. [SD/MicroSD Memory Card (4 GB SDHC)](https://www.adafruit.com/product/102) PID: 102


I should note that I bought the Raspberry Pi B+ the day before version 2 was released.
I strongly recommend purchasing the [Raspberry Pi 2 Model B](https://www.adafruit.com/products/2358) for the same price.
Everything in the list will work fine since it has the same layout and pinout as the B+.

The optional case and standoffs I was a little disappointed with.
The standoffs don’t come with screws to secure the hat and the case has its own plastics standoffs that prevent the Pi from securing properly.
One or both of these parts will meet the Dremel shortly.

Prerequisites
=============

I am going to assume the OS is already installed. I chose [Raspbian](http://www.raspbian.org/) but any OS should work.


Soldering
---------

I was a little surprised to find the GPS hat came fully populated except for the header.
The 40 pin header needs to be soldered to the hat. It’s not that hard but if you’re new to soldering I recommend NASA.
See Section 6 of the [NASA Workmanship Standards](http://workmanship.nasa.gov/lib/insp/2%20books/frameset.html)
and their six [instructional videos](http://radiojove.gsfc.nasa.gov/telescope/soldering.htm).

![Remember, Good volcanos!](https://insideidt.files.wordpress.com/2015/02/wp_20150205_11_49_10_pro1.jpg)


Instructions
============

With a working Pi and an assembled GPS hat attached, let’s get started. I should note that I was a terrible person and did everything as root. My install didn’t come with a user account. I’m not sure if this is a change in Rasbian or if I glossed over it. If you’re running as the user ‘pi’ be sure to add ‘sudo’ to any command that is denied.

Expand Filesystem
-----------------

Enough packages will be installed that the Pi will out of space on the stock image. Fix this using ‘raspi-config’ but once again, it wasn’t on my Raspbian install. Follow <a href="http://www.raspberrypi.org/forums/viewtopic.php?f=51&amp;t=45265">this guide</a> for assistance.

Get the latest Kernel
---------------------

Now it's time to upgrade to a kernel with support for PPS interrupts. Thankfully newer versions have this built in. 
There is no need to build a custom kernel.
~~~
apt-get update
~~~

~~~
apt-get dist-upgrade
~~~

~~~
rpi-update
~~~


Then reboot.

Turn off Serial console
-----------------------

open up _/boot/cmdline.txt_ and remove `console=ttyAMA0,115200 kgdboc=ttyAMA0,115200` from it. save and close.

Enable Kernel PPS Interrupts
----------------------------

Several packages need to be installed.
~~~
apt-get install pps-tools libcap-dev libssl-dev
~~~



Libssl-dev may not be needed, but I am showing it anyway


Open _/boot/config.txt_ and add `dtoverlay=pps-gpio,gpiopin=4` on a new line


Make sure the GPIO pin is the one your PPS output is hooked up to. The Adafruit GPS hat is 4.


Open _/etc/modules_ and add `pps-gpio` on a new line.


Reboot. To verify everything is correct, first run:
~~~
lsmod | grep pps
~~~



The modules pps_gpio and pps_core loaded. If those modules are running,verify PPS is working by running:
~~~
dmesg | grep pps
~~~



The output should be similar to this:
~~~
[ 10.159940] pps_core: LinuxPPS API ver. 1 registered<
[ 10.161448] pps_core: Software ver. 5.3.6 - Copyright 2005-2007 Rodolfo Giometti <giometti@linux.it>
[ 10.172015] pps pps0: new PPS source pps-gpio.18
[ 10.173557] pps pps0: Registered IRQ 188 as PPS source
~~~

If only the first two lines are visible, the wrong pin is probably selected. Go back and verify /boot/config.txt is correct.

Install GPS software
--------------------

There's a few packages needed to read the GPS.
~~~
apt-get install gpsd gpsd-clients python-gps
~~~



then test the GPS. First, make sure gpsd is reading the serial port.
~~~
gpsd/dev/ttyAMA0-F/var/run/gpsd.sock
~~~



To verify the GPS working, run either ‘gpsmon’ or ‘cgps -s’. If everything looks good, reconfigure gpsd:
~~~
dpkg-reconfigure gpsd
~~~



Tell gpsd to start on boot, use /dev/ttyAMA0 and point to /var/run/gpsd.sock just like our test command.

![gpsmon output](https://insideidt.files.wordpress.com/2015/02/gpsstatus2.jpg)

Build NTP with PPS
------------------

The current version of NTP may vary but here we go…

* Install the necessary packages:
~~~
apt-get build-dep ntp
~~~


* Get the sources:
~~~
apt-get source ntp
~~~

* Modify debian/rules:

add `--enable-ATOM` and `--enable-linuxcaps` to configure call. You may need only the first, but I used both.

* Bump version number in debian/changelog to be just below the next version (using "~"):
e.g. change 4.2.6.p5+dfsg-2 to 4.2.6.p5+dfsg-3~pps1

*  Recompile the package:
~~~
dpkg-buildpackage -b
~~~


*Install the new binary:
~~~
dpkg -i ntp_4.2.6.p5+dfsg-3~pps1_armhf.deb
~~~


Configure NTP
-------------


NTP will not read the GPS and PPS signals by default.
* If DHCP is enabled, remove `ntp-servers` from the `request` line in /etc/dhcp/dhclient.conf
1. Remove _/var/lib/ntp/ntp.conf.dhcp_ if present
1. Edit _/etc/ntp.conf_ to add:

~~~
# pps-gpio on /dev/pps0
server 127.127.22.0 minpoll 4 maxpoll 4
fudge 127.127.22.0 refid PPS
fudge 127.127.22.0 flag3 1 # enable kernel PLL/FLL clock discipline
# gpsd shared memory clock
server 127.127.28.0 minpoll 4 maxpoll 4 prefer # PPS requires at least one preferred peer
fudge 127.127.28.0 refid GPSfudge 127.127.28.0 time1 0.000 # coarse processing delay offset
server ntp1.ptb.de iburst prefer # another stable preferred peer change this to someone near you
~~~

restart NTP. To verify everything is correct, run
~~~
ntpq -pn
~~~



The PPS and GPS signals should appear.


![ntp output](https://insideidt.files.wordpress.com/2015/02/ntpstatus2.jpg)

Future
======

There's some things that I'd like to cover/do that aren't here:

* NTP status monitoring
* Build at least two more and have them reference each other
* Use GLONASS for one of the time servers
* How to configure your network to use your NTP server.


Resources
=========

Sites I did not explicitly call out but used. I may have lifted some text directly from them.

* I used 2 articles from satsignal.eu <http://www.satsignal.eu/ntp/Raspberry-Pi-NTP.html> and <http://www.satsignal.eu/ntp/Raspberry-Pi-quickstart.html>
* <http://ntpi.openchaos.org/pps_pi/>
* <https://learn.adafruit.com/adafruit-ultimate-gps-hat-for-raspberry-pi>
* many others that got lost in the many browser tabs open for this project.
