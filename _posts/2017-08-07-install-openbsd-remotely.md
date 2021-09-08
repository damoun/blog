---
layou: post
title: Install OpenBSD on a remote server
categories: [ops]
tags: [openbsd, install, server]
---

This week-end, I was upgrading my secondary dns server from OpenBSD 6.0 to OpenBSD 6.1 on a Digi ONE server from [Digicube](https://www.digicube.fr/), a french hosting provider.

## Install

From the console of your hoster, you need to reboot your server in a rescue mode to get a fresh debian running in Ramdisk.
When yout server is finnaly up in rescue, connect with ssh. To perform, you have to install qemu and download the `miniroot61.fs`.

```shell
apt-get install qemu
wget 'https://mirror.dalenys.com/pub/OpenBSD/6.1/amd64/miniroot61.fs'
```

You can now start the qemu virtual machine with the `miniroot61.fs` and the hard drive and follow the install process. On the file sets selection, add the multi-process kernel `bsd.mp`.

```shell
qemu-system-x86_64 -curses -drive file=miniroot61.fs -drive file=/dev/sda -net nic -net user
```

Don't forget to change the interface at the end of the install, because the qemu network interface is different than the Realtek network interface.

```shell
mv /mnt/etc/hostname.em0 /mnt/etc/hostname.re0
```

On the first boot, run `syspatch` to patch the base system with ERRATA.

## Troubleshooting

### Syspatch - Unable to access bsd.sp

On my first run of `syspatch`, it couldn't find bsd.sp. I needed to move `/bsd.syspatch61` to `/bsd.sp` after reverting every patches.

```shell
while true; do syspatch -r || break; done
mv /bsd.syspatch61 /bsd.sp
syspatch
```
