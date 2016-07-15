# HDBitT_hdmi_extender
My notes and tools from reverse engineering the HDbitT HDMI extender

# Acquisition

I purchased this thing as a receiver/transmitter pair from [Amazon](https://www.amazon.com/gp/product/B01C9CI1B6/).  It was marketed as ` LKV373A HDMI Extender over Ethernet up to 120m / 390 ft (new ver.)`.  The box itself has no brand, model numbers, or part numbers, only a "V3.0" label.

At the time of purchase, there were no Amazon reviews, but some documentation from other reverse engineers which led me to believe this would be a fun project:
 - [TimVideos](https://github.com/timvideos/HDMI2USB/wiki/Alternatives#lenkeng-hdmi-over-ip-extender)
 - [Danman's Blog - Reverse Engineering Lenkeng HDMI over IP Extender](https://blog.danman.eu/reverse-engineering-lenkeng-hdmi-over-ip-extender/)

# First boot

When the transmitter boots, it tries to get an IP via DHCP, but falls back to 192.168.1.238.  It also sends out a bit of IGMP/multicast control traffic before it begins transmitting the video stream via UDP multicast to 239.255.42.42:5004.  The packets are consistently 1370 bytes, regardless of whether it has an HDMI signal.

The transmitter responds to pings on 192.168.1.238.

# nmap scan

```
root@kali:~/hdmi# nmap -T5 192.168.1.238

Starting Nmap 6.49BETA4 ( https://nmap.org ) at 2016-07-14 21:52 CDT
Warning: 192.168.1.238 giving up on port because retransmission cap hit (2).
Nmap scan report for 192.168.1.238
Host is up (1.0s latency).
Not shown: 996 closed ports
PORT     STATE    SERVICE
80/tcp   open     http
514/tcp  filtered shell
7000/tcp open     afs3-fileserver
7002/tcp open     afs3-prserver

Nmap done: 1 IP address (1 host up) scanned in 5.32 seconds
```

# HTTP Server

The box has a kludgey web server which allows for upgrading the "firmware" and "encoder firmware".  My transmitter arrived with the following firmware versions:

```
Version : 3.0.0.0.20151028
Encoder Version : 7.1.2.0.9.20151028
```

The upgrades appear to be packaged with pre-specified file extensions.  Firmware is a .PKG file; Encoder firmware is a .BIN file.

Looking through the source of the page, there is a CGI script at `/dev/info.cgi` which does all the heavy lifting.  It is controlled through a GET variable (`action`) which, at a minimum, can be set to `macaddr`, `upgrade`, `reboot`, `Reset`, `network`, `softap`, and `wifi`:

- `macaddr` allows the end user to change the MAC address of the device.  Validation takes place client-side, and there is likely a command injection here if the device is using `ifconfig`
- `reset` reverts the device to factory defaults.  A time GET parameter (`t`) is given, and a command injection may be possible, provided that `time` is fed to a sleep command.
- `reboot` presumably calls reboot, and also includes a time GET parameter (`t`) which may provide a command injection.  The reboot would try to occur first, which would cause issues with the command injection.
- `network` takes two GET parameters: `ipaddr0` and `netmask0`, which are validated client-side.  There is likely command injection here, assuming use of `ifconfig` as in `macaddr` above.
- `upgrade` uses an IFRAME (`iframeupload`).  (*TODO*: Look into the upgrade function.  Find a firmware package to inspect.)
- `softap` and `wifi` suggest that there is another device which has WiFi functionality to connect to a Wifi network or serve as an ad-hoc node.
