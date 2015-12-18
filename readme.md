Rasbian NTP Server
==================

Setup guide for a Raspberry Pi-based NTP server. 

### Bill of materials

Substitutions may apply. Your mileage may vary. Additional equipment
required for setup (i.e. keyboard, mouse, monitor).

| Description                                              | $USD (est) |
|----------------------------------------------------------|------------|
| [Raspberry Pi Model 1 B+][bom0]                          |        30  |
| [Adafruit Ultimate GPS Breakout version 3][bom1]         |        40  |
| [CR1220 Lithium Coin Cell Battery 3V][bom2]              |         1  |
| [SMA to uFL/u.FL/IPX/IPEX RF Adapter Cable][bom3]        |         4  |
| [GPS Antenna, External, Active, 3-5V 28dB 5m SMA][bom4]  |        13  |
| [4GB SD Card (w/ Rasbian Jessie Lite)][bom5]             |        10  |
| cheap boxy case                                          |         8  |
|                                                  *Total* |       106  |

  [bom0]: https://www.adafruit.com/products/1914
  [bom1]: https://www.adafruit.com/products/746
  [bom2]: https://www.adafruit.com/products/380
  [bom3]: https://www.adafruit.com/products/851
  [bom4]: https://www.adafruit.com/products/960
  [bom5]: https://www.adafruit.com/products/2820

### Setup

#### Assembly

![Hardware diagram](images/hardware-diagram.png)

#### SD Card Prep

This project is based on [Raspbian Jessie Lite][url-rpidl] (2015-11-21). 
The Lite distribution is more appropriate for embedded systems, which also 
means it doesn't have a graphical system -- command line only.

If using a blank/used SD card, use your preferred [install method][url-imgin]
to write the image. A Linux user could do as follows, assuming the image has
been downloaded into `~/Downloads`, then unzipped; *and* the SD card is 
unmounted and has a device address of `/dev/sdb`:

````
user@comp:~$ ls Downloads
2015-11-21-raspbian-jessie-lite.img
2015-11-21-raspbian-jessie-lite.zip
user@comp:~$ sudo dd bs=4M if=Downloads/2015-11-21-raspbian-jessie-lite.img of=/dev/sdb
user@comp:~$ sudo sync
````

  [url-rpidl]: https://www.raspberrypi.org/downloads/raspbian/
  [url-imgin]: https://www.raspberrypi.org/documentation/installation/installing-images/README.md

#### Initial OS Setup

Run `sudo raspi-config` after logging in (`pi`/`raspberry`) to start the
initial setup utility.

> *If you have already connected the GPS to the Pi, you may see horrible
> errors covering your screen after boot. Try pressing 'Enter' several
> times to get a visible login prompt.*

* Expand the file system
    * Yes
* Internationalization
    * Set your preferred locale (we use `en_US.UTF-8`)
    * Set your time zone (we use `GMT+8` for Pacific Standard Time year-round;
      the inverted sign (+8) is a POSIX quirk)
    * Set your keyboard (we use US English)
* Overclock
    * 800 Mhz (modest)
    * (or don't, it's not required)
* Advanced Options
    * Set the hostname (say, `ntpi`)
    * Enable SSH (if you use it)
    * Enable Device Tree (yes, for pulse-per-second support)
    * Disable serial shell/kernel messages (to repurpose UART for the GPS)
* Exit saving changes and reboot

Next fetch and apply system updates:

````
pi@ntp:~ $ sudo apt-get update
pi@ntp:~ $ sudo apt-get upgrade
pi@ntp:~ $ sudo apt-get dist-upgrade # I actually skip this     FIXME
pi@ntp:~ $ sudo rpi-update # I don't do this either             FIXME
````

#### Install Packages

You will need

* **pps-tools** for PPS support & testing
* **libcap-dev** and **libssl-dev** for rebuilding NTP

but not any GPS-related utilities (gpsd, ..) because NTP will listen to the
GPS receiver directly. 

````
pi@ntp:~ $ sudo apt-get install pps-tools libcap-dev libssl-dev
````

#### Enable Kernel Modules

The GPS receiver provides a pulse-per-second (PPS) signal for enhanced
precision. In early 2015, kernel support for PPS via GPIO pins was added
and with Device Tree in Raspbian Jessie, PPS is pretty smoothly integrated.
Just add the overlay statement to `/boot/config.txt`:

