---
title: "Grub BadRAM"
date: 2019-11-05T09:37:07-05:00
draft: false
---

I learned about a neat little snippet today that you can use to mask out bad RAM
segments within GRUB.  This can be declared in /etc/default/grub to prevent
usage of ram segments that you know are dodgy, or that you've identified to be
bad through other means.

The option is:
{{< highlight text >}}
    GRUB_BADRAM="<start_address>,<mask>"
{{< / highlight >}}

After editing your config, simply run grub-mkconfig and reboot.
