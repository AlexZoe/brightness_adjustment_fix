# Brightness Adjustment Fix for Linux (Mint)

## Description
I am currently running Linux Mint 19.2 (Tina) on a Lenovo Thinkpad with a Ryzen 5 CPU and AMD Radeon Vega GPU.
This particular system configuration seems to have issues with power management settings such as adjusting brightness or preventing turning off the display when idle, where all settings are ignored by the system.

There seem to be some fixes available online which mostly involve changing GRUB settings which appear to work for some users.
Since the solution presented here is more of a quick fix/work around, you may want to try these first.
However, neither of the proposed solutions has worked in my case, even after upgrading the kernel from the default 4.15 to 5.3.

Since I did not want to sink any more time in this issue I decided to adjust the brightness directly using the '/sys/class' interface in combination with a simple udev rule.
This allows to adjust the brightness of the display, however, it does *NOT* prevent the system from turning off the display if idle for too long (which is not important to me).

## Applying Fix

### Check System

First get the device responsible for adjusting the brightness which is located at '/sys/class/backlight'

```
ls /sys/class/backlight/
```

On my system there are two entries 'amdgpu\_bl0' and 'thinkpad\_screen'.
Both directories have an entry 'brightness' and 'max\_brightness'.
Calling `cat` on these files will give you their current values.  
For example:

```
:~$ cat /sys/class/backlight/amdgpu_bl0/max_brightness

255
```

The value of brightness can be changed by echoing a value to the 'brightness' file.
You might have to be root to be allowed to change the value (e.g. sudo su).  
For example:

```
:~$ echo 30 > /sys/class/backlight/amdgpu_bl0/brightness
```

On my system only the 'amdgpu\_bl0' seems to work whereas the entries at 'thinkpad\_screen\' result in-/output errors.

### Change Device Permission

In order to change the brightness as non-root user, permissions of the device files have to be changed.
A new group 'display' is created and all users who should have access to the brightness control are added to this group.
The following example creates the group 'display' and adds the current user to it.
Add more users as you see fit.

```
:~$ sudo groupadd display
:~$ usermod -a -G display $(whoami)
```

Reboot for the group setting to take effect.

### Prepare Bash Script

Adjust the 'BACKLIGHT\_LOC' path in the 'brightness\_control' bash script contained in this repository to point to the device used for adjusting brightness on your machine (see [Check System](#check-system)).
Next, change the 'MIN' value to be less than what 'max\_brightness' returned.
'RATE' may be changed to adjust the speed at which the brightness should be adjusted.

Check if you can increase the brightness with the following command:

```
:~$ bash brightness_control -up
```

and lower it with:

```
:~$ bash brightness_control
```

Copy 'brightness\_control' to your preferred location (e.g. /usr/local/bin)


Note:
If you have insufficient permissions to execute the script, try using the script as root user (e.g. sudo su).
Alternatively change the group of the 'brightness' file and change permissions, assuming the current user is already a member of the group (see also [Change Device Permission](#change-device-permission)).  
For example:

```
:~$ chgrp display /sys/class/backlight/amdgpu_bl0/brightness
:~$ chmod g+w /sys/class/backlight/amdgpu_bl0/brightness
```

### Prepare UDEV Rule

Since the '/sys/class' devices are created during boot up, group and file permissions are not persistent.
The '99-backlight.rules' udev rule of this repository automatically sets the correct group and file permissions when the device for controlling the backlight is added to the file system.

Confirm that the 'SUBSYSTEM' in '99-backlight.rules' matches the one of the display device.
The subsystem of the device can be queried with the 'udevadm' command.  
For example:

```
:~$ udevadm info --path=/sys/class/backlight/amdgpu_bl0

P: /devices/pci0000:00/0000:00:08.1/0000:05:00.0/backlight/amdgpu_bl0
E: DEVPATH=/devices/pci0000:00/0000:00:08.1/0000:05:00.0/backlight/amdgpu_bl0
E: ID_PATH=pci-0000:05:00.0
E: ID_PATH_TAG=pci-0000_05_00_0
E: SUBSYSTEM=backlight
E: SYSTEMD_WANTS=systemd-backlight@backlight:amdgpu_bl0.service
E: TAGS=:systemd:
E: USEC_INITIALIZED=3677164
```

Adjust the path of the display device in the 'PROGRAM' section of the '99-backlight.rules' udev script if necessary.

Copy '99-backlight.rules' to '/etc/udev/rules.d'.

Reboot in order for the udev rule to take effect.

### Add Keybindings

In order to adjust the brightness using your keyboard, add keybindings to 'Control Center -> Keyboard Shortcuts'.  
For example:

```
Name:		Brightness down
Command:	/bin/bash /usr/local/bin/brightness_control
Shortcut:	Ctrl + F5
```
```
Name:		Brightness up
Command:	/bin/bash /usr/local/bin/brightness_control -up
Shortcut:	Ctrl + F6
```
