# Raspberry Pi 3B+ NTP Server - Stratum 1 (with Adafruit Ultimate GPS HAT)
A straightforward and optimized approach to achieve a cost-effective (€100) Stratum 1 NTP server, coordinated with highly precise PPS (Pulse Per Second) sourced from the GPS radio service plus NTP public servers across the internet to get the absolute time reference.

Can be prepared to be used with *off-the-grid* applications such as IoT in remote locations/air-gapped systems or WAN connected IoT ones (as presented here).

The end result with a Raspberry Pi 3B+ and an Adafruit Ultimate GPS HAT MTK3339:

![The Server Fully Assembled](./img/rpi_fully_assembled.jpg)

This is my recipe for Raspberry Pi lite OS `Bookworm`, kernel 6.1.72-v8+.

## Achievements @ January 2024:
- [X] ns local clock timekeeping (std dev < 200 ns on PPS source)
- [X] µs timekeeping across multiple networks (RMS offset < 100 ns)
- [X] stable operation with low frequency value (usually ~ 5 ppm)
- [X] serve time to more than 160 clients (capable of many more)
- [X] optimize the MK3339 for NMEA timming only
- [X] increase the serial baudrate to its maximum (up to 57600 bps)
- [ ] correct the timekeeping skew from CPU temperature flutuation

![Chrony Source Statistics after 1 week of uptime](./img/chrony_sourcestats_apr_2022.JPG)

## Checklist aiming a low latency and jitter environment @ January 2024:
- [X] Research system hardware topology, using lscpu 
- [X] Determine which CPU sockets and I/O slots are directly connected.
- [X] Follow hardware manufacturer's guidelines for low latency hardware tuning.
- [X] Ensure that adapter cards are installed in the most performant I/O.
- [X] Ensure that CPU/memory/storage is installed and operating at its **nominal** supported frequency.
- [X] Make sure the OS is fully updated.
- [X] Enable network-latency tuned overlay settings.
- [X] Verify that power management settings are correct and properly setup.
- [X] Stop all unnecessary services/processes.
- [ ] Unload unnecessary kernel modules *(to be assessed)*
- [X] Apply low-latency kernel command line setup(s).
- [X] Perform baseline latency tests.
- [X] Iterate, making isolated tuning changes, testing between each change.


# List of materials and tools needed

**Mandatory**:
- Soldering iron to attach the pins to the Adafruit HAT
- SD Card with 8GB or more
- USB SD Card reader or other similar device to install Raspberry PI OS on the SD Card.
- Raspberry PI 3B+ with a suitable power adaptor
- Adafruit Ultimate GPS HAT
- RJ45 Ethernet CAT5 (or better) cable with proper lenght

