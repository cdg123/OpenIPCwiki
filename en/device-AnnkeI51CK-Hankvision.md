# Annke I51CK
- [Introduction](#introduction)
  - [Device info](#Device-info)
- [Connectors](#Connectors)
  - [Front side](#Front-side)
  - [Back side](#Back-side)
- [GPIOs](#GPIOs)
  - [Muxing](#Muxing)
  - [SD Card](#SD-Card)
  - [Speaker](#Speaker)
- [Flashing](#Flashing)
  - [Flash memory layout](#Flash-memory-layout)
- [Summary](#Summary)
- [TODO](#TODO)

# Introduction
This article is a record of the steps taken to try and reverse engineer an IP camera labelled as Annke I51CK so OpenIPC software can be loaded with the hope Onvif PTZ functionality can be developed. It was purchased from an online auction as a low cost way to get a camera house however as it turned out to be a Goke Soc and Sony sensor it was decided to attempt to reuse it as is for OpenIPC.

This is definately work in progress and if nothing else it may help you with your own projects

## Device info
| System | Description | Comments |
|-|-|-|
|Hankvision|V6202 IR-IMX335|-|
| SoC | GK7205V300 | |
| Flash | XMC XM25QH128C | Nor 16MB |
| Sensor | Sony IMX335 | i2c 0x34 |
| Audio | MIC + SPK | |
| Storage | Micro SD | |
| Memory | - | 128M (Media 80M) |
| LAN | - | eth0 |
| WiFi | - | - |
| Motors | 2x Stepper | - |
| Dimensions | - | |
| Bootload address | 0x41000000 | - |


## Steps so far:
1) Boot camera as is on physical lan connection run ipscanner to look for device.
2) See if SSH or Telnet access is possible.
3) No SSH but Telnet offer login prompt. Try common passwords none work, very little results in on line searching.
4) Open camera to look for way to connect to the Uart. Fairly easy and 3 pins obvious to try Gnd,tx,rx.
5) Putty used to get access default 115200,8 etc.
6) Bootloader not password protected so looking good
7) Setup NFS server on dev machine with gokenfs as directory for backup.
8) Try to get day1 backup of firmware image but no tftp put or other backup option I can find
9) Try to connect flash programmer to get an image copy, couldn't get good connection so ....
10) Dump rom to memory, then echo to screen, capture to log file and convert to binary image.
11) Check with binwalk
12) editenv bootargs add **single init=/bin/sh** end to drop to Linux recovery shell
13) Look at /etc/init.d scripts to see what is loading and where.
14) Mmmmmm rootfs is squashfs and /tmp/flash is used as overlay it seems
15) Looks like flash partitions are for Server module, Configuration...
16) So lets try booting to root-fs from nfs
17) Root-fs extracted using binwalk, unsquashfs and copied to devpc nfs
18) Make a copy of original bootargs on camera setenv bootargs-day1 $bootargs
19) editenv $bootargs to make rootfs mounted over network
    **bootargs=mem=48M console=ttyAMA0,115200 root=/dev/nfs rootfstype=nfs nfsroot=192.168.1.222:/srv/gokenfs/,v3,nolock,tcp rw ip=192.168.1.16:192.168.1.222:192.168.1.254:255.255.255.0**
21) set env vars for ipaddr, gateway, serverip
22) can change u-boot if need to use openipc but turns out similar on cam already.
23) reboot and works but then locks up as camera software loaded saying no network.
24) so ensure single init=bin/sh saved to bootcmd
25) Success we have a booted system with root-fs running from our NFS share
26) Now run passwd to change root password.
27) Check what IPCTool can tell us, no sensor recognied, need to run rcS script autorun.sh
28) This loads all required drivers and builds on /mnt/flash/Server mount area.
29) Hangs when we get to wathcall app that is executed as looks like network, ip etc all reset in the app.
30) So on our nfs image create our own copy of the autorun.sh script and at the very end part comment out the /root/watchall app.
31) Now run this to see if all the drivers are loaded ok. Obviosly loads everythign for the cam as white leds and infa on.
32) IPCTool now can see sensor so better.
33) Only /dev /proc /sys and /mnt/flash and /mnt/nfs are usable as the rootfs etc is squashfs so
   **mount -o nolock,tcp 192.168.1.222:/srv/gokenfs /mnt/nfs**
  By default mount nfs is trying to connect with UDP (thanks wireshark for helping me to find that) and it is not enabled by default in Ubunto ver
34) Don't want to risk breaking the original firmware by creating new bins and writing to flash so lets try sd card as root-fs
35) Attempt 1 create ext4 partition on flash drive copy all of root-fs from nfs /mnt/gokenfs.
36) Try to create an init script to swap root-fs but just throws back help text
37) Try to mount with -vvv option exposed no ext4 filetype supported only squashfs, exfat etc
38) So create squashfs image and burn to flash card
39) Try to mount first AAARGH doesn't support LZ compression only xd. Try again
40) OK so we can mount this squashfs now lets boot to it
41) Update bootargs  from **mem=48M console=ttyAMA0,115200 root=/dev/nfs rootfstype=nfs nfsroot=192.168.1.222:/srv/gokenfs/,v3,nolock,tcp rw ip=192.168.1.16:192.168.1.222:192.168.1.254:255.255.255.0  mtdparts=sfc:192K(boot),64K(bootargs),1920K(kernel),1408K(rootfs),384K(config),12416K(data)**
 to **bootargs mem=48M console=ttyAMA0,115200 root=/dev/mmcblk0p1 rootfstype=squashfs mtdparts=sfc:192K(boot),64K(bootargs),1920K(kernel),1408K(rootfs),384K(config),12416K(data)**
