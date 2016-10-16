---
layout: post
title: Shipping Out&#58; One Month Later
comments: true
---

![Entrance to the CERN Computer Centre](/resources/cern-computer-center.jpg)

This September, I started a one-year internship at CERN's Database
department (more specifically the IT-DB-IMS department, as they are
clearly very fond of hierarchies). My assignment is related to logging
and storage monitoring, and my primary task will be to provide a
solution for automatic propagation and reporting of storage-related
errors (think dead hard drives, power failures etc).

## Shipping Out

Early morning August the 27th, I started my journey. Split between a
suitcase and two backpacks, I brought the following with me on the plane:

- Soft-shell jacket and rain gear
- Clothes for at least ten days, vacuum packed (*)
- Micro-fibre towel
- Sleeping bag + ultra-light sleeping pad
- Various toiletries
- Computer with charger
- 6 USB chargers with cables
- Spare AA cells
- Bluetooth keyboard
- Bluetooth headphones
- Running gear for at least two weeks
- Swimming trunks and goggles
- 3 single-board computers (two CHIPs, one Raspberry Pi)
- Stuffed tiger, vacuum packed
- USB microphone
- A few USB thumb drives
- External disk drive (for back-ups)
- Various tools (including lock-picks, various special-purpose screwdrivers and a side cutter)
- Thermos mug
- Pyjamas
- Mountaineering/hiking boots

(*) this turned out to be largely false -- perhaps ten normal days, but
not ten _really hot_ days.

![View from my office](/resources/cern-computer-center-parking.jpg)

After an uneventful (and very WiFi-less) flight of about two hours, I
landed in Geneva, found my luggage after walking an impressive number of
hallways, each one plastered with advertisements for luxury watches,
various banking services, perfumes etc, and proceeded to board the bus
to Ferney-Voltaire, the small French village near the Swiss border where
I would be staying.

The process involved much frustration and about one hour of pacing the
arrivals hall trying to find someone who could help me buy a ticket from
the vending machine. It turned out the Right Thing To Do was to buy a
ticket in the machine for a different company than the one who operated
the bus I ended up taking. From there, the rest of the trip was _mostly_
uncomplicated, as I had practised walking most of the way from the bus
stop to the place I would be staying at on Google Maps in
advance. Something which, I suppose, is evidence of the same combination
of absolute neuroticism and ruthless pragmatism as vacuum-packing one's
stuffed animal for the journey.

During the first month of my stay, I was living without a mobile data
plan. The experience reminds me of Cory Doctorow's short story _After
the Siege_ (available online as part of
_[Overclocked](http://craphound.com/overclocked/download/)_). Everything
works mostly as normal (only worse) -- until I leave the
apartment. Then _everything_ immediately stops working. Accidentally
closed an email attachment when trying to read the email that contained
it? Sorry, it's now unavailable. The song I just added to my playlist?
Gone. Too bad the EU hasn't succeeded in
[outlawing roaming fees](https://en.wikipedia.org/wiki/European_Union_roaming_regulations)
yet.

## On the Betrayal of Things

![The pensioned synchrotron from the official CERN Tour](/resources/cern-synchrotron.jpg)

Living in another country for any longer duration is a lot like being at
the wrong end of a set of really evil unit tests. In life in general, as
in programming, at any given point in time one holds a lot of
preconceptions about how the world works -- unreflected habits and
assumptions. Living in another country immediately invalidates a
non-empty, randomly selected set of these assumptions. It is _precisely_
the experience that
[Foucault describes in the Preface to _Les mots de choses_](https://serendip.brynmawr.edu/sci_cult/evolit/s05/prefaceOrderFoucault.pdf#page=2)
of the "laugh that shatters"; at first one laughs at the three different
positions of dried beans in the supermarket, the consistent placement of
the potato crisps next to the hard liquor, or the three different,
non-consecutive candy aisles -- until one realises that categories such
as "baking supplies", "candy", etc are entirely arbitrary, and just took
on an air of ontological truths by virtue of being a widely accepted
convention. There are, after all, several ways to skin a cat.

## _Ils sont fous ces gaules_

The culture shock I experience with the French culture is admittedly a
very mild one, but there are definitely subtle but unsettling
differences. Milk typically comes high-temperature-pasteurised, which
gives it a different flavour than fresh milk. And _everything_ is
perfumed and/or tinted pink; toilet paper (!), trash bags --
everything. Loose-leaf tea is not available anywhere, except in gift tin
boxes without any option for refills. Muesli is hard to come by, and the
"healthy" granola/muesli mixes "without added sugar" contains little
chocolate drops. Not to mention the entire pastries-for-breakfast
thing. I'm still not over that.

If anything, French administrative culture is a confusing jumble of
obstinate strictness and unregulated chaos. Applying for a bank account?
Sign 200 papers and prove your residence with a phone or utilities bill,
which is apparently the standard way of doing it. Going for a swim?
Well, certain types of swimming trunks are forbidden but there is no
standardised vocabulary for describing swimming trunks so you just have
to figure out which ones are ok through trial and error. Also, everybody
showers in their bathing suits, but gender-separated. There is no
discernible structure or scheme to the organisation of the pools, from
what I have been able to tell. You just do your best not to collide
with anyone, which is easier said than done, as people enter and exit
the pool all the time, turn suddenly and changes swim-styles without
warning.

## How's Work?

![A fiber switch in the CERN Computer centre](/resources/cern-fiber-switch.jpg)

I have never seen anyone suffer so badly from
[NIH](https://en.wikipedia.org/wiki/Not_invented_here) syndrome as CERN
does. For almost every given task, they have their own, subtly different
solution from the industry standard; Mattermost in stead of Slack,
Terrible Exchange Web Mail That Probably Escaped From 2007 in stead of
Google Apps (the only instance where I wish they'd have had _more_ NIH
and just gone with their own solution), self-hosted GitLab in stead of
Github, and a customized variant of Dropbox called "CERNBox". Oh, and
they have their own GNU/Linux distros. Two of them. And they are both
RPM-based.

From an economical standpoint this makes sense. They are already running
operations on huge server farms. Adding a few extra services is probably
very cheap, both in terms of labour and other resources, especially
compared to paying for a service offered by someone else. And in the
long run, their involvement in several FLOSS replacements for common
industry applications is most likely greatly improving them. I just wish
they would have put more effort into the UX on some of these apps, so
that they didn't all suffer from flakiness and the general "a cheaper
copy of X" look and feel (well-known from Libre/OpenOffice, which
somehow manages to look _even worse_ than their proprietary counterparts
-- all things considered no small feat). Frankly, I like my FLOSS
implementations as I like my tech products in general -- either better
or at least as good as the things they are replacing, or nonexistent.

Other than that, the work is interesting, if a bit mundane. I learn a
lot every day, but it is really hard to muster any real enthusiasm for a
small python daemon with the task of basically carting data from one API
to another. The only real challenges I have found so far is to implement
a query language and to test things _really, really well_, about which I
might go into some detail in the future.