**Optional** :
- 3D printed case for housing the fully assembled server **(RPI 3B+)**:
  > I suggest this [top](./stl/case_top_custom.stl) and this [bottom](https://www.thingiverse.com/thing:4200246) parts.
  > PLA or PETG are generally appropriate, depending on the ambient temperature and environment you'll apply this server in.
- 4x 2.5mm X 10mm bolts and nuts
- CR1220 battery for the MK3339
- Outdoor GPS active antenna with 28dB Gain, inline powered at 3-5V DC, with 5 meters of cable lenght and SMA male connector
- SMA female connector to IPEX (UFL) Adapter

# Setup the server

## Upgrade your system and install the required software

### For Ultimate GPS HAT from Adafruit 
> sudo apt update && sudo apt upgrade -y
> 
> sudo apt install gpsd gpsd-tools gpsd-clients pps-tools chrony setserial -y


## Disable the serial TTY (linux console) on the UART interface
> sudo systemctl disable --now serial-getty@ttyAMA0.service
> 
> sudo systemctl disable --now hciuart


## Disable the kernel support for the serial TTY
> sudo nano /boot/cmdline.txt

Remove this ```console=serial0,115200``` and this ```kgdboc=ttyAMA0,115200``` (if applicable) sequence(s) only and save.


## Configure the Raspberry Pi

Add this to your `/boot/config.txt` file:

```
[pi3+]
# Purposely made empty for advanced system tuning

[all]
# Uses the /dev/ttyAMA0 UART GPS instead of Bluetooth
dtoverlay=miniuart-bt

# Disables Bluetooth for better accuracy and lower interferance - optional
dtoverlay=disable-bt

# Disables Wifi for better accuracy and lower interferance - optional
dtoverlay=disable-wifi

# For Ultimate GPS HAT from Adafruit 
dtoverlay=pps-gpio,gpiopin=4

# Disables kernel power saving
nohz=off

# Disables Energy Efficient Ethernet - improves jitter and lag (~200us)
dtparam=eee=off

# Force CPU high speed clock
force_turbo=1
```

## Remove the support to receive NTP servers through DHCP
> sudo rm /etc/dhcp/dhclient-exit-hooks.d/timesyncd
> 
> sudo nano /etc/dhcp/dhclient.conf

Remove the references for `dhcp6.sntp-servers` and `ntp-servers`

## Disable and stop systemd-timesyncd to eliminte conflicts with chrony later on
> sudo systemctl disable --now systemd-timesyncd

## Decrease the serial latency for improved accuracy and stability
> sudo nano /etc/udev/rules.d/gps.rules

Add the content:

```
KERNEL=="ttyAMA0", RUN+="/bin/setserial /dev/ttyAMA0 low_latency"
```

## Force the CPU governor from boot, being always 'performance', aiming better timekeeping resolution
> sudo sed -i 's/CPU_DEFAULT_GOVERNOR="\${CPU_DEFAULT_GOVERNOR:-ondemand}"/CPU_DEFAULT_GOVERNOR="\${CPU_DEFAULT_GOVERNOR:-performance}"/; s/CPU_ONDEMAND_UP_THRESHOLD="\${CPU_ONDEMAND_UP_THRESHOLD:-50}"/CPU_ONDEMAND_UP_THRESHOLD="\${CPU_ONDEMAND_UP_THRESHOLD:-10}"/; s/CPU_ONDEMAND_DOWN_SAMPLING_FACTOR="\${CPU_ONDEMAND_DOWN_SAMPLING_FACTOR:-50}"/CPU_ONDEMAND_DOWN_SAMPLING_FACTOR="\${CPU_ONDEMAND_DOWN_SAMPLING_FACTOR:-10}"/' /etc/init.d/raspi-config

## Reboot to apply the system configurations
> sudo reboot

## Setup the GPSd daemon
> sudo nano /etc/default/gpsd

Replace all the content with - Ultimate GPS HAT from Adafruit :
```
START_DAEMON="true"
USBAUTO="false"
DEVICES="/dev/ttyAMA0 /dev/pps0″
GPSD_OPTIONS="--nowait --passive --speed 9600"
```

## Restart the GPSd service
> sudo systemctl restart gpsd


## Setup chrony as the service for the NTP server
> sudo nano /etc/chrony/chrony.conf 

Replace all the content with:

```
# Welcome to the chrony configuration file. See chrony.conf(5) for more
# information about usable directives.

# Include configuration files found in /etc/chrony/conf.d.
confdir /etc/chrony/conf.d

# ** CHANGE THIS ** -- DISABLE THIS FOR ISOLATED/AIRGAPED SYSTEMS
pool 0.pool.ntp.org iburst minpoll 5 maxpoll 5 polltarget 16 maxdelay 0.030 maxdelaydevratio 2 maxsources 6
pool 1.pool.ntp.org iburst minpoll 5 maxpoll 5 polltarget 16 maxdelay 0.030 maxdelaydevratio 2 maxsources 6

# ENABLE THIS FOR ISOLATED/AIRGAPED SYSTEMS
#cmdport 0

# Use NTP sources found in /etc/chrony/sources.d.
sourcedir /etc/chrony/sources.d

# This directive specify the location of the file containing ID/key pairs for
# NTP authentication.
keyfile /etc/chrony/chrony.keys

# This directive specify the file into which chronyd will store the rate
# information.
driftfile /var/lib/chrony/chrony.drift

# Save NTS keys and cookies.
ntsdumpdir /var/lib/chrony

# Set the NTS intermediate certificates
#ntsserverkey /etc/pki/tls/private/foo.example.net.key
#ntsservercert /etc/pki/tls/certs/foo.example.net.crt
#ntsratelimit interval 3 burst 1 leak 2


# Uncomment the following line to turn logging on.
#log tracking measurements statistics 
log rawmeasurements measurements statistics tracking refclocks tempcomp

# Log files location.
logdir /var/log/chrony

# The lock_all directive will lock chronyd into RAM so that it will
# never be paged out. This mode is only supported on Linux. This
# directive uses the Linux mlockall() system call to prevent chronyd
# from ever being swapped out. This should result in lower and more
# consistent latency.
lock_all

# Stop bad estimates upsetting machine clock.
maxupdateskew 100.0

# Use it as reference during chrony startup in case the clock needs a large adjustment.
# The 1 indicates that if the system’s error is found to be 1 second or less, a slew will be used to correct it; if the error is above 1 secods, a step will be used.
initstepslew 1 time.facebook.com time.google.com

# enables response rate limiting for NTP packets - reduce the response rate for IP addresses sending packets on average more than once per 2 seconds, or sending packets in bursts of more than 16 packets, by up to 75% (with default leak of 2).
ratelimit interval 1 burst 16 leak 2

# specifies the maximum amount of memory that chronyd is allowed to allocate for logging of client accesses and the state that chronyd as an NTP server needs to support the interleaved mode for its clients. 
# 1GB
clientloglimit 10000000

# This directive enables kernel synchronisation (every 11 minutes) of the
# real-time clock. Note that it can’t be used along with the 'rtcfile' directive.
rtcsync

# Step the system clock instead of slewing it if the adjustment is larger than
# one second, but only in the first three clock updates.
makestep 1 3

# Get TAI-UTC offset and leap seconds from the system tz database.
# This directive must be commented out when using time sources serving
# leap-smeared time.
leapsectz right/UTC

# Defining the networks allowed to access the service - DISABLE THIS FOR ISOLATED/AIRGAPED SYSTEMS
allow

# Expedited Forwarding DSCP directive traffic
dscp 48

# set larger delay to allow the NMEA source to overlap with
# the other sources and avoid the falseticker status
# https://chrony.tuxfamily.org/faq.html#using-pps-refclock
refclock SHM 0 poll 8 refid GPS precision 1e-1 offset 0.502 delay 0.2 noselect

# (4ns per foot)-  http://www.philrandal.co.uk/blog/archives/2019/04/entry_213.html
#refclock SHM 1 refid PPS precision 1e-7 prefer offset 65.62e-9
refclock PPS /dev/pps0 lock GPS maxlockage 2 poll 4 refid PPS precision 1e-7 prefer offset 65.62e-9

# Compares and saves the SoC temperature with the temperature correlation table bellow, every 30 seconds - future improvement
tempcomp /sys/class/thermal/thermal_zone0/temp 30 /etc/chrony/chrony.tempcomp
```

## Create a simple and innocuous temperature calibration file for chrony
> sudo nano /etc/chrony/chrony.tempcomp 

Add the content:

```
20000 0
21000 0
25000 0
30000 0
35000 0
40000 0
45000 0
50000 0
55000 0
60000 0
65000 0
```

## Restart the chrony service
> sudo systemctl restart chronyd.service


## Check the sources for correct operation
> watch chronyc sources -v

> [!IMPORTANT]
> Wait for 15 minutes, at least, allowing the system clock to converge into a proper offset range, around sub milisecond.

# Advanced Adafruit MK3339 chip tuning

> [!WARNING] 
> This is optional! Proceed with caution and at your own risk!

## Set NMEA sentences for timming only

Change the NMEA settings, sending exclusivly the timing one (`$GPRMC`), reducing the jitter on the `GPS` `refclock`:

> [!IMPORTANT]
> This setting might be not fully successful due to the way gpsd MTK-3301 driver works. Your mileage may vary.

Stop the gpsd service:
> sudo service gpsd stop

Set the instruction:

```echo -e '$PMTK314,0,1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0*29\r\n' > /dev/ttyAMA0```

Restart the gpsd service:
> sudo service gpsd start


### Revert to the default setting

Although the new setting is non-persistant, you might need to revert to the default one:
> echo -e '$PMTK314,-1*04\r\n' > /dev/ttyAMA0

> [!TIP]
> As a failsafe, you might power it off and remove the CR1220 battery for a few minutes.

## Set the MK3339 baudrate to its maximum

This reduces even further the NMEA sentences processing latency, by increasing the transmission speed.

Stop the gpsd service:
> sudo service gpsd stop

Set the baudrate to 57600 bps:
> echo -e '$PMTK251,57600*2C\r\n' > /dev/ttyAMA0

### Setup the GPSd daemon
> sudo nano /etc/default/gpsd

Replace all the content with:
```
START_DAEMON="true"
USBAUTO="false"
DEVICES="/dev/ttyAMA0 /dev/pps0″
GPSD_OPTIONS="--nowait --passive --speed 57600"
```
Restart the GPSd service

> sudo systemctl restart gpsd

### Revert to the default setting

Although the new setting is non-persistant, you might need to revert to the default one:
> echo -e '$PMTK251,9600*17\r\n' > /dev/ttyAMA0

> sudo nano /etc/default/gpsd

Replace all the content with:
```
START_DAEMON="true"
USBAUTO="false"
DEVICES="/dev/ttyAMA0 /dev/pps0″
GPSD_OPTIONS="--nowait --passive --speed 9600"
```
Restart the GPSd service:
> sudo systemctl restart gpsd

> [!TIP]
> As a failsafe, you might power it off and remove the CR1220 battery for a few minutes.

# Advanced system tuning

> [!WARNING] 
> This is optional! Proceed with caution and at your own risk!

## Check and acknowledge the the pinout of these GPS expansions
- Ultimate GPS HAT from Adafruit: https://pinout.xyz/pinout/ultimate_gps_hat

## Check and acknowledge the logical topology of your particular SoC setup, on the PNG image generated
> sudo apt update && sudo apt install hwloc -y
>
> lstopo --logical --output-format png > \`hostname\`.png

## Improve Chrony process priority, using systemd
Due to the Chrony software has not the mechanism to reduce itself its `nice` process value, we'll force it through systemd:

> sudo sed -i '/\[Service\]/a Nice=-10' /usr/lib/systemd/system/chrony.service
>
> sudo systemctl daemon-relead
> 
> sudo systemctl restart chrony

## Disable and stop unnecessary services, reducing cpu time consumption, latency and jitter
> sudo systemctl disable --now alsa-restore.service
>
> sudo systemctl disable --now alsa-state.service
>
> sudo systemctl disable --now alsa-utils.service
>
> sudo systemctl disable --now apt-daily-upgrade.timer
>
> sudo systemctl disable --now apt-daily.timer
>
> sudo systemctl mask apt-daily-upgrade.service
>
> sudo systemctl mask apt-daily.service
>
> sudo systemctl disable --now avahi-daemon.service
>
> sudo systemctl disable --now bluetooth.service
>
> sudo systemctl disable --now bthelper@.service
>
> sudo systemctl disable --now triggerhappy.service
>
> sudo systemctl disable --now rpi-display-backlight.service
>
> sudo systemctl disable --now systemd-timesyncd.service
>
> sudo systemctl disable --now wpa_supplicant.service
>
> sudo systemctl disable --now x11-common.service

## Allocate the strictly minimum RAM from GPU to the OS system, as running headless
Add this to your `/boot/config.txt` file under the `[ALL]`section:

```
# Allocates the base minimum gpu memory, as running headless
gpu_mem=16mb
``` 

## De-activates sound, aiming less resources and latency expected, as running headless
Add this to your `/boot/config.txt` file under the `[ALL]`section:

```
# De-activates sound, aiming less resources, fewer latency and interferance expected, for a headless server
dtparam=audio=off
```

## Set the correct SDCard clock only if using UHS-I cards - RPI 3B+ only
Check for markings about `UHS-I`, `I`, `U1` or `U3`. If they exist, go ahead **(at your own risk)**.

Add this to your `/boot/config.txt` file under the `[pi3+]`section:

```
[pi3+]
## Set the SDcard clock on 100 MHz, for UHS-I specs if appropriate.
dtparam=sd_overclock=100
```

## Auto-restart 10 seconds after a Kernel panic
The Raspberry Pi OS does not have this setting, useful in extreme cases, forcing a full system restart.

> echo "kernel.panic = 10" | sudo tee /etc/sysctl.d/90-kernelpanic-reboot.conf >/dev/null


## Remove the uneeded support for 2G/3G/4G/5G modems

> sudo apt remove --purge modemmanager -y
> 
> sudo apt autoremove --purge -y

## Disable the support for Swap
> sudo nano /boot/cmdline.txt

Add this ```noswap```, after this ```rootfstype=ext4```, and save.

## Disable sdcard swapping for improving its lifespan and reducing unnecessary I/O latency
> sudo dphys-swapfile swapoff
>
> sudo dphys-swapfile uninstall
>
> sudo update-rc.d dphys-swapfile remove
>
> sudo reboot

That's all! :-)

# References
- http://www.gregledet.net/computers/building-a-stratum-1-ntp-server-with-a-raspberry-pi-4-and-adafruit-ultimate-gps-hat/
- https://wiki.polaire.nl/doku.php?id=dragino_lora_gps_hat_ntp
- http://www.philrandal.co.uk/blog/archives/2019/04/entry_213.html
- https://chrony.tuxfamily.org/doc/4.2/chrony.conf.html#tempcomp
- https://hallard.me/enable-serial-port-on-raspberry-pi/
- https://www.thingiverse.com/thing:2980860 *(top case part original design, not available anymore)*
- https://www.thingiverse.com/thing:4200246
- https://gpsd.gitlab.io/gpsd/gpsd-time-service-howto.html#_arp_is_the_sound_of_your_server_choking
- https://dimon.ca/how-to-build-own-stratum-1-ntp-server/#h.1kdm8ehjrplc
- https://psychogun.github.io/docs/linux/Stratum-1-NTP-Server-using-Raspberry-Pi/
- https://chrony.tuxfamily.org/faq.html#_how_can_i_improve_the_accuracy_of_the_system_clock_with_ntp_sources
- https://tf.nist.gov/general/pdf/2871.pdf
- https://chrony-project.org/comparison.html *(good reference on why I choose chrony over ntpd)*
- https://dotat.at/@/2023-05-26-whence-time.html *(time chain of service)*
- https://raspberrypi.stackexchange.com/posts/comments/216264 *(MTK-3301 gpsd driver undoing the setting of NMEA sentences for timming only)*
- https://www.dzombak.com/blog/2023/12/Mitigating-hardware-firmware-driver-instability-on-the-Raspberry-Pi.html
- https://www.dzombak.com/blog/2023/12/Disable-or-remove-unneeded-services-and-software-to-help-keep-your-Raspberry-Pi-online.html
- https://www.dzombak.com/blog/2023/12/Stop-using-the-Raspberry-Pi-s-SD-card-for-swap.html
- https://access.redhat.com/sites/default/files/attachments/201501-perf-brief-low-latency-tuning-rhel7-v2.1.pdf *(impressive guide aiming low latency on linux OS)*
- https://blog.dan.drown.org/nic-interrupt-coalesce-impact-on-ntp/
