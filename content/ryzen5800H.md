+++
title = "Modern CPUs and sleeping on Linux"
date = 2021-10-08
draft = false
type = "post"

[taxonomies]
tags = ["linux", "xiaomi", "systemd"]
+++

The ryzen 7 5800H CPU has some weird sleeping quirks on linux that I couldn't figure out. This [looooooooooooooooong thread](https://gitlab.freedesktop.org/drm/amd/-/issues/1230) explains the issue in great detail, but I couldn't understand a word of it. Through fiddling and mashing however I now have things working. I'm documenting my boot parameters and sleep configs here for any future lost souls. My laptop (the 2021 Xiaomi Notebook Pro Ryzen Edition) now correctly sleeps, hibernates and restores from those states just fine. Note this works on Kernel 5.14+
<!-- more -->

First follow [this guide](https://confluence.jaytaala.com/display/TKB/Use+a+swap+file+and+enable+hibernation+on+Arch+Linux+-+including+on+a+LUKS+root+partition) to setup a swap file to hibernate to. Next, configure `/etc/systemd/sleep.conf`, we want to hibernate after 20mins of sleeping, and hibernate to disk.
```
[Sleep]
HibernateDelaySec=1200
AllowHibernation=yes
HibernateMode=platform shutdown
HibernateState=disk
```

To configure sleep settings, we put them in `/etc/systemd/logind.conf` 
```
[Login]
HandleLidSwitch=suspend-then-hibernate
```

Finally, we need to setup our boot parameters, I have them in my grub config setting `GRUB_CMDLINE_LINUX_DEFAULT` to `quiet splash resume=UUID=PARTUUID_WITH_SWAPFILE resume_offset=SWAPFILEOFFSET amd_iommu=off mem_sleep_default=deep"`. The swapfile parameters are explained in the tutorial linked earlier. 

I hope this helps someone! I'm sure there are many working setups but this is good enough for me. I'm not too concerned with battery life, as long as the screen goes off and comes back on!

