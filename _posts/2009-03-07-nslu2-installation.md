---
layout: post
permalink: /:year/:month/:day/:title/
title: "nslu2 installation"
date: "2009-03-07"
tags: 
  - "linux"
  - "technology"
---

I need to change the HDD of my Linksys NSLU2 so I will use the opportunity to write a small HowTo to show how to set up an unslung firmware into my nslu2. Let's start

First of all I need to download the last version of [Unslung](http://www.slug-firmware.net/u-dls.php) firmware and install in my ubuntu box the software to put the firmware into my nslu2:

```shell
sudo aptitude install upslug2
```

I put my nslu2 into [upgrade mode](http://www.nslu2-linux.org/wiki/HowTo/UseTheResetButtonToEnterUpgradeMode) and ...

```shell
# upslug2
 [no NSLU2 machines found in upgrade mode]
```

What happen? I always forget I have two network cards (eth0 and eth1) and I use eth1

```
upslug2 -help
upslug2: usage: upslug2 {options}
options:
[extract]
-d --device[eth0]: local ethernet device to use
-t --target: NSLU2 to upgrade (MAC address)
-i --image: complete flash image to use
```

so ...

```
sudo upslug2 -d eth1 -t  00:13:10:d9:03:ff -i Unslung-6.10-beta.bin
```
network is a my home network is a 192.168.2.0/24 so I need to change my IP if I want to connect to my nslu2:

```
sudo ifconfig eth1 192.168.1.2 netmask 255.255.255.0 up

```

Now I open a browser and type http://192.168.1.77/

From the web interface I change the ip from 192.168.1.77 to 192.168.2.77, the gateway and the dns.

change again my IP

```
sudo ifconfig eth1 192.168.2.2 netmask 255.255.255.0 up
```

and open again a browser http://192.168.2.77/

Now I format plug the HDD to the device (into Disk1) and format it from the web inteface.

Now I enable telnet, telnet to the device with the default username/password (root/uNSLUng) and unsling disk1

```
telnet 192.168.2.77
Trying 192.168.2.77...
Connected to 192.168.2.77.
Escape character is '^]'.
LKGD903B8 login: root
Password:
Welcome to Unslung V2.3R63-uNSLUng-6.10-beta
-------- NOTE: RUNNING FROM INTERNAL FLASH  --------
This system is currently running from the internal flash memory,
it has NOT booted up into "unslung" mode from an external drive.
In this mode, very few services are running, and available disk
space is extremely limited.  This mode is normally only used
for initial installation, and system maintenance and recovery.
BusyBox v1.3.1 (2007-12-29 03:38:35 UTC) Built-in shell (ash)
Enter 'help' for a list of built-in commands.
 
# unsling disk1
Waiting for /share/hdd/data ...
Target disk is /share/hdd/data
Checking that /share/hdd/data has been properly formatted...
Checking that /share/hdd/data is clean...
Please enter the new root password.  This will be the new root
password used when the NSLU2 boots up with or without disks
Changing password for root
Enter the new password (minimum of 5, maximum of 8 characters)
Please use a combination of upper and lower case letters and numbers.
Enter new password:
Re-enter new password:
Password changed.
Copying the complete rootfs from / to /share/hdd/data ...
(this will take just a couple of minutes)
Copy complete ...
Linking /usr/bin/ipkg executable on target disk.
Linking /etc/motd to the unslung motd on target disk.
Updating /home/httpd/html/home.htm with target disk info.
Creating /.sdb1root to direct switchbox to boot from /share/hdd/data.
Unsling complete.
Leave the device disk1, /dev/sdb1, plugged in and reboot (using
either the Web GUI, or the command "DO_Reboot") in order to boot
this system up into unslung mode.
# Connection closed by foreign host.
```

Reboot the device enable telnet again and install openssh:

```
# ipkg update
# ipkg install openssh
```
