---
layout: post
title: Raspberry Pi Steamlink fix
---

A reminder for myself on how to fix a bug, until I can figure out how to get
this fixed upstream.

### Description

I'm using RetroPie on a Raspberry Pi 4, and I use the steamlink package to run
games on my desktop machine but play them from my Pi (plugged into my TV). I
updated some packages and found that steam link just gives me a completely
blank black screen.


### Cause

Looking at contents of `/dev/shm/runcommand.log` after a few seconds of the
screen being blank, we can see the console output from steamlink:

```
Executing: steamlink
You are running with less than 128 MB video memory, you may need to go to the Raspberry Pi Configuration and increase your GPU memory.
Press enter to continue:
```

This is a pretty odd design choice in the linux world. There's no reason to
make the user press enter, and in our case it's the cause of our issues: how
are we supposed to press enter on RetroPie? It didn't even pop up a terminal!

Incidentally, it seems like this error message is a red herring on the Pi 4,
where
(GPU memory is allocated by the kernel)[https://retropie.org.uk/docs/Memory-Split/#raspberry-pi-4].

RetroPie triggers the script `/opt/retropie/ports/steamlink/steamlink_xinit.sh`
which in turn runs `/usr/bin/steamlink`, which itself runs
`/home/pi/.local/share/SteamLink/steamlink.sh`
(assuming RetroPie is running under the `pi` user).

Looking at `/home/pi/.local/share/SteamLink/steamlink.sh`, we can remove the
three offending calls to `read`, so that steamlink doesn't wait around forever
for an enter press.


```patch
--- steamlink.sh	2022-10-19 06:10:46.000000000 +1300
+++ /home/pi/.local/share/SteamLink/steamlink.sh	2024-02-24 11:24:01.795157816 +1300
@@ -25,9 +25,9 @@
 		tmpfile="$(mktemp || echo "/tmp/steam_message.txt")"
 		echo -e "$*" >"$tmpfile"
 		if [ "$DISPLAY" = "" ]; then
-			cat $tmpfile; echo -n 'Press enter to continue: '; read input
+			cat $tmpfile; echo -n 'Do not need to press enter to continue.'
 		else
-			xterm -T "$title" -e "cat $tmpfile; echo -n 'Press enter to continue: '; read input"
+			xterm -T "$title" -e "cat $tmpfile; echo -n 'Do not need to press enter to continue.'"
 		fi
 		rm -f "$tmpfile"
 	fi
@@ -98,7 +98,6 @@
 sudo cp -v $TOP/udev/modules-load.d/* /etc/modules-load.d/ && sudo modprobe uinput && sleep 3
 sudo cp -v $TOP/udev/rules.d/$UDEV_RULES_FILE $UDEV_RULES_DIR/$UDEV_RULES_FILE && sudo udevadm trigger && sudo usermod -a -G input,plugdev $(id -un)
-echo -n "Press return to continue: "
+echo -n "Do not need to press return to continue."
-read line
 __EOF__
 	if [ "$DISPLAY" = "" ]; then
 		/bin/sh $script
```
