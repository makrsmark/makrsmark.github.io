Stratum 1 NTP server and ADS-B Feeder on Le Potato

Parts List
==========
Needed the following. Links are provided if i bought it. Otherwise, i used what i had.

1. [Le Potato with Heatsink](https://www.amazon.com/Libre-Computer-Potato-Single-Heatsink/dp/B0BQG668P6)
1. [GPS Hat with RTC](https://v3.airspy.us/product/upu-rpi-gps-rtc/)
1. [Magnetic GPS Antenna](https://v3.airspy.us/product/ant-gps-m/)'
1. SD Card
1. MicroUSB Power Supply


*Note*: i bought without the heatsink and ended up purchasing it separately as the potato can get hot very easily and thermal throttle.

Prerequisites
=============

I am going to assume the OS is already installed. 
I chose the 
[Raspbian](http://www.raspbian.org/https://hub.libre.computer/t/raspbian-11-bullseye-for-libre-computer-boards/82)built
image built by libre computer in the hopes of setting it up like my pis, but i learned it didn't matter. 
Next time i'd probably use plain Debian or Ubuntu.



Hardware Setup
==============
The GPS hat I chose has both a GPS module and RTC.
Since the Le Potato is configured differently than the Raspberry Pi, the provided instructions don't work.
Luckily, I figured it out for you. Instead of using `raspi-config` there's the wiring tool to modify the kernel tree.

Configure PPS
-------------
To use the PPS pin, you need to use a kernel that will allow hardware interrupts. 
You will also need to enable a kernel module that configures the correct pin.
Run these two commands:
~~~
sudo apt install linux-image-6.1.y-lts-irq-arm64
sudo ldto merge pps-gpio-7j1-12
~~~

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
[    0.664052] pps_core: LinuxPPS API ver. 1 registered
[    0.664067] pps_core: Software ver. 5.3.6 - Copyright 2005-2007 Rodolfo Giometti <giometti@linux.it>
[    9.741317] pps pps0: new PPS source pps-gpio.-1
[    9.741394] pps pps0: Registered IRQ 32 as PPS source
~~~

If only the first two lines are visible, the wrong pin is probably selected. Go back and verify /boot/config.txt is correct.


Configure RTC
-------------
To enable to i2c bus on the header, run
~~~
sudo ldto merge i2c-ao
~~~

If you're luckly, you should can enable the rtc device driver by running
~~~
sudo ldto merge i2c-ao-rv3028
~~~

I was not so luckly and had to create my own.

To do that, you'll need to build the wiring tool yourself

~~~
git clone https://github.com/libre-computer-project/libretech-wiring-tool
cd libretech-wiring-tool
make
sudo ./ldto merge i2c-ao-rv3028
~~~

Reboot.

Verify you have a clock at _/dev/rtc0_. you can test it by running

~~~
sudo hwclock -R
~~~


Software
========
now that the hardware is configured and recognized by the kernel, let's put it to work


GPSD
----

There's a few packages needed to read the GPS.
~~~
apt-get install gpsd gpsd-clients
~~~

Configure gpsd to read both the serial port (_/dev/ttyAML6_) and pps pin (_/dev/pps0_)
Also add the options `-n -s 115200`
Th `-n` flag has gpsd run without any clients and `-s 115200` sets the serial port speed.

Your _/etc/default/gpsd_ should look like:
~~~
# Devices gpsd should collect to at boot time.
# They need to be read/writeable, either by user gpsd or the group dialout.
DEVICES="/dev/ttyAML6 /dev/pps0"

# Other options you want to pass to gpsd
GPSD_OPTIONS="-n -s 115200"

# Automatically hot add/remove USB GPS devices via gpsdctl
USBAUTO="true"
~~~

Chrony
------
Previously, I've used ntpd as my NTP server. This time, I'm using chrony

create _/etc/chrony/conf.d/gpsd.conf_ and add the following line to it
~~~
refclock SOCK /run/chrony.pps0.sock refid PPS delay 0.0
~~~

this tells chrony to create a socket file for gpsd.
Restart chrony and gpsd (in that order)

verify everything is running with
~~~
chronyc sources
~~~

The PPS refclock should appear.

To enable ntp clients, you'll haveto specifically allow it. Create _/etc/chrony/conf.d/clients.d_ and populate it with what IPs you wish to allow.
For example:
~~~
# allow local traffic to query
allow 192.168/16
allow fe80::/10

# allow all
allow
~~~

restart chrony, and you should be able to connect from another server.
