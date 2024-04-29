# Install debian on android guide

> [!IMPORTANT]
> **Root** required, if not consider termux. And this is a **VERY** dangerous process. I'm not responsible for any bootlooped or bricked phone. Do this with your broken phone or something

## Install lhroot

- Install [busybox](https://github.com/Magisk-Modules-Alt-Repo/BuiltIn-BusyBox)

- Install [lhroot](https://github.com/FerryAr/lhroot).

- Install debian as in lhroot repo README.


### Troubleshooting

Well, lhroot is not well maintained, so you can have very mystery bug: "Module not detected". The fix depends on device, but take a look at [my comment](https://github.com/FerryAr/lhroot/issues/45#issuecomment-1946347252)


## Initial setup

Chrooted in debian (using `bootlinux` in `lhroot`)

Setup network

> [!TIP]
> `aid_net_raw`, `aid_net` is 2 groups have network permission, with you got any network, socket issue, try to add your user to these groups.

```
groupadd -g 3004 aid_net_raw
groupadd -g 1003 aid_graphics
usermod -g 3003 -G 3003,3004 -a _apt
usermod -G 3003 -a root
```

Update debian mirror

> [!TIP]
> There are many guide online for this part, and more updated than this

Look for the nearest mirror based on your country at https://www.debian.org/mirror/list, mine is vietnam


```
# path: /etc/apt/sources.list

deb http://mirrors.bkns.vn/debian bullseye main
```

Update system
```
apt update -y && apt upgrade -y
```

Fix the debconf errors:

```
apt install -y dialog apt-utils
```

### Create normal user

```
groupadd storage
groupadd wheel
useradd -m -g users -G wheel,audio,video,storage,aid_inet -s /bin/bash <your username>
passwd <your username>
```

## Setup SSH and other services


Install [openrc](https://github.com/OpenRC/openrc)

> Why? Android using `init` init system, `systemd` does not work with that.

```
apt install openrc

# Remove default config, because this is a chrooted system, they don't work

rm -f /etc/rc*.d/*
rm -f /etc/runlevels/*/*

rc-update # <- Should have no result
```

Install openssh-server

```
apt install openssh-server
```

Re-check

```
rc-update # <- Should have ssh as default
```

Execute:

```
/sbin/openrc default
```

and sshd should be started normally at port 22.

Read openrc [user-guide](https://github.com/OpenRC/openrc/blob/master/user-guide.md) for more.



## Persist between reboot

> Required adb rooted shell. You may `adb shell` then type `su`

To mount our chroot and execute openrc on boot, we need to write a [magisk bootscript](https://topjohnwu.github.io/Magisk/guides.html#boot-scripts). If you use KernelSU, find your own ways

### Modify lhroot

Copy lhroot source code to somewhere (using adb push from pc), mine is `/data/scripts/`
```
busybox pwd
/data/scripts/

busybox tree
.
└── lhroot
    ├── bootlinux
    ├── bootlinux_env
    ├── bootlinux_init
    ├── bootlinux_log
    ├── customize.sh
    ├── killlinux
    ├── lhroot
    ├── make_image
    ├── mod-util.sh
    └── mount_image
```

Create a `autoboot` file in lhroot folder with following content

```
#!/system/bin/sh

######### IMPORT BOOTLINUX ENVIRONMENT #########
SCRIPT_PATH=$(readlink -f $0)
. ${SCRIPT_PATH%/*}/bootlinux_env

$busybox chroot $mnt /usr/bin/env su -c 'rm -rf /run/openrc; /sbin/openrc default;'
```

Make it executable

```
busybox chmox +x /data/scripts/lhroot/autoboot
```

### Create Boot script

Create a script at `/data/adb/service.d/` for example `/data/adb/service.d/my_startup.sh`

```
#!/system/bin/sh

/data/scripts/lhroot/autoboot
```

Make it executeable

```
busybox chmod +x /data/adb/service.d/my_startup.sh
```

Reboot

```
adb reboot
```

## Connect to ssh from pc

```
ssh <user>@<ip>
```

You can get `ip` from `ifconfig` command execute in adb (rooted) shell. `user` is user you have created.

## mDNS and avoid ip change.

> Following commands is executed in chrooted shell 

### Access to router

If you have access to router, try use a static ip and dynamic DNS.

### No access to router

Install avahi-daemon

```
apt install avahi-daemon
```

Add `avahi` user to network group

```
usermod -a -G aid_net_raw,aid_net
```

Config avahi, add service your shelf, there's debian docs for [that](https://wiki.gentoo.org/wiki/Avahi).

Results:

```
ssh ppvan-android@android.local
```


## Resources and useful links

https://wiki.gentoo.org/wiki/Avahi

https://github.com/meefik/linuxdeploy/issues/1092#issuecomment-532027612

https://github.com/FerryAr/lhroot/issues/14#issuecomment-1774036577

https://ivonblog.com/en-us/posts/termux-chroot-ubuntu/#4-create-a-regular-user-and-setup-language

https://www.reddit.com/r/termux/comments/15voay6/start_services_for_proot_installations_with/

https://topjohnwu.github.io/Magisk/guides.html#boot-scripts

https://github.com/FerryAr/lhroot/pull/29