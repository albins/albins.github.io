---
layout: post
title: Installing Steam in ElementaryOS
---
1. Install the .deb from steam: `sudo dpkg -i steam_latest.deb`.
2. Fix the proken dependencies (because Steam does not just use the default terminal emulator...): `sudo apt-get -f install` (it will pull in xterm).

To run Steam in Freya on a 64-bit machine, you need a 32-bit environment. However, as ElementaryOS is based on Ubuntu LTS, you can't install the "normal" libGL/Mesa libraries as Steam tries to do. In stead, you need the LTS versions. So before running Steam, do `sudo aptitude install libgl1-mesa-dri-lts-trusty:i386 libgl1-mesa-glx-lts-trusty:i386 libc6:i386`.

*Now* you can start Steam!
