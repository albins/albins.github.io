---
layout: post
title: Shoebox Time Machine backup server with Raspberry Pi ("poor man's Time Capsule")
---

![IKEA Kassett box with HDD sticking out, labelled "DATA"](/resources/data_shoebox.jpg)

It is possible and fairly easy to build an acceptable Time Machine backup server using a Raspberry Pi and a spare USB drive. Basically, the process is as follows:

1. Install dependencies
2. Format and set up the drive
3. Download and compile netatalk
4. Configure netatalk

## 1. Install dependencies

Just do
```
sudo apt-get install build-essential libevent-dev libssl-dev libgcrypt-dev libkrb5-dev libpam0g-dev libwrap0-dev libdb-dev libtdb-dev libmysqlclient-dev avahi-daemon libavahi-client-dev libacl1-dev libldap2-dev libcrack2-dev systemtap-sdt-dev libdbus-1-dev libdbus-glib-1-dev libglib2.0-dev libio-socket-inet6-perl tracker libtracker-sparql-0.16-dev libtracker-miner-0.16-dev libgrypt20-dev
```

## 2. Format and set up the drive

Some guides call for using a hfsplus-formatted drive. If you have one that you previously used with macOS, by any means -- use it. If not, I would prefer a more well-adapted GNU/Linux file system. I would recommend ext4, which is what most distros use, unless you have any particular reason to use something else.

First use `sudo fdisk /dev/sda` to partition your hard drive (this is assuming that the hard drive is the only USB harddisk or pendrive attached to your Raspberry Pi -- if it is not, make sure you pick the right one!).

1. Press `o` to create a new partition table.
2. Create a new partition using the `n` command. Choose primary (`p`) and use the suggested sizes (unless you have reason to change them). Choose the `Linux filesystem`, which should be the default.
3. When you are back at the main menu, press `w` to write everything to disk and quit.

Now, format the drive as ext4: `sudo mkfs.ext4 -L "Time Machine" /dev/sda1`. When you are done, determine the unique identifier (UUID) of the new partition with `blkid /dev/sda1` and copy the entire `UUID="LOTS OF TEXT"` part.

Create a mount point at `/media/time_machine` with `sudo mkdir /media/time_machine`.

Edit `/etc/fstab` and add a line like this:
`UUID="ca92ada0-e97a-44fc-badb-f0e934d775c9" /media/time_machine ext4 defaults,rw,user,auto 0 0`

Mount it using `sudo mount -a` and try writing anything to `/media/time_machine` (e.g. `touch /media/time_machine/test.file`). If you get permissions errors, do `sudo chown -R user /media/time_machine`, where `user` is your user name (again, proably pi).

## 3. Download and compile netatalk

Netatalk is a free implementation of Apples network/file sharing protocols. It is available from the official repositories, but that version is _ancient_, so we need to compie it ourselves.

First, download it. You can either do it on your local machine and copy it to your raspberry pi using `scp`, or you can find a link on the download page and use `wget <file link>` to download it directly. After you have it, just extract it with the usual `tar xvf netatalk-3.1.9.tar.bz2` (assuming you have the same version). 

You can follow [the official guide](http://netatalk.sourceforge.net/wiki/index.php/Install_Netatalk_3.1.9_on_Debian_8_Jessie), but basically it is a matter of running `configure` with the right flags. 

First, see which version of tracker you got from apt-get: `pkg-config --list-all | grep tracker` and insert it after --with-tracker-pkgconfig-version in the proposed command. In my case, I got 1.0:

```sh
./configure         --with-init-style=debian-systemd         --without-libevent        \     --without-tdb         --with-cracklib         --enable-krbV-uam        \ --with-pam-confdir=/etc/pam.d         --with-dbus-daemon=/usr/bin/dbus-daemon\         --with-dbus-sysconf-dir=/etc/dbus-1/system.d         --with-tracker-pkgconfig-version=1.0
```

Verify that DHX2 is enabled (you need it for password authentication on recent versions of macOS, at least El Capitan):

```
UAMS:
         DHX     (PAM SHADOW)
         DHX2    (PAM SHADOW)
```

Then `make && sudo make install` it!

## 4. Configure netatalk

After that, update your `/usr/local/etc/afp.conf`:

```ini
[Global]
; Global server settings
vol preset = default_for_all_vol
log file = /var/log/netatalk.log
uam list = uams_dhx.so,uams_dhx2.so
mimic model = TimeCapsule6,106

[default_for_all_vol]
file perm = 0664
directory perm = 0774
cnid scheme = dbd
valid users = albin

[Time Machine]
path = /media/time_machine
time machine = yes
```

Substitute your own user name for `albin` at `valid users`, or omit the line entirely to allow all users. Remember, this is a user account _on the server_. If you are using a Raspberry Pi, this is probably the "pi" user and "raspberry" password (but you really should create a user of your own _and_ change the password of the default account!). `mimic model` doesn't really do much, except giving your Raspberry Pi a nice Time Cube icon in Finder.

Start up netatalk:

```sh
sudo service netatalk start
```

Keep track of the logs using `sudo tail -f /var/log/netatalk.log`, and try to browse and log in from your computer. The Raspberry Pi should appear in Finder, and you should be able to log in as your chosen user (e.g. pi).

When you are confident everything works, you can set up netatalk to start when the server does:

```
sudo systemctl enable netatalk
```

After this, you should be able to find and select your shoebox server as a target in Time Machine. Now, go and make a _very big_ cup of tea as your computer runs its initial backup.

![Time Machine: Backup Completed](/resources/Time_machine_backup_completed.png)

## Sources
- [https://gist.github.com/oscarcck/3135109](https://gist.github.com/oscarcck/3135109)
- [http://www.techradar.com/how-to/computing/how-to-make-a-mac-time-capsule-with-the-raspberry-pi-1319989](http://www.techradar.com/how-to/computing/how-to-make-a-mac-time-capsule-with-the-raspberry-pi-1319989) (but this guide is _obviously_ misinformed!)
