# owrtflash-ng

## owrtflash-ng.sh

### What does it do?

`owrtflash-ng.sh` can be used to flash **OpenWrt** en mass on multiple
devices nearly *at once*, which are connected to a client _on the same_ switch.

The minimum config about a 'node' needs to reflect at least a `model` (which
kind of hardware it is) and an ethernet `macaddr`.
Dissent options can specified as well, like: `router_ip`, `firmware`, `user` and 
`password` and so on to have various ways of flashing a device. These settings
however define the **state** of adevice. Either when coming right out a box,
or specific configuration of the device (like default mangement interface 
addresses).

### Features

* flash via **http**-, **telnet**[1]- or **ssh**-protocol over a
**factory**-, **openwrt**-, or **custom**[2]-firmware
* network is configured with `iproute2` (welcome to the future!!)
* Disable `network-manger` (if you are using it on your ethernet card) with
 a commandline argument. You do not want to have `network-manager` running
 on `eth0`, while using that tool! #TODO:HOWTO
* modular function design which is hopefully reuseable by other tools, or
are at least easily enhanceable.
* node config is written in shell script, so they are just variables to set.
You need to write the model definition and the defaults of the "factory" state,
and the "failsafe" mode. "OpenWRT" has defaults, too. If a more specific name
of a model is the same as the defaults for a whole family, then just use 
symlinks, please. Due the nature of shell-syntax, you can use includes.

[1] Cause of a bug, I had to cheat a little bit, and just set the password
via `telnet` to enable `ssh`, and then just flash via `ssh`.

[2] By *custom* I mean of course a OpenWrt derivat.

### Example use-cases

```
# Unpack your device,
# Setup your node config file... and
- `./defaults/models`
- `./defaults/factory`
- `./defaults/failsafe`, if the device does support tftp-boot
# place i.e. a firmware-file in `./firmware-images/custom`
# `./firemware-images/openwrt` contains either stable release or
# something like this....

# power on te router for the first time, or go ahead to the next command...


time ./owrtflash-ng.sh --nodes 0142 --from factory --to openwrt --sudo

# Now flash your own firmware over the virgin OpenWrt
time ./owrtflash-ng.sh --nodes 0142 --from openwrt --to custom --sudo

# If you want to /reset/ the device
time ./owrtflash-ng.sh --nodes 0142 --from custom --to factory --sudo
```

### Dependencies

Please refer to `_check_requirements`. It is are not that much. Anyway,
 Bastian pointed out, if it should run under OpenWrt (or POSIX shell) as
 well, `seq` can not be used in the current form. So tests under OpenWrt
 have to be done

The script is developed and tested on Debian Testing (07/2015) and runned
 in `/bin/sh` aka `/bin/dash`.

### Limits

While testing I noticed certain bugs in certain situations, ...

* **There is nearly no way to ensure the flash progress worked!**
`_flash_*_via_http` like `_flash_*_via_ssh` give you no way to monitor the
 process! `ssh`s output is moved aside to not fuckup the screen output
 of the tool, and `sysupgrade` have to be executed with a `nohup`
 functunality, to detach from the `tty` and the remote shell. It is
 planned to have some function like a ping test at the end of the flash
 function, which checks and monitors pings to the node. If they are up,
 and then go down after ~30 seconds, and came back up after ~ 30-60 secounds later,
 it seams like that flashing was successful and the device had `reboot`ed.
 "Immerhin was..."
* After setting a password via `telnet`, `dropbear` takes it's time to restart.
 So `_set_password_via_telnet` which is a `expect` script, just restarts `dropbear`
 by itself. However we still have to wait...
* As a result of an unfound bug in my `expect` script `_flash_*_via_telnet`
 just uses `_flash_*_via_ssh` in the end, to simplify my life.
* Sometimes OpenWrt sucks, or my devices are overall a pit full of shit!
 They have problems after a power cycle, or... *hehe* this is funny: Once,
 after flashing OpenWrt over OpenWrt and rebooting, there where no files
 available under `'/'`.
 Yeah funny things like this, which the script can not determine, and
 will cause it to fail. This is robbing your nervs while testing.
 Most of the time there are no minor problems, but sometimes you just
 can't know why even...

Anyway, enough hate speech for now... now to something totaly different ...

### Example directory structures