````
pi@ntp:~ $ sudo nano /boot/config.txt
````
````
...
dtoverlay=pps-gpio
````

By default, this enables PPS support on GPIO (BCM) 18 and creates a
device `/dev/pps0`. If you want to use a different pin, use the `gpiopin`
argument and the Broadcom (GPIO) [pin number][url-pinout]:

  [url-pinout]: http://pinout.xyz

````
...
dtoverlay=pps-gpio,gpiopin=23
````

> You could alternately enable PPS support by adding this text to
> `/boot/cmdline.txt`. We picked `/boot/config.txt` for the extra 
> breathing room :D -- choose one or the other.

#### Reboot

````
pi@ntp:~ $ sudo reboot
````

#### Test PPS Support

Now you can check the PPS signal using the `ppstest` command. Before you
do, ensure the GPS has signal lock (slow ~15sec LED flashes) because it
won't provide a PPS signal without full lock.

````
pi@ntp:~ $ sudo ppstest /dev/pps0
trying PPS source "/dev/pps0"
found PPS source "/dev/pps0"
ok, found 1 source(s), now start fetching data...
source 0 - assert 1450338134.999993693, sequence: 97362 - clear 0.0000000000, sequence: 0
source 0 - assert 1450338135.999994591, sequence: 97363 - clear 0.0000000000, sequence: 0
source 0 - assert 1450338136.999994489, sequence: 97364 - clear 0.0000000000, sequence: 0
````

#### Symlink GPS

The NTP driver expects to listen to `/dev/gps0` and `/dev/gpspps0` so we
create permanent symlinks with some udev rules. These rules go into a new
text file: `sudo nano /etc/udev/rules.d/10-pps.rules`

````
KERNEL=="ttyAMA0", SUBSYSTEM=="tty", GROUP="dialout", MODE="0660", SYMLINK+="gps0"
KERNEL=="pps0", SUBSYSTEM=="pps", GROUP="dialout", MODE="0660", SYMLINK+="gpspps0"
````

Reboot and test the new symlinks:

* `cat /dev/gps0`
* `sudo ppstest /dev/gpspps0`

#### Configure GPS module

The Adafruit Ultimate GPS Breakout v3 sends 5 NMEA sentences by default. These
extra sentences introduce unnecessary processing and increase jitter. 

To disable all but the Recommended Minimum GPS data sentence (GPRMC), issue the 
`PMTK314` command as described in the [command packet][url-pmtk] (pg 12). 
(You will need a [checksum calculator][url-chksm] to construct the command.)

  [url-pmtk]: https://www.adafruit.com/datasheets/PMTK%20command%20packet-Complete-C39-A01.pdf
  [url-chksm]: http://www.hhhh.org/wiml/proj/nmeaxor.html

Run `sudo crontab -e`, add this line and reboot:

````
@reboot echo -e '$PMTK314,0,1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0*29\r\n' > /dev/gps0
````

#### Remove DHCP Hook

To prevent the Pi from getting NTP configuration from the router, 
remove `ntp-servers` from the end of the `request` block in the 
file `/etc/dhcp/dhclient.conf`:

````
...
request subnet-mask, broadcast-address, time-offset, routers,
        ...
        rfc3442-classless-static-routes; # removed ntp-servers
...
````

#### Rebuild NTP

The repository version (4.2.6.p5+dfsg-7+deb8u1) is out-of-date and lacks
PPS support. It's easy enough to download the latest version and build it. 
Start by getting the required libraries:

````
pi@ntp:~ $ sudo apt-get install libcap-dev libssl-dev
````

