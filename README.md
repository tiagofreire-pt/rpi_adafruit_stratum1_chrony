# Raspberry PI 3B+ NTP Server Stratum 1 with Adafruit GPS HAT
A straightforward approach to achieve a cost-effective Stratum 1 NTP server using a Raspberry Pi 3B+ and an Adafruit Ultimate GPS HAT MTK3339.

This is my recipe for Raspbian Bullseye, kernel 5.10.103-v7+.

Achievements @ April 2022:
- [X] ns local clock timekeeping (std dev 95 ns)
- [X] µs timekeeping across multiple networks (std dev 35 µs)
- [X] stable operation with low frequency value (< 10 ppm)
- [X] serve time to more than 160 clients
- [ ] correct the timekeeping deviation from ambient temperature flutuation
- [ ] replace the fake RPI RTC with a DS3231 high precision one.

## Upgrade your system and install the required software
> sudo apt update && sudo apt upgrade -y
> 
> sudo apt install gpsd gpsd-tools gpsd-clients pps-tools chrony setserial -y


## Disable the serial TTY (linux console) on the UART interface
> sudo systemctl stop serial-getty@ttyAMA0.service
> 
> sudo systemctl disable serial-getty@ttyAMA0.service
> 
> sudo systemctl disable hciuart

## Disable the kernel support for the serial TTY:
> sudo nano /boot/cmdline.txt

remove this sequence only and save: 
```console=serial0,115200```

## Configure the Raspberry PI

Add this to your '/boot/config.txt' file

```
# Use the /dev/ttyAMA0 UART GPS instead of Bluetooth
dtoverlay=disable-bt

# https://hallard.me/enable-serial-port-on-raspberry-pi/
dtoverlay=miniuart-bt

# Disable Wifi for better accuracy and lower interferance
dtoverlay=disable-wifi

# Enable UART serial - http://www.philrandal.co.uk/blog/archives/2019/04/entry_213.html
#enable_uart=1

# enable GPS PPS
dtoverlay=pps-gpio,gpiopin=4

# Disable kernel power saving
nohz=off

# Disable Energy Efficient Ethernet - improves jitter and lag (~200us)
dtparam=eee=off

# Force CPU high speed clock
force_turbo=1
```

## Remove the support to receive NTP servers through DHCP
> sudo rm /etc/dhcp/dhclient-exit-hooks.d/ntp
> 
> sudo rm /lib/dhcpcd/dhcpcd-hooks/50-ntp.conf
> 
> sudo nano /etc/dhcp/dhclient.conf

Remove the references for `dhcp6.sntp-servers` and `ntp-servers`

## Decrease the serial latency for improved accuracy and stability
> sudo nano /etc/udev/rules.d/gps.rules

Add the content:

```
KERNEL=="ttyAMA0", RUN+="/bin/setserial /dev/ttyAMA0 low_latency"
```

## Force the CPU governor from boot, being always 'perfomance'
> sudo nano /etc/init.d/raspi-config

Replace all the content with:
```
#!/bin/sh
### BEGIN INIT INFO
# Provides:          raspi-config
# Required-Start: udev mountkernfs $remote_fs
# Required-Stop:
# Default-Start: S 2 3 4 5
# Default-Stop:
# Short-Description: Switch to ondemand cpu governor (unless shift key is pressed)
# Description:
### END INIT INFO

. /lib/lsb/init-functions

case "$1" in
  start)
    log_daemon_msg "Checking if shift key is held down"
    if [ -x /usr/sbin/thd ] && timeout 1 thd --dump /dev/input/event* | grep -q "LEFTSHIFT\|RIGHTSHIFT"; then
      printf " Yes. Not modifiing the scaling governor"
      log_end_msg 0
    else
      printf " No. Switching to performance scaling governor"
      SYS_CPUFREQ_GOVERNOR=/sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
      if [ -e $SYS_CPUFREQ_GOVERNOR ]; then
        echo "performance" > $SYS_CPUFREQ_GOVERNOR
        echo 10 > /sys/devices/system/cpu/cpufreq/ondemand/up_threshold
        echo 100000 > /sys/devices/system/cpu/cpufreq/ondemand/sampling_rate
        echo 10 > /sys/devices/system/cpu/cpufreq/ondemand/sampling_down_factor
      fi
      log_end_msg 0
    fi
    ;;
  stop)
    ;;
  restart)
    ;;
  force-reload)
    ;;
  *)
    echo "Usage: $0 start" >&2
    exit 3
    ;;
esac

```

## Reboot to apply the system configurations
> sudo reboot

## Setup the GPSd daemon
> sudo nano /etc/default/gpsd

Replace all the content with:
```
START_DAEMON=”true”
USBAUTO=”false”
DEVICES=”/dev/ttyAMA0 /dev/pps0″
GPSD_OPTIONS=”-n”
```
> sudo systemctl restart gpsd

## Setup chrony as the service for the NTP server
> sudo nano /etc/chrony/chrony.conf 

Replace all the content with:

```
# Welcome to the chrony configuration file. See chrony.conf(5) for more
# information about usable directives.

# Include configuration files found in /etc/chrony/conf.d.
confdir /etc/chrony/conf.d

# Use Debian vendor zone.
#pool 2.debian.pool.ntp.org iburst

# Use the Portuguese zone ** CHANGE THIS **
pool 0.pt.pool.ntp.org iburst minpoll 5 maxpoll 5

# Use time sources from DHCP.
#sourcedir /run/chrony-dhcp

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
log tracking measurements statistics tempcomp

# Log files location.
logdir /var/log/chrony

# Stop bad estimates upsetting machine clock.
maxupdateskew 100.0

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

# Defining the networks allowed to access the service
allow

# Expedited Forwarding DSCP directive traffic
dscp 48

# set larger delay to allow the NMEA source to overlap with
# the other sources and avoid the falseticker status
#refclock SOCK /tmp/chrony.ttyAMA0.sock refid GPS precision 1e-1 offset 0.9999
#refclock SOCK /tmp/chrony.pps0.sock refid PPS precision 1e-7


# set larger delay to allow the NMEA source to overlap with
# the other sources and avoid the falseticker status
refclock SHM 0 refid GPS precision 1e-1 offset 0.47 delay 0.2

# (4ns per foot)-  http://www.philrandal.co.uk/blog/archives/2019/04/entry_213.html
refclock SHM 1 refid PPS precision 1e-7 prefer offset 65.62e-9

# https://chrony.tuxfamily.org/doc/4.2/chrony.conf.html#tempcomp
# Compara a temperatura real com a de correção a cada 30 segundos
tempcomp /sys/class/thermal/thermal_zone0/temp 30 /etc/chrony/chrony.tempcomp
```

> sudo nano /etc/chrony/chrony.tempcomp 
#### Add the content:
```
20000 0
21000 0
25000 0
30000 0
40000 0
```

> sudo systemctl restart chronyd.service

## Check the sources for correct operation
> watch chronyc sources -v


# References
- http://www.gregledet.net/computers/building-a-stratum-1-ntp-server-with-a-raspberry-pi-4-and-adafruit-ultimate-gps-hat/
- https://wiki.polaire.nl/doku.php?id=dragino_lora_gps_hat_ntp
- http://www.philrandal.co.uk/blog/archives/2019/04/entry_213.html
- https://chrony.tuxfamily.org/doc/4.2/chrony.conf.html#tempcomp
