---
layout: post
title:  "Hello ThinkPad, Goodbye Macbook Pro"
date:   2018-07-29 20:00:00 +09:00
categories:
- Linux
---


_NOTE: I only write about a laptop for development use. My casual use machine is still mac ;)_


Goodbye Macbook Pro 2016
----

I like Apple products, for example Mac mini, Macbook Air, Macbook Pro 2013, iPhone, iPad, Apple TV, Express, etc. These Apple products make me happy! But, I couldn't make myself like the Macbook Pro 2016.

One day, I finally understand the reason why I hate new MBP. I hate butterfly keyboard. I always enjoy key typing. But, the Butterfly keyboard is tooo bad for me. I can't enjoy happy hacking...

And, I don't need Touch Bar (I like Touch ID). I decide to stop buying Apple's high-end laptop for a few years.


Hello ThinkPad T480s
----

I think it is a time to use Linux desktop. I'm interesting to server-side and systems programming. I don't need beautiful graphical environment.

- High-end laptop
- Good value for money (CPU, RAM, SSD, ...)
- Non-glare display (11 - 14 inch)
- Keystroke (No too deep, No too shallow)
- US keyboard
- Lightweight (nearly 1 kg)
- TrackPoint
- Linux friendly hardware


How do I feel Ubuntu on ThinkPad T480s
----

### Pros

- I can enjoy typing, it is goood keystroke!
- I can concentrate, thanks to non-glare display!
- The Black housing is so beautiful!! (like G3)
- It was less light than expected.
- It easy to install Ubuntu.

### Cons

- [Can't use WWAN device in Ubuntu.](https://forums.lenovo.com/t5/Linux-Discussion/Linux-support-for-WWAN-LTE-L850-GL-on-T580-T480/m-p/4067969)
- Can't use Fingerprint reader in Ubuntu.

### WIP

- CPU Performance and TDP.


Conclusion
----

I'm enjoyng! Goodbye MBP!!


MEMO
----

### Install via USB

I wrote installer image to USB drive using MBP.

```
$ hdiutil convert -format UDRW \
  -o ~/Downloads/ubuntu-18.04-desktop-amd64 \
  ~/Downloads/ubuntu-18.04-desktop-amd64.iso
$ mv ~/Downloads/ubuntu-18.04-desktop-amd64.img ~/Downloads/ubuntu-18.04-desktop-amd64.dmg
$ diskutil list
...
/dev/disk2 (external, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:     FDisk_partition_scheme                        *7.9 GB     disk2
   1:                 DOS_FAT_32 USB3_NST1               7.9 GB     disk2s1

$ diskutil unMountDisk /dev/disk2
$ sudo dd if=$HOME/Downloads/ubuntu-18.04-desktop-amd64.dmg of=/dev/rdisk2 bs=1m
1832+1 records in
1832+1 records out
1921843200 bytes transferred in 89.132149 secs (21561729 bytes/sec)
```

### Basic key config

I'm an Emacs user.

```
$ gsettings set org.gnome.desktop.input-sources xkb-options "['ctrl:nocaps']"
$ gsettings set org.gnome.desktop.interface gtk-key-theme Emacs
```