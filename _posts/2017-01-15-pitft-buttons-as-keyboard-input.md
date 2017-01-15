---
layout: post
title:  "PiTFT buttons as keyboard input with gpio-keys"
date:   2017-01-15 23:13:14 +1100
categories: linux raspberry-pi
tags: raspberry-pi linux adafruit kernel
---

The first stop I found in getting PiTFT buttons as input into the pi is [gpio_keys_device](https://github.com/notro/fbtft_tools/tree/master/gpio_keys_device), a nice little kernel module which allows gpio buttons to be mapped to key codes using module options.  For example:

``` terminal
sudo insmod gpio_keys_device.ko keys=22:108,27:105,17:106,23:103  pullup=1 active_low=1 repeat=1
```
The kepresses are somewhat unreliable - `dmesg` tells us:

``` terminal
gpio_keys_device: Pull up/down not supported on this platform
input: gpio-keys as /devices/platform/gpio-keys.0/input/input1
```

That's a pain - but looking in the source suggests it does support the raspberry pi - specifically for pull up/down.

Poking around back on Adafruit I can find their key-press handler [for powering down only](https://learn.adafruit.com/adafruit-2-8-pitft-capacitive-touch/extras).  This works perfectly so it seems like I'm missing something.  Luckily (there are only three raspberry pi sites for any project on the entire internet in a sea of unrelated misinformation) it turns out the `fbtft_tools` project itself housed the original version of `rpi_power_switch` and it already has a patch for the Pi2 and Pi3.  Reapplying that and everything looks nice:

``` terminal
$ evtest
No device specified, trying to scan all of /dev/input/event*
Not running as root, no devices may be available.
Available devices:
/dev/input/event0:	ft6x06_ts
/dev/input/event1:	gpio-keys
Select the device event number [0-1]: 1
Input driver version is 1.0.1
Input device ID: bus 0x19 vendor 0x1 product 0x1 version 0x100
Input device name: "gpio-keys"
Supported events:
  Event type 0 (EV_SYN)
  Event type 1 (EV_KEY)
    Event code 103 (KEY_UP)
    Event code 105 (KEY_LEFT)
    Event code 106 (KEY_RIGHT)
    Event code 108 (KEY_DOWN)
Properties:
Testing ... (interrupt to exit)
Event: time 1484484129.283515, type 1 (EV_KEY), code 106 (KEY_RIGHT), value 1
Event: time 1484484129.283515, -------------- EV_SYN ------------
Event: time 1484484129.423485, type 1 (EV_KEY), code 106 (KEY_RIGHT), value 0
Event: time 1484484129.423485, -------------- EV_SYN ------------
Event: time 1484484129.923524, type 1 (EV_KEY), code 108 (KEY_DOWN), value 1
Event: time 1484484129.923524, -------------- EV_SYN ------------
Event: time 1484484130.043503, type 1 (EV_KEY), code 108 (KEY_DOWN), value 0
Event: time 1484484130.043503, -------------- EV_SYN ------------
Event: time 1484484130.923509, type 1 (EV_KEY), code 103 (KEY_UP), value 1
Event: time 1484484130.923509, -------------- EV_SYN ------------
Event: time 1484484131.083501, type 1 (EV_KEY), code 103 (KEY_UP), value 0
Event: time 1484484131.083501, -------------- EV_SYN ------------
Event: time 1484484131.733511, type 1 (EV_KEY), code 105 (KEY_LEFT), value 1
Event: time 1484484131.733511, -------------- EV_SYN ------------
Event: time 1484484131.913505, type 1 (EV_KEY), code 105 (KEY_LEFT), value 0
Event: time 1484484131.913505, -------------- EV_SYN ------------
```

So being a good internet citizen I post back a pull-request for the Pi2/3 patch, but get the somewhat of-putting response:

> This will stop working when we switch to Linux 4.9 in a month or so. Only CONFIG_ARCH_BCM2835 will be defined for Pi1, 2 and 3. Which one is determined at runtime.
> 
> I suggest you use a device tree overlay instead.

The details are that the `gpio_keys_device` is simply configuration boilerplate for the already extant `gpio_keys` kernel module, which should nowadays be configured via dts.

So I get to learn a whole other language - Device Tree Overlay.  Now I've been peering over that for some work-related issues, the [kernel](https://www.kernel.org/doc/Documentation/devicetree/overlay-notes.txt) [documentation](https://www.kernel.org/doc/Documentation/devicetree/dynamic-resolution-notes.txt) are 100% true with absolutely no applicability.

According to my Google-Fu DT Overlay documentation on the internet only ever applies to either Raspberry Pi or BeagleBone.  Most of the documentation defines *how you could* use DT overlay, not **how** you **could** use it. Thankfully for this one we're on the pi so we have a chance here - and this should clarify a bunch of things on the work side.

First hurdle is very easy the [kernel documentation for gpio_keys](https://www.kernel.org/doc/Documentation/devicetree/bindings/input/gpio-keys.txt) is very good, for what it does.  The problem is there is no documentation on how to achieve what I had with the module approach `pullup=1 active_low=1`.  But first things first lets code up an overlay and apply it.

Second hurdle: familiarise self with `dtoverlay` and the `dtc` "device tree compiler".  Sadly:


``` terminal
-bash: dtoverlay: command not found
```

This took a bit of searching since `rpi-update` handles this for you - as I discussed [here](/linux/raspberry-pi/2017/01/10/pi-kernel-source-with-adafruit-tft.html) - Adafruit doesn't play ball with `rpi-update`.  TL;DR the package needed is `libraspberrypi-bin`, which Adafruit also repackage.  `device-tree-compiler` is a standard Debian package.

I'll skip the details here, since someone with DTS experience on raspberry-pi probably got this easily.  Needless to say going round the horn a number of times I eventually came up with the following:

``` dts
/dts-v1/;
/plugin/;

/ {
	compatible = "brcm,bcm2835", "brcm,bcm2708", "brcm,bcm2709";

	fragment@0 {
		target = <&gpio>;
		__overlay__ {
			pitft_buttons_pins: pitft_buttons_pins {
				brcm,pins = <27 22 23 17>; /* gpio pins */
				brcm,function = <0 0 0 0>; /* boot up direction:in=0 out=1 */
				brcm,pull = <2 2 2 2>; /* pull direction: none=0, 1=down, 2=up */
			};
		};
	};

	fragment@1 {
		target-path = "/soc";
		__overlay__ {
			pitft_buttons: pitft_buttons {
				compatible = "gpio-keys";
				#address-cells = <1>;
				#size-cells = <0>;

				pinctrl-names = "default";
				pinctrl-0 = <&pitft_buttons_pins>; // note must dtc compile with -@ to resolve this

				button@17 {
					label = "BTN1";
					linux,code = <87>; //F11
					gpios = <&gpio 17 1>; // 1 is "active low"
					debounce-interval = <5>; // default 5ms
				};
				button@22 {
					label = "BTN2";
					linux,code = <88>; //F12
					gpios = <&gpio 22 1>; // 1 is "active low"
				};
				button@23 {
					label = "BTN3";
					linux,code = <183>; //F13
					gpios = <&gpio 23 1>; // 1 is "active low"
				};
				button@27 {
					label = "BTN4";
					linux,code = <184>; //F14
					gpios = <&gpio 27 1>; // 1 is "active low"
				};
			};
		};
	};

};
```

The two parts are as follows:

`fragment@0` adds a gpio configuration entry with the name `pitft_buttons_pins` which describes 4 pins in input mode with pullups enabled.  This takes absolutely no effect unless a separate driver makes use of it.

`fragment@1` is the `gpio-keys` entry that should be familiar from all of the documentation around the internet on on `gpio-keys`.  The key *(sic)* part is the `pinctrl-0`, `pinctrl-names` entries that define an indexed list of configurations and refer back to the entry added in `fragment@0`.  Once you've seen and read this I hope most things are clear.

*Important note* that wasn't obvious from any other documentation - the last parmeter in the `gpios = <&gpio 17 1>;` stanza is the invert sense - ie `0` means active high, `1` means active low.  I haven't seen this documented anywhere else.

Here's my gotcha though (**don't copy below it is wrong**):

``` terminal
$ dtc -o pitft_buttons.dtbo pitft_buttons.dts
Warning (unit_address_vs_reg): Node /fragment@0 has a unit name, but no reg property
Warning (unit_address_vs_reg): Node /fragment@1 has a unit name, but no reg property
Warning (unit_address_vs_reg): Node /fragment@1/__overlay__/pitft_buttons/button@17 has a unit name, but no reg property
Warning (unit_address_vs_reg): Node /fragment@1/__overlay__/pitft_buttons/button@22 has a unit name, but no reg property
Warning (unit_address_vs_reg): Node /fragment@1/__overlay__/pitft_buttons/button@23 has a unit name, but no reg property
Warning (unit_address_vs_reg): Node /fragment@1/__overlay__/pitft_buttons/button@27 has a unit name, but no reg property
$ sudo dtoverlay -v -d $(pwd) pitft_buttons
run_cmd: which dtoverlay-pre >/dev/null 2>&1 && dtoverlay-pre
DTOVERLAY[debug]: Wrote 1287 bytes to '/tmp/.dtoverlays/0_gpio_keys.dtbo'
DTOVERLAY[debug]: Wrote 1287 bytes to '/sys/kernel/config/device-tree/overlays/0_gpio_keys/dtbo'
run_cmd: which dtoverlay-post >/dev/null 2>&1 && dtoverlay-post
```

But evtest says we don't have a device and `dmesg` says:

``` terminal
gpio-keys soc:pitft_buttons: could not find pctldev for node /soc/interrupt-controller@7e00b200, deferring probe
```

The error is fairly clear and simple - the `pinctl` line is broken - so the `gpio-key` component won't be initialised.  Commenting this line out:

``` dts
pinctrl-0 = <&pitft_buttons_pins>; // note must dtc compile with -@ to resolve this
```

Makes it go away but also breaks gpio pin configuration - pins remain in state they were in.  This is subtle if you've already set them up from previous runs..

Finally found the solution which I hinted above.  `dtc` **REQUIRES** the `-@` parameter to allow any symbol resolution to occur.  I would never have guessed that local symbols (ie within the same compiler run) would not always be cross-referenced:

``` terminal
$ dtc -@ -c pitft_buttons.dtbo pitft_buttons.dts
$ dmesg | tail -1
$ sudo dtoverlay -v -d $(pwd) pitft_buttons
input: soc:pitft_buttons as /devices/platform/soc/soc:pitft_buttons/input/input5
```

Success!

You can now move that `.dtbo` file into `/boot/overlays` and add `dtoverlay=pitft_buttons` if you want it to stick permanently - or add the `dtoverlay` to your application startup scripts.  Perhaps I'll learn about `dtparams` one day and parameterise it since the gpio lines, pullups and active-low are fixed on the board, you'll only need to map the key-codes and optionally the debounce.

## Credit where its due

The [full text](https://github.com/notro/fbtft_tools/pull/11#issuecomment-271711477) of the message explaining that the `gpio_keys_device` module fix wouldn't apply actually had all of the information I needed in its content + the subsidiary links.  And if I'd read the email in full, rather than in the "digest" form seen in the email notification on my phone, and thus seen there was sample code and useful links - I could have had about 5 hours of my life back.

Other sources that helped:

* [pinctrl](https://www.kernel.org/doc/Documentation/devicetree/bindings/pinctrl/pinctrl-bindings.txt)
* [Broadcom GPIO driver pinctrl](https://www.kernel.org/doc/Documentation/devicetree/bindings/pinctrl/brcm,bcm2835-gpio.txt)
* [rpi forum post](https://www.raspberrypi.org/forums/viewtopic.php?f=107&t=115394)
* [Armadeus wiki](http://www.armadeus.org/wiki/index.php?title=GPIO_keys)
