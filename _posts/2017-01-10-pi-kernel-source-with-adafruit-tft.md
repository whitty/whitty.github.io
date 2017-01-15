---
layout: post
title:  "Compiling kernel modules against PiTFT adafruit kernel - Raspberry Pi"
date:   2017-01-10 20:50:12 +1100
categories: linux raspberry-pi
tags: raspberry-pi linux adafruit kernel
---

Adafruit [provides](https://learn.adafruit.com/adafruit-2-8-pitft-capacitive-touch/easy-install) a turn-key distribution and script for getting started quickly with your [PiTFT](https://www.adafruit.com/product/2423).  This is all well and good until you want to start looking under the hood or making use of custom solutions.

One of the more confusing (well I've messed it up) aspects is their use of a custom kernel package.  This completely subverts the typical `rpi-update` methods of updating the kernel.  If you follow any online instructions put them away as soon as you see `rpi-update` - you'll lose your adafruit kernel (and PiTFT).

Now adafruit provides a `curl | sudo bash` type script to install their kernel - but (a) there's no documentation for what it does and (b) you should _never_ **ever** run a `curl | sudo bash` file off the internet.  That's called "getting root".

The script sets up a new apt repository by using the *wrong* method.  Put *this* in `/etc/apt/sources.list.d/adafruit.list` instead:

``` terminal
deb http://apt.adafruit.com/raspbian/ jessie main
```

and *this* in `/etc/apt/preferences.d/adafruit`

``` terminal
Package: *
Pin: origin "apt.adafruit.com"
Pin-Priority: 1001
```

which will add the repository and prefer their versions over the official `rpi` kernels.  After updating you'll see what's going on:

``` terminal
$ apt-cache policy raspberrypi-kernel
raspberrypi-kernel:
  Installed: 1.20161101-1
  Candidate: 1.20161101-1
  Version table:
     1.20161125-1 0
        500 http://archive.raspberrypi.org/debian/ jessie/main armhf Packages
 *** 1.20161101-1 0
       1001 http://apt.adafruit.com/raspbian/ jessie/main armhf Packages
        100 /var/lib/dpkg/status
     1.20161027-1 0
       1001 http://apt.adafruit.com/raspbian/ jessie/main armhf Packages
     1.20161018-1 0
       1001 http://apt.adafruit.com/raspbian/ jessie/main armhf Packages
```

The upstream Raspberry Pi Foundation kernel `1.20161125-1` is being overriden by the pin and adafruit kernel chosen instead.

# Installing kernel source

So looks like its as simple as installing the adafruit version of `raspberrypi-kernel-headers`.  The pin will select the right version as you see below (even though the rpi version is more up-to-date).

``` terminal
$ apt-cache policy raspberrypi-kernel-headers 
raspberrypi-kernel-headers:
  Installed: (none)
  Candidate: 1.20161101-1
  Version table:
     1.20161125-1 0
        500 http://archive.raspberrypi.org/debian/ jessie/main armhf Packages
     1.20161101-1 0
       1001 http://apt.adafruit.com/raspbian/ jessie/main armhf Packages
     1.20161027-1 0
       1001 http://apt.adafruit.com/raspbian/ jessie/main armhf Packages
     1.20161018-1 0
       1001 http://apt.adafruit.com/raspbian/ jessie/main armhf Packages
```

The 1001 variants will be installed ahead of the `raspberrypi.org` ones.

# Compiling

Now the kernel packages seem to be a mess - the typical idiom of `/lib/modules/$(uname -r)/build` doesn't work because the source/kernel version directories don't match up.

``` terminal
$ uname -r
4.4.27-v7+
$ ls  /lib/modules
4.4.26+  4.4.26-v7+  4.4.27+  4.4.27-v7+
$ make -C /lib/modules/4.4.26-v7+/build M=$(PWD) modules
$ insmod ./gpio_keys_device.ko # works despite version mess!
```


## Postscript


You'll need their gpg key to validate the packages - I can't find a key-server for their `78661FA5` key or a signature so here's a fig leaf:

``` terminal
$ wget https://apt.adafruit.com/apt.adafruit.com.gpg.key
$ sha256sum apt.adafruit.com.gpg.key 
1ef59b74783b1423d13ca58eb1e3210d407143b5b50a9e39cf96a52f75a935f6  apt.adafruit.com.gpg.key
$ sha384sum apt.adafruit.com.gpg.key 
92db40b8cddef632ade84cf3bfc54f12338674f28762d83da6d02b6c1b02b93d1d038ae445da2d9ef2b8b2318e1e8dd1  apt.adafruit.com.gpg.key
```

I should really learn how to gpg properly.
