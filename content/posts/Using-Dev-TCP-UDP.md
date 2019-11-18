---
title: "Using /dev/tcp and /dev/udp"
date: 2019-11-18T14:00:44-05:00
draft: false
---

I recently had to check if a port was open on a system that didn't have netcat installed, and which I lacked permissions to install any software.  I wound up finding out that you can directly call /dev/tcp (or /dev/udp) to open a connection to a specific port to check if it's open or not.  The syntax for this is: {{< highlight text >}} /dev/tcp/<host>/<port> {{< / highlight >}}