42) Brainwave add nfs mount to /etc/fstab so we can us dd ip=/mnt/nfs/filename op=/mnt/flash/myflash  (need to create mountpoint myflash)
43) Can we umount the flash card to right to it if that is the root-fs. If not swap the bootargs and make changes then put back
44) So without camera software running watchall app I assume kicks everything off we can remotely telnet in but ip address is changed when server software loads.
45) Need to check the init scripts to make sure network etc come up as without nfs this is not done automagically
46) Cam ip changes when server loaded as weel as root password
47) Need to be careful with IP address changes as can cause arp problems with duplicate ip if not careful using random MAC address a6:94:2f:54:b2:ce
48) So cam now refusing to boot to flash card so will burn again
49) Try mounting nfs to nfsroot can we swap root-fs to there ??


 mount -o nolock,tcp 192.168.1.222:/srv/gokenfs /nfsroot







# Everything below is being used as a template to populate as we go along


## Front side
| Connector | Type |
|:-:|:-|
| IRCUT | 2pin JST |
| LED | 5pin JST |
| MIC | 2pin JST |

## Back side
- Micro SD Card Socket
- UART (unsoldered, to the left of SPK, pin1 RX, pin2 TX)

| Connector | Type |
|:-:|:-|
| SPK | 2pin JST |
| H | 5pin JST |
| V | 5pin JST |
| +5V | 2pin JST |
| RF | UF.L (IPX) |

# GPIOs
| GPIO | Connector | Description |
|:-:|:-:|:-:|
| 0* | - | Reset button |
| 4 | LED pin 5 | WLED |
| 8 | WiFi module pin 3 | LO - Power ON |
| 12 | H pin 5 | Mot H |
| 13 | H pin 2 | Mot H |
| 14 | H pin 4 | Mot H |
| 15 | H pin 3 | Mot H |
| 16 | LED pin 4 | IRLED |
| 52 | V pin 2 | Mot V |
| 53 | V pin 3 | Mot V |
| 54 | V pin 4 | Mot V |
| 55 | V pin 5 | Mot V |
| 56 | IRCUT pin 1 | LO - IRCUT ON |
| 57* | LED pin 3 | IRSens |
| 58 | IRCUT pin 2 | LO - IRCUT OFF |
| 70 | - | SD PWR (LO - Power ON) |
| 51 | - | AUDIO AMP |



## Muxing
No muxing required if Majestic takes control over pins. Otherwise, muxing can be done using the following commands.

Muxing GPIO16 for taking control over IRLED pin:
```sh
devmem 0x120c0020 32 0x432      # GPIO2_0 (GPIO16)
```

Also for motors.  
Muxing GPIO12, GPIO14, GPIO15 (motors H connector):
```sh
devmem 0x120c0010 32 0x1e02     # GPIO1_4 (GPIO12)
devmem 0x120c0018 32 0x1d02     # GPIO1_6 (GPIO14)
devmem 0x120c001c 32 0x1402     # GPIO1_7 (GPIO15)
```

Shortly after **Loading of kernel modules...** GPIO13 turns to HI (one of motors winding constantly powered), so maybe necesary turn it to LO:
```sh
gpio clear 13
gpio unexport 13
```

## SD Card
By default SD Card unpowered, so we need turn GPIO70 to LO somehow.

To poweron SD CARD from Kernel:
```sh
gpio clear 70
```
or
```sh
devmem 0x120B8400 32 0x40       # turn GPIO8_6 to output mode
devmem 0x120B8100 32 0x00       # set GPIO8_6 to LO
```
And reattach SD card.

To poweron SD CARD from U-Boot:
```sh
mw 0x120B8400 0x40      # turn GPIO8_6 to output mode
mw 0x120B8100 0x00      # set GPIO8_6 to LO
mmc rescan
```

## Speaker
Device supports playing PCM signed 16-bit little-endian, 8000 Hz, 1CH by sending data to http://192.168.0.10/play_audio endpoint.

Audio file can be encoded like this:
```sh
ffmpeg -i input.wav -f s16le -ar 8000 -ac 1 output.pcm
```

And send to camera's speaker:
```sh
curl -v -u user:pass -H "Content-Type: application/json" -X POST --data-binary @audio.pcm http://192.168.0.10/play_audio
```

# Flashing
Stock firmware is pwd locked and LAN interface does not present, so I'm guessing following methods are available to flash this board:
- [burn](https://github.com/OpenIPC/burn)  + [u-boot-gk7202v300-universal.bin](https://github.com/OpenIPC/firmware/releases/download/latest/u-boot-gk7202v300-universal.bin) and then upload FW via X/Y/ZMODEM (e.g. **loady**. Tip: use **baud** option for speed up) or from SD card (power supply required, [see above](#SD-Card))
- load full image thru stock web interface (untested)
- flash programmer
- somehow get into stock bootloader

## Flash memory layout
| Offset | Size | Description | 
|:-|:-|:-|
| 0x00000000 | 0x00040000 (262144 bytes) | bootloader |
| 0x00040000 | 0x00010000 (65536 bytes) | env |
| 0x00050000 | 0x00200000 (2097152 bytes) | kernel |
| 0x00250000 | 0x00500000 (5242880 bytes) | rootfs |
| 0x00750000 | 0x000B0000 (720896 bytes) | rootfs_data |

# Summary
- [X] WiFi works
- [X] Video tested/streamed
- [X] Day/night works (IRCUT and IRLED)
- [X] MIC works
- [X] Speaker works
- [ ] PTZ/Motors (GPIO pins found/accessible, driver untested)

# TODO
- somehow patch/adapt camhi-motor.ko, so make it works on this board.  