For `owrtflash-ng.sh` to work properly, you need a bunch of config files.
The following is an **example**, which can be found in the `git` repository as well.
If you want to contribute additional config files for yet unsupported devices,
please get in touch with me and send me your code.

#### Default settings

A general overview:

```
./defaults/
├── factory
│   ├── tl
│   ├── tl-wr841
│   ├── tl-wr841n-v8
│   └── tl-wr841n-v9
├── generic
├── models
│   ├── tl-wr841
│   ├── tl-wr841n-v8 -> tl-wr841
│   ├── tl-wr841n-v9 -> tl-wr841
│   ├── tl-wr842nd
│   └── tl-wr842nd-v1 -> tl-wr842nd
└── openwrt
```

As you can see, there are default settings for...

* **generic**, where you will find boring stuff like the `'${user_agent}'` for `curl`,
* **models**, which specifys for now only the `${chipset}`,
* **factory**, which contains various configuration for the factory firmware state,
 like the `'${router_ip}'` and `'${client_ip}'`, `'${user}'` and `'${password}'`,
* **openwrt**, also gives you a `'${user}'` and `'${password}'`, as well
 as a `'${router_ip}'` and `'${client_ip}'`.

 These 4 config sections are loaded dynamicly and so they will overwrite
 some shell variables. Later you will see, that this also the way to
 overload functions like `_flash_over_factory_via_http`.


##### factory settings

Factory default config files contain i.e.

* `router_ip`
* `client_ip`
* `user`
* `password`
* `protocol`

If a model has some kind of defauls across different revisions or alike,
config files shoud be reused by sourcing them from one to another.

```
# cat defaults/factory/tl-wr841n-v8
#!/bin/sh

. "${__basedir}"/defaults/factory/tl-wr841


# cat defaults/factory/tl-wr841
#!/bin/sh

. "${__basedir}"/defaults/factory/tl


# cat defaults/factory/tl
#!/bin/sh
# TP-Link generic factory defaults

router_ip="192.168.0.1"
client_ip="192.168.0.100"

protocol="http"
user="admin"
password="admin"
```

##### Firmware repository

Then there is a archiv or repository for firmware image files.

```
./firmware-images/
├── custom
├── factory
│   ├── README
│   ├── tl-wr841n-v8
│   │   ├── download_factory.sh
│   │   ├── TL-WR841ND_V8_140724
│   │   │   ├── How to upgrade TP-LINK Wireless N Router&AP(192.168.0.1 version).pdf
│   │   │   ├── wr841nv8_en_3_15_9_up_boot(140724).bin
│   │   │   └── wr841nv8_en_3_15_9_up_boot(140724).bin.stripped
│   │   └── TL-WR841ND_V8_140724.zip
│   ├── tl-wr841n-v8.bin -> ./tl-wr841n-v8/TL-WR841ND_V8_140724/wr841nv8_en_3_15_9_up_boot(140724).bin
│   ├── tl-wr841n-v8.bin.stripped -> ./tl-wr841n-v8/TL-WR841ND_V8_140724/wr841nv8_en_3_15_9_up_boot(140724).bin.stripped
│   ├── tl-wr842nd-v1
│   │   ├── download_factory.sh
│   │   ├── TL-WR842ND_V1_130322.zip
│   │   └── wr842ndv1_en_3_12_25_up_boot(130322)
│   │       ├── wr842ndv1_en_3_12_25_up_boot(130322).bin
│   │       └── wr842ndv1_en_3_12_25_up_boot(130322).bin.stripped
│   ├── tl-wr842nd-v1.bin -> ./tl-wr842nd-v1/wr842ndv1_en_3_12_25_up_boot(130322)/wr842ndv1_en_3_12_25_up_boot(130322).bin
│   └── tl-wr842nd-v1.bin.stripped -> ./tl-wr842nd-v1/wr842ndv1_en_3_12_25_up_boot(130322)/wr842ndv1_en_3_12_25_up_boot(130322).bin.stripped
└── openwrt
    ├── openwrt-ar71xx-generic-tl-wr841n-v8-squashfs-factory.bin
    ├── openwrt-ar71xx-generic-tl-wr841n-v8-squashfs-sysupgrade.bin
    ├── tl-wr841n-v8-factory.bin -> openwrt-ar71xx-generic-tl-wr841n-v8-squashfs-factory.bin
    └── tl-wr841n-v8-sysupgrade.bin -> openwrt-ar71xx-generic-tl-wr841n-v8-squashfs-sysupgrade.bin
```

