---
layout: post
title:  "Raspberry Pi Bluetooth Audio - first connection notes"
date:   2016-11-23 2016 20:39:26 +1100
categories: linux raspberry-pi
tags: raspberry-pi bluetooth linux
---

I have a little project for the pi that would benefit from bluetooth
audio.  I played around about a year ago with a cheapy non-BLE USB
bluetooth dongle using the pi's built-in desktop support for
bluetooth.  It was not a success.  Fast-forward to today where I've
just gotten a pi3 in the post, with built-in bluetooth and Wi-Fi.

Built-in bluetooth support ought to mean more stable bluetooth stack,
and a bunch of
[notes on the Arch wiki](https://wiki.archlinux.org/index.php/Bluetooth_headset),
so things should be good.

So lets see what's happening:
 
``` terminal
$ sudo apt-get install pi-bluetooth
Reading package lists... Done
Building dependency tree        
Reading state information... Done
pulseaudio-module-bluetooth is already the newest version.
0 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
```

Nothing to install from base `jessie` September 2016.  Nice.

```terminal
$ bluetoothctl
[bluetooth]# scan on
Discovery started
[CHG] Controller B8:27:EB:55:B9:3E Discovering: yes
[CHG] Device 8C:C8:CD:88:FB:00 RSSI: -59
[DEL] Device 8C:C8:CD:88:FB:00 DTVBluetooth
[NEW] Device 8C:C8:CD:88:FB:00 DTVBluetooth
[NEW] Device 00:11:67:01:06:C0 BH-04A
[bluetooth]# pair 00:11:67:01:06:C0
Attempting to pair with 00:11:67:01:06:C0
[CHG] Device 00:11:67:01:06:C0 Connected: yes
[CHG] Device 00:11:67:01:06:C0 Modalias: bluetooth:v0039p13A4d0104
[CHG] Device 00:11:67:01:06:C0 UUIDs:
	00001108-0000-1000-8000-00805f9b34fb
	0000110b-0000-1000-8000-00805f9b34fb
	0000110c-0000-1000-8000-00805f9b34fb
	0000110e-0000-1000-8000-00805f9b34fb
	0000111e-0000-1000-8000-00805f9b34fb
	0000112e-0000-1000-8000-00805f9b34fb
	00001200-0000-1000-8000-00805f9b34fb
[CHG] Device 00:11:67:01:06:C0 Paired: yes
Pairing successful
[CHG] Device 00:11:67:01:06:C0 Connected: no
[bluetooth]# connect 00:11:67:01:06:C0
Attempting to connect to 00:11:67:01:06:C0
Failed to connect: org.bluez.Error.Failed
[bluetooth]# 
```

Hmmm,...  Connection fails

```
Nov 23 21:21:57 pi5 bluetoothd[650]: a2dp-sink profile connect failed for 00:11:67:01:06:C0: Protocol not available
```

Turns out this is because bluez doesn't support audio directly it
needs another service to provide audio support.  Currently this is
pulseaudio - alsa would have been simpler, but apparently that is no
longer supported in bluez.

Pulse audio is normally tied to a login session so this is destined to
become a pain.  It turns out there is an embedded appropriate non-user
session support for pulseaudio, but they
[spend a great deal of time explaining why its unsupported](https://www.freedesktop.org/wiki/Software/PulseAudio/Documentation/User/WhatIsWrongWithSystemWide/).
This doesn't bode well.

There's also no real documentation on how you should go about
system-wide (probably because its not supported).  The best reference
I found was [here](https://possiblelossofprecision.net/?p=1956), but
doesnt' have too much overlap with the bluetooth stuff.  Lets hope it
pans out.

```terminal
$ sudo apt-get install pulseaudio-module-bluetooth
```

Nothing to install - that's promising.

From the arch wiki link we find some pulseaudio modules are needed.
The arch link refers to them with the name `bluez5`, but I assume
that's just old guff.  Added the following lines to the bottom of
`/etc/pulse/system.pa`:

```
load-module module-bluetooth-policy
load-module module-bluetooth-discover
```

Now see what happens when it starts

```terminal
$ sudo pulseaudio --system --realtime --disallow-exit
W: [pulseaudio] main.c: Running in system mode, but --disallow-module-loading not set!
N: [pulseaudio] main.c: Running in system mode, forcibly disabling SHM mode!
N: [pulseaudio] main.c: Running in system mode, forcibly disabling exit idle time!
W: [pulseaudio] main.c: OK, so you are running PA in system mode. Please note that you most likely shouldn't be doing that.
W: [pulseaudio] main.c: If you do it nonetheless then it's your own fault if things don't work as expected.
W: [pulseaudio] main.c: Please read http://pulseaudio.org/wiki/WhatIsWrongWithSystemMode for an explanation why system mode is usually a bad idea.
E: [pulseaudio] bluez4-util.c: org.bluez.Manager.GetProperties() failed: org.freedesktop.DBus.Error.UnknownMethod: Method "GetProperties" with signature "" on interface "org.bluez.Manager" doesn't exist
Nov 23 21:47:16 pi5 bluetoothd[650]: Endpoint registered: sender=:1.27 path=/MediaEndpoint/A2DPSource
Nov 23 21:47:16 pi5 bluetoothd[650]: Endpoint registered: sender=:1.27 path=/MediaEndpoint/A2DPSink
```

Hmm,... some sort of dbus error.  Great - another desktop-designed
tool needed to be run out of context for which there's bound to be no
documentation.

But wait - endpoint registered to `bluetoothd` - maybe:

```terminal
[bluetooth]# connect 00:11:67:01:06:C0
Attempting to connect to 00:11:67:01:06:C0
[CHG] Device 00:11:67:01:06:C0 Connected: yes
Connection successful
```

A bit of playing shows we have connection.  First off we can't talk to
pulseaudio with `Connection refused`.  So:

```terminal
$ sudo adduser pi pulse-access
```

And log back in again.

```terminal
$ pactl info
Server String: /var/run/pulse/native
Library Protocol Version: 29
Server Protocol Version: 29
Is Local: yes
Client Index: 1
Tile Size: 65496
User Name: pulse
Host Name: pi5
Server Name: pulseaudio
Server Version: 5.0
Default Sample Specification: s16le 2ch 44100Hz
Default Channel Map: front-left,front-right
Default Sink: bluez_sink.00_11_67_01_06_C0
Default Source: bluez_sink.00_11_67_01_06_C0.monitor
Cookie: 52ca:c414
```

And we need to keep calling `connect` in `bluetoothctl` every time it
disconnects.  You can bypass this with `trust`

```terminal
[bluetooth]# trust 00:11:67:01:06:C0
[CHG] Device 00:11:67:01:06:C0 Trusted: yes
Changing 00:11:67:01:06:C0 trust succeeded
[CHG] Device 00:11:67:01:06:C0 Connected: no
[CHG] Device 00:11:67:01:06:C0 Connected: yes
```

Now disconnects automatically reconnect.


Before I started writing these notes I wasted a few hours trying to
work out how to get dbus running in the context for which bluez would
be happy (and later how that would all work when I gave it to
systemd).  Thankfully I noticed the endpoint was already there while
re-writing these notes.

Next steps:

1. Note warning about `--disallow-module-loading` - adding this option
   in stops the setup working.  [Some notes here](https://forum.armbian.com/index.php/topic/2872-pulseaudio-status-in-armbian-h3/?p=19497)
2. Get it working on system start.
3. Work out how hotplugging/connection sensing works so we can monitor
   when to send data or not.
4. Work out if dbus error is causing any problem.
