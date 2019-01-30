## Rasbian Jessie Instructions

See the [readme](readme.md) for assembly instructions.

#### SD Card Prep

This project is based on [Raspbian Jessie Lite](https://www.raspberrypi.org/downloads/raspbian/),
release date Nov 21, 2015. The Lite distribution is more appropriate for
an embedded system because it doesn't have a graphical desktop.

With a prepared SD card, preform initial OS configuration as necessary.
Then, use `raspi-config` to set the following under *Advanced Options*:
* Enable Device Tree (for pulse-per-second support)
* Disable serial shell/kernel messages (to repurpose UART for the GPS)

Next fetch and apply system updates:
````
sudo apt-get update
sudo apt-get dist-upgrade -y
````
````
sudo reboot
````

#### Install Packages

You will need

* **pps-tools** for PPS support & testing
* **libcap-dev** and **libssl-dev** for rebuilding NTP

but not any GPS-related utilities (gpsd, ..) because NTP will listen to the
GPS receiver directly. 

````
sudo apt-get install pps-tools libcap-dev libssl-dev
````

#### Enable Kernel Modules

The GPS receiver provides a pulse-per-second (PPS) signal for enhanced
precision. In early 2015, kernel support for PPS via GPIO pins was added
and with Device Tree in Raspbian Jessie, PPS is pretty smoothly integrated.
Just add the overlay statement to `/boot/config.txt`:
````
sudo nano /boot/config.txt
````
````diff
 ...
+## pps-gpio
+##     Enable kernel support for GPS receiver pulse-per-second (PPS) input
+dtoverlay=pps-gpio
````
````
sudo reboot
````

By default, this enables PPS support on GPIO (BCM) 18 and creates a
device `/dev/pps0`. If you want to use a different pin, use the `gpiopin`
argument and Broadcom (GPIO) [pin number](http://pinout.xyz) (for example):
````
dtoverlay=pps-gpio,gpiopin=23
````

#### Test PPS Support

Now you can check the PPS signal using the `ppstest` command. Before you
do, ensure the GPS has signal lock (slow ~15sec LED flashes) because it
won't provide a PPS signal without full lock.

````
sudo ppstest /dev/pps0
````
````
trying PPS source "/dev/pps0"
found PPS source "/dev/pps0"
ok, found 1 source(s), now start fetching data...
source 0 - assert 1450338134.999993693, sequence: 97362 - clear 0.0000000000, sequence: 0
source 0 - assert 1450338135.999994591, sequence: 97363 - clear 0.0000000000, sequence: 0
source 0 - assert 1450338136.999994489, sequence: 97364 - clear 0.0000000000, sequence: 0
````

#### Symlink GPS

The NTP driver expects to listen to `/dev/gps0` and `/dev/gpspps0` so we
create permanent symlinks with some udev rules. 
````
sudo nano /etc/udev/rules.d/10-pps.rules
````
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
`PMTK314` command as described in the 
[command packet](https://www.adafruit.com/datasheets/PMTK%20command%20packet-Complete-C39-A01.pdf)
(pg 12). 
````
sudo crontab -e
````
````
@reboot echo -e '$PMTK314,0,1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0*29\r\n' > /dev/gps0
````

#### Remove DHCP Hook

To prevent the Pi from getting NTP configuration from the router, 
remove `ntp-servers` from the end of the `request` block:
````
sudo nano /etc/dhcp/dhclient.conf
````
````diff
 ...
 request subnet-mask, broadcast-address, time-offset, routers,
         ...
-        rfc3442-classless-static-routes, ntp-servers;
+        rfc3442-classless-static-routes;
 ...
````

#### Rebuild NTP

The repository version (4.2.6.p5+dfsg-7+deb8u1) is out-of-date and lacks
PPS support. It's easy enough to download the latest version and build it. 
Start by getting the required libraries:

````
sudo apt-get install libcap-dev libssl-dev
````

Then download the [latest version of NTP](http://archive.ntp.org/ntp4/ntp-4.2)
and unzip it. (At I write this, it's 4.2.8p4 - editor's note: as of Jan 29, 2019
latest version is 4.2.8p12) 

````
wget http://archive.ntp.org/ntp4/ntp-4.2/ntp-4.2.8p4.tar.gz
tar xvzf ntp-4.2.8p4.tar.gz
... lots of output...
````
````
cd ntp-4.2.8p4
````

These first two next steps take roughly 10 and 20 minutes to complete,
respectively. The `--enable-linuxcaps` flag is required. After the build 
is finished, the new NTP executables are installed (for your user), the 
system NTP is stopped and then we copy the newly built versions over it.

````
./configure --enable-linuxcaps
...lots of output...
````
````
make
...lots of output...
````
````
sudo make install
...lots of output...
````
````
sudo service ntp stop
sudo cp /usr/local/bin/ntp* /usr/bin/
sudo cp /usr/local/sbin/ntp* /usr/sbin/
````

To check that things are working, reboot and look over the boot-up output
for warnings or errors from NTP. Then login and check the running version:

````
ntpq --version
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

  [url-nmea20]: https://www.eecis.udel.edu/~mills/ntp/html/drivers/driver20.html
  [url-ppse]: https://www.raspberrypi.org/forums/viewtopic.php?f=41&t=1970&start=225
  [url-ppsd]: https://www.raspberrypi.org/forums/viewtopic.php?f=41&t=1970&start=225

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


### Enable SAMBA support

Share NTP statistics files via Samba (Windows network shares).

````
sudo apt-get install samba samba-common-bin -y
````
````
sudo nano /etc/samba/smb.conf
````
````diff
 ...
+[data]
+   browseable = yes
+   comment = Data directory
+   create mask = 0700
+   directory mask = 0700
+   only guest = yes
+   path = /home/pi
+   public = yes
+   read only = yes

...comment the rest out: printers, etc...
````
````
sudo /etc/init.d/samba restart
[ ok ] Restarting nmbd (via systemctl): nmbd.service.
[ ok ] Restarting smbd (via systemctl): smbd.service.
[ ok ] Restarting samba-ad-dc (via systemctl): samba-ad-dc.service.
````

This results in a public, read-only share which can be automatically
discovered. (Note: hidden files are exposed too but have the hidden
attribute correctly set.)

* http://elinux.org/R-Pi_NAS
* https://www.raspberrypi.org/forums/viewtopic.php?p=107253