The idea is to have just **factory**, **openwrt**, and **custom** as a
directory structure. **factory** shoud be sorted by model, **openwrt**
by release (have to be done), and under **custom** you put your own builds.

##### Flash over factory helper scripts

Model files which define a `_flash_over_factory_via_http` function.

```
./flash-over-factory
├── tl-wr841n-v8.sh
└── tl-wr842n-v1.sh
```

##### Various helper scripts

Do do, like they say...

```
./helper_functions/
├── flash_over_openwrt_via_telnet.exp
├── nohup.sh
└── set_passwd_via_telnet.exp
```

#### Node configuration

Here you go, and state your `model` and `macaddr`.

```
./nodes
├── 001
├── 002
├── 003
├── 0041
└── 0142
```

```
$ cat nodes/0142
#!/bin/sh
# 0142

model="tl-wr841n-v8"
macaddr="a0:f3:c1:05:8a:c2"

#firmware="wr841nv8_en_3_15_9_up_boot(140724).bin"

#protocol="ssh"

#router_ip="192.168.1.1"
#client_ip="192.168.1.100"
#user="root"
#admin="admin"
```

With these information you should hopefully be enabled to use that tool.

```
time ./owrtflash-ng.sh --nodes 0142 --from openwrt --to custom --sudo
2015-07-31 00:11:56 [info]  Checking for `sudo`
[sudo] password for ed:
2015-07-31 00:12:00 [info]  Checking requirements passed.
2015-07-31 00:12:00 [info]  Checking for `sudo`
2015-07-31 00:12:00 [log]   Loop over nodes '0142'.
2015-07-31 00:12:00 [log]   Next device in list: '0142'.
2015-07-31 00:12:00 [info]  *** 0142: Load node config file.
2015-07-31 00:12:00 [info]  *** 0142: Load generic defaults.
2015-07-31 00:12:00 [info]  *** 0142: Load hardware defaults for 'tl-wr841n-v8'.
2015-07-31 00:12:00 [info]  *** 0142: Load OpenWrt defaults.
2015-07-31 00:12:00 [info]  Resetting network
2015-07-31 00:12:00 [info]  *** 0142: Setting client IP to 192.168.1.100.
2015-07-31 00:12:00 [info]  *** 0142: Setting arp table entry for 192.168.1.1 to a0:f3:c1:05:8a:c2.
2015-07-31 00:12:00 [info]  *** 0142: Testing network connection to 192.168.1.1 via arping.
2015-07-31 00:12:01 [info]  *** 0142: Network status: OK
2015-07-31 00:12:01 [info]  *** 0142: Load generic defaults.
2015-07-31 00:12:01 [info]  *** 0142: Load hardware defaults for 'tl-wr841n-v8'.
2015-07-31 00:12:01 [info]  *** 0142: Load OpenWrt defaults.
2015-07-31 00:12:01 [log]   *** 0142: Setting password via `telnet`.
2015-07-31 00:12:27 [info]  *** 0142: Checking `ssh` remote shell login (Try 1/5).
2015-07-31 00:12:31 [log]   *** 0142: Checking `ssh` passed.
2015-07-31 00:12:31 [log]   *** 0142: Trying to flash with '/home/ed/src/owrtconfig/firmware-images/openwrt/tl-wr841n-v8-sysupgrade.bin'...
2015-07-31 00:12:36 [info]  *** 0142: Checking `ssh` remote shell login (Try 1/5).
2015-07-31 00:12:38 [log]   *** 0142: Checking `ssh` passed.
2015-07-31 00:12:38 [info]  *** 0142: Installing `nohup.sh` to "0142"...
2015-07-31 00:12:41 [info]  *** 0142: Copying "/home/ed/src/owrtconfig/firmware-images/openwrt/tl-wr841n-v8-sysupgrade.bin" to "0142"...
2015-07-31 00:12:48 [log]   *** 0142: Starting `sysupgrade`...
2015-07-31 00:12:48 [info]  Resetting network
2015-07-31 00:12:48 [log]   Loop over nodes finished.
2015-07-31 00:12:48 [info]  Exit

real    0m51.598s
```