Then download the [latest version of NTP](http://archive.ntp.org/ntp4/ntp-4.2)
and unzip it. (At I write this, it's 4.2.8p4) 

````
pi@ntp:~ $ wget http://archive.ntp.org/ntp4/ntp-4.2/ntp-4.2.8p4.tar.gz
pi@ntp:~ $ tar xvzf ntp-4.2.8p4.tar.gz
... lots of output...
pi@ntp:~ $ cd ntp-4.2.8p4
pi@ntp:~/ntp-4.2.8p4 $
````

These first two next steps take roughly 10 and 20 minutes to complete,
respectively. The `--enable-linuxcaps` flag is required. After the build 
is finished, the new NTP executables are installed (for your user), the 
system NTP is stopped and then we copy the newly built versions over it.

````
pi@ntp:~/ntp-4.2.8p4 $ ./configure --enable-linuxcaps
...
pi@ntp:~/ntp-4.2.8p4 $ make
...
pi@ntp:~/ntp-4.2.8p4 $ sudo make install
...
pi@ntp:~/ntp-4.2.8p4 $ sudo service ntp stop
pi@ntp:~/ntp-4.2.8p4 $ sudo cp /usr/local/bin/ntp* /usr/bin/
pi@ntp:~/ntp-4.2.8p4 $ sudo cp /usr/local/sbin/ntp* /usr/sbin/
````

To check that things are working, reboot and look over the boot-up output
for warnings or errors from NTP. Then login and check the running version:

````
pi@ntp:~ $ ntpq --version
ntpq 4.2.8p4@1.3265 Tue Dec 15 17:42:50 UTC 2015 (1)
````

#### Configure NTP

Finally, modify the NTP configuration to use the local GPS+PPS:
`sudo nano /etc/ntp.conf`

````
server 127.127.20.0 mode 17 prefer
fudge 127.127.20.0 flag1 1 time2 0.350 refid GPPS
````

* The odd-looking IP address specifies a [generic NMEA][url-nmea20] driver
  with the local device `/dev/gps0`. 
* `mode` accepts a bitmask: 0x01 (1) to only process $GPRMC sentences and 
  0x10 (16) for 9600 bps baud rate.
* `prefer` gives higher weight to the GPS; it isn't strictly necessary
* `flag1 1` enables PPS processing; it's required
* `time2 0.350` specifies the serial end of line time offset (seconds); it's
  just a rough value used to disambiguate the PPS edge timing
* `refid GPPS` is a tweak of the default refid `GPS` to indicate PPS support

These flags are not required because you use the default values:

* `flag2`: capture of PPS on the rising (0, default) or falling (1) edge  
  *GlobalTop states PPS output is rising edge [2013-10-18 13:19][url-ppse]*
* `flag3`: for PPS, use ntpd clock (0, default) or kernel (1) discipline  
  *Linux PPS expects ntpd clock discipline, not kernel (hardpps)
  discipline [2013-10-16 19:24][url-ppsd]*

NTP requires multiple servers to determine the time and recognize bad clocks.
Your file should specify a few other sources, optionally with the `iburst`
argument to speed up initialization (recommended). This argument isn't
available for the GPS.

> *The `pool` directive is often recommended but didn't work for me.*

````
server 0.us.pool.ntp.org iburst
server 1.us.pool.ntp.org
server 2.us.pool.ntp.org
server 3.us.pool.ntp.org
````

  [url-nmea20]: https://www.eecis.udel.edu/~mills/ntp/html/drivers/driver20.html
  [url-ppse]: https://www.raspberrypi.org/forums/viewtopic.php?f=41&t=1970&start=225
  [url-ppsd]: https://www.raspberrypi.org/forums/viewtopic.php?f=41&t=1970&start=225

### Further Tuning

There's two other kernel tweaks to consider for improved/more stable
performance. To enable, add to new lines in `/boot/config.txt`.

* `force_turbo=1` Disables dynamic clocking, so NTP needs to react less
* `smsc95xx.turbo_mode=0` Reduces Ethernet latency by doing something to
  decouple it from the USB

### References

This project is hugely indebted to a handful of well-treaded resources.
In no particular order, and possibly not comprehensively:

* http://www.satsignal.eu/ntp/RaspberryPi-notes.html#JorgeAmaralThoughts
* http://www.satsignal.eu/ntp/Raspberry-Pi-NTP.html#compile-ntp
* http://www.satsignal.eu/ntp/Raspberry-Pi-quickstart.html
* http://www.satsignal.eu/ntp/Raspberry_Time%20-%20Broadband%20Ham%20Net.pdf
* http://linuxpps.org/wiki/index.php/LinuxPPS_NTPD_support
* https://www.eecis.udel.edu/~mills/ntp/html/refclock.html
* https://www.raspberrypi.org/forums/viewtopic.php?f=9&t=1970
* http://open.konspyre.org/blog/2012/10/18/raspberry-flavored-time-a-ntp-server-on-your-pi-tethered-to-a-gps-unit/



