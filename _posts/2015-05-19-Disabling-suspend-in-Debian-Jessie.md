---
layout: post
title: Disabling suspend on lid close in Debian 8.0 Jessie
---
As Jessie now uses systemd by default, the old trick of editing `/etc/default/acpi-support` doesn't work. In stead, you have to edit `/etc/systemd/logind.conf` and add the line `HandleLidSwitch=ignore`.

In my case, suspend was really slow to kick in, and I have no idea why. In other words, in case you have a newly installed laptop server that dies a while after having its lid closed, this is probably your problem. Also, mine typically crashed on resume, because, hey, it's 2015 and suspend/resume is apparently still hard.
