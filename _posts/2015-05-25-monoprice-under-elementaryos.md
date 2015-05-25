---
layout: post
title: Monoprice tablets and multiple screens under ElementaryOS (and other GNU/Linuxes)
---
I recently got a 10 x 6.25-inch Graphic Monoprice drawing tablet via Massdrop. It turns out that these tablets are actually re-branded Huion devices, supported by recent kernels and by [the Digimend driver](http://digimend.github.io/). Mine is a [Huion H610](https://digimend.github.io/tablets/Huion_H610/). ElementaryOS Freya's kernel is apparently patched to contain the necessary drivers, and everything worked out of the box in terms of reading input.

However, the coordinates on the tablet are mapped to X11 coordinates. Since I have two monitors, this means that every coordinate on the tablet is mapped against a corresponding coordinate on the combined surface of both my screens, which makes navigation extremely awkward. This can be solved by modifying [the coordinate transformation matrix](http://linuxwacom.sourceforge.net/wiki/index.php/Dual_and_Multi-Monitor_Set_Up#Coordinate_Transformation_Matrix). The easiest way to do this is to use [the neat tool xrestrict](https://github.com/ademan/xrestrict), which simply prompts you to click on the desired screen and sets up the coordinate transformation matrix to restrict inputs to that screen for the tool you just clicked with. It is currently not available in the official repos, but building and running it was fairly straightforward.










