---
weight: 999
title: "Pine64 ClusterBoard - Part1"
description: ""
icon: "article"
date: "2023-09-21T13:58:39-04:00"
lastmod: "2023-09-21T13:58:39-04:00"
draft: true
toc: true
---

## Console Connection ##
Most of this information was obtained from the [Pine64 Forum](https://forum.pine64.org/showthread.php?tid=7285)  the diagram / images were no longer stored on the forum and I've added information for clarity.

I'm using the [SERIAL CONSOLE “Woodpecker” Edition](https://store.pine64.org/?product=padi-serial-console) board to connect to the ClusterBoard's UART GPIO slot 

| Woodpecker| ClusterBoard slot GPIO (7 slots)|
|--- | ---|
| GND Brown | Pin 6 GND|
| RTX Red | Pin 7 PL10 G362 (E) S_PWM |
| TXD White | Pin 8 	P8 PB0 G32 (E) UART2_TX |

Plug "Woodpecker" into an available USB port on a workstation.  Identify the device /dev/ttyUSB#, and connect a Screen instance:
```bash
screen /dev/ttyUSB0 115200
```
Power on the ClusterBoard, it looks like a SD card is required to begin the boot process.  The console should display the boot log as the process proceeds.  In my case the Armbian image did not boot.
## Boot Image ##
I initially I tried an Armbian Buster image for the [Sopine64 board](https://www.armbian.com/sopine-a64/).  I got output, but the system ultimately ended up in the uboot environment.  The first step in the diagnostic process is capturing the boot log messages for a little more detail, this will most likely be done with screen.  There appear to be three possible images / systems that may be possible to boot.  The most likely image Armbian Sopine, and, possibility Alpine. Ultimately K3os with replace the root partition image.

ARM64 images seem to include DTB files that I believe define the components need for particular boards.  The Armbian image includes the file "sun50i-a64-sophie-baseboard.dtb" this certainly looks encouraging.
### Alpine  ###
Most of the package for the K3os install are from this disro, so it may be the best option for a bootable kernel.
### Armbian ###
This is the most likely bootable image.  The image booted with not network, this may be a known issue. Resetting the board and reconnecting the network connection seem to resolve the problem.  There maybe some I/O errors on the mSD card.

I've used a few combinations of images and techniques for  the boot process, however many have failed somewhere alone the line.  This is what I have so far:
* sopine1 is up using Armbian, but I don't remember which image. (Probably Bionic)
* sopine0  
  Armbian Bionic image for Pine64so on an mSSD.  This seems to boot.  The same image on the eMMC hangs at "Starting kernel...".  It looks like U-Boot is working but "boot.scr" seems to be incorrect.

Another option for the base system seems to be the [OpenEnbedded](https://wiki.pine64.org/index.php/SOPINE_Software_Release#OpenEmbedded.2FYocto_Images) build sysetm. 

### K3os ###
Ultimately this is the image that should be installed in the cluster and will be managed by the K3s system, but it looks like it will require a bootable kernel for installation.

Running the following on a bootable Armbian images seems to have worked.  There is no console messages once the image switches to the root partition, and no tty console, however ssh with the configured rancher user (config.yaml) worked.
```bash
curl -sfL https://github.com/rancher/k3os/releases/download/v0.8.0/k3os-rootfs-arm64.tar.gz| tar zxvf - --strip-components=1 -C /
cp myconfig.yaml /k3os/system/config.yaml
sync
reboot -f
```
config.yaml  
```yaml
ssh_authorized_keys:
- ssh-rsa <...>= sbrown@bianchi
hostname: sopine#
run_cmd:
- "echo hi && echo bye"
boot_cmd:
- "echo hi && echo bye"
init_cmd:
- "echo hi && echo bye"

k3os:
  data_sources:
  - cdrom
  dns_nameservers:
  - 8.8.8.8
  - 1.1.1.1
  ntp_servers:
  - 0.us.pool.ntp.org
  - 1.us.pool.ntp.org
  password: rancher
```

Add config.yaml to the worker node's /var/lib/rancher/k3os directory to join the cluster
```conf
k3os:
  server_url: https://sopine0.6browns.org:6443
  token: K101263c79eda3447e2e671f4f59a99914a989c44684dfc2fd121bbb3cc57d4b0e8::server:0d33b0dbf4499e27bafd7a28cd99a2c8
```

### Rio ###
Rio is a layered product on the Rancher K3s infrastructure that provides a micro Platform as a Service (uPaaS).  There are many features included in the platform including Prometheus and Grafana for monitoring.