---
layout: post
title:  "Raspberry Pi Bluetooth Audio - startup and hotplugging"
date:   2016-12-01 2016 20:01:12 +1100
categories: linux raspberry-pi
tags: raspberry-pi bluetooth linux
---

In a previous post
[here](/linux/raspberry-pi/2016/11/23/pi-bluetooth-audio.html) I
started trying to get audio connection over bluetooth on a pi 3.

```terminal
$ sudo pulseaudio --system --disallow-exit --no-cpu-limit --high-priority --realtime
```

## Robustness

Experimenting with gst123 I can constrain the output port to the specific bluetooth device by name:

``` terminal
$ gst123 -Z -a pulse=bluez_sink.FC_58_FA_D0_07_77 /path/to/files
```

This gives a nice clear failure in case the bluetooth device is not
connected.  Making it easy to work out whether data is flowing
properly in the system.

However disconnecting the device whilst playing gives a less
satisfactory response.  Firstly `gst123` stalls.  Then after some time
it continues counting time and cycling through music. with no sound
output - nor any failure.  Obviously I can poll for the disappearance
of the device with `pactl` the default sink reverts back to analog.
This would be ok, except a temporary drop connects almost
immediately.  Killing `gst123` and starting again picks up straight
away, but we still get the actively cycling with no output bit.

Perhaps I'll be making a fully fledged dbus client yet `:-(`.

Looks like there's a workaround though:

``` terminal
$ pactl list short
...
22	module-bluez5-device	path=/org/bluez/hci0/dev_FC_58_FA_D0_07_6B	
...
```

shows for each connect a new handle for each reconnect (22 in this case).

So I should be able to poll for a connection to disappear and
reappear.  Sadly the related sink doesn't always disappear.

Also in many cases pulesaudio crashes during the disconnect.

```
D: [pulseaudio] core-subscribe.c: Dropped redundant event due to change event.
I: [pulseaudio] module-rescue-streams.c: Successfully moved sink input 1 "gst123" to alsa_output.0.analog-stereo.
D: [pulseaudio] module-rescue-streams.c: No source outputs to move away.
D: [pulseaudio] core-subscribe.c: Dropped redundant event due to remove event.
D: [bluetooth] module-bluez5-device.c: IO thread shutting down
D: [pulseaudio] module-bluez5-device.c: Switching the profile to off due to IO thread failure.
[Thread 0x7167e3f0 (LWP 3467) exited]
D: [pulseaudio] module-bluez5-device.c: Releasing transport /org/bluez/hci0/dev_FC_58_FA_D0_07_6B/fd14
I: [pulseaudio] bluez5-util.c: Transport /org/bluez/hci0/dev_FC_58_FA_D0_07_6B/fd14 auto-released by BlueZ or already released
D: [pulseaudio] module-bluez5-device.c: Audio stream torn down
I: [pulseaudio] sink.c: Freeing sink 1 "bluez_sink.FC_58_FA_D0_07_6B"
I: [pulseaudio] source.c: Freeing source 1 "bluez_sink.FC_58_FA_D0_07_6B.monitor"
I: [pulseaudio] card.c: Changed profile of card 1 "bluez_card.FC_58_FA_D0_07_6B" to off
E: [pulseaudio] asyncmsgq.c: Assertion 'pa_atomic_load(&(a)->_ref) > 0' failed at pulsecore/asyncmsgq.c:176, function pa_asyncmsgq_get(). Aborting.

Program received signal SIGABRT, Aborted.
0x76bddf70 in __GI_raise (sig=sig@entry=6)
    at ../nptl/sysdeps/unix/sysv/linux/raise.c:56
56	../nptl/sysdeps/unix/sysv/linux/raise.c: No such file or directory.
(gdb) bt
#0  0x76bddf70 in __GI_raise (sig=sig@entry=6)
    at ../nptl/sysdeps/unix/sysv/linux/raise.c:56
#1  0x76bdf324 in __GI_abort () at abort.c:89
#2  0x76f131fc in pa_asyncmsgq_get (a=0x76cee094 <lock>, a@entry=0x42ed8, 
    object=object@entry=0x7efff104, code=code@entry=0x7efff108, 
    userdata=userdata@entry=0x7efff10c, offset=offset@entry=0x7efff110, 
    chunk=chunk@entry=0x7efff118, wait_op=wait_op@entry=false)
    at pulsecore/asyncmsgq.c:177
#3  0x76f13af4 in pa_asyncmsgq_flush (a=0x42ed8, run=run@entry=true)
    at pulsecore/asyncmsgq.c:338
#4  0x76f77720 in pa_thread_mq_done (q=q@entry=0x4024c)
    at pulsecore/thread-mq.c:154
#5  0x7169c710 in stop_thread (u=u@entry=0x40210)
    at modules/bluetooth/module-bluez5-device.c:1304
#6  0x7169d2cc in set_profile_cb (c=<optimized out>, new_profile=0x56f60)
    at modules/bluetooth/module-bluez5-device.c:1574
#7  0x76f259e0 in pa_card_set_profile (c=c@entry=0x49e00, profile=0x56f60, 
    save=save@entry=false) at pulsecore/card.c:281
#8  0x71699b9c in transport_state_changed_cb (y=<optimized out>, t=0x54ae0, 
    u=0x40210) at modules/bluetooth/module-bluez5-device.c:1760
#9  0x76f2a92c in pa_hook_fire (hook=0x5ab98, data=data@entry=0x54ae0)
    at pulsecore/hook-list.c:106
#10 0x716e4458 in transport_state_changed (t=0x54ae0, 
    state=PA_BLUETOOTH_TRANSPORT_STATE_DISCONNECTED)
    at modules/bluetooth/bluez5-util.c:186
#11 0x716e7de4 in endpoint_clear_configuration (Cannot access memory at address 0x1
conn=<optimized out>, 
    userdata=0x5ab78, m=0x5f1c0) at modules/bluetooth/bluez5-util.c:1415
#12 endpoint_handler (c=<optimized out>, m=0x5f1c0, userdata=0x5ab78)
    at modules/bluetooth/bluez5-util.c:1464
#13 0x76df1bd4 in ?? () from /lib/arm-linux-gnueabihf/libdbus-1.so.3
```

Candidate patch (here)[https://bugs.freedesktop.org/attachment.cgi?id=114580] seems to work.

## Starting as a service

Taken from (here)[https://possiblelossofprecision.net/?p=1956]

```ini
[Unit]
Description=PulseAudio Daemon
 
[Install]
WantedBy=multi-user.target
 
[Service]
Type=simple
PrivateTmp=true
ExecStart=/usr/bin/pulseaudio --system --realtime --disallow-exit --no-cpu-limit --high-priority
```

Next steps:

1. Note warning about `--disallow-module-loading` - adding this option
   in stops the setup working.  [Some notes here](https://forum.armbian.com/index.php/topic/2872-pulseaudio-status-in-armbian-h3/?p=19497)
2. Work out if dbus error is causing any problem.
