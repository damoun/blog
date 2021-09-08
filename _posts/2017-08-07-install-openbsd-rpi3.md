---
layou: post
title: Install OpenBSD on a Raspberry Pi 3
categories: [ops]
tags: [openbsd, install, raspberry pi]
---

To get a simple ssh gate at home, I installed OpenBSD 6.1 on a Raspberry Pi 3 with a broken HDMI port.

## Requirements

### Peripherals

To perform the install, you will need the following peripherals :

* a USB key
* a microSD
* a [Adafruit](https://www.adafruit.com/product/954) 954 USB serial cable

### Media install

Connect the microSD to your workstation and write a `miniroot61.fs` on it.

```shell
wget 'https://mirror.dalenys.com/pub/OpenBSD/6.1/arm64/miniroot61.fs'
dd bs=1M if=miniroot61.fs of=/dev/mmcblk0
```

System will be installed on the USB key because OpenBSD 6.1 don't support the microSD port on the Raspberry Pi 3. At the end of the install, let the microSD in the Raspberry Pi 3 because U-Boot save the boot order on it.

### Plug-in serial cable

The Adafruit serial cable have to be plug-in on the GPIO pin of the Raspberry Pi 3 like the following schema (black on GND, white on TX, green on RX).

![How to plug-in serial cable on GPIO](https://developer.android.com/things/images/raspberrypi-console.png)

On your workstation, connect the USB serial cable and run `screen` to get console on the Raspberry Pi 3.

```shell
screen /dev/ttyUSB0 115200
```

## Install

Connect the microSD and the USB key on the Raspberry Pi 3 and turn it on. Be careful that the Raspberry Pi 3 need a 5V/2.5A power supply.
You should see the boot sequence on your screen session and finnally, the Install menu.
Perform the install like a regular one except you have to choose Auto-layout on the partition part. It will format in FAT32 the microSD for the U-Boot settings.

Reboot at the end and on the U-Boot prompt, right the following like to configure boot order.

```shell
setenv boot_targets usb0 mmc0 pxe dhcp
saveenv
boot
```

U-Boot will save his settings on the microSD and boot on the new OpenBSD 6.1.
Don't forget to `syspatch`.