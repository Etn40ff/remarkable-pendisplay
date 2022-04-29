This repository is meant to be a guide to use a Remarkable2 tablet as a pen
display for linux hosts. It is mainly a description of the process I went
through to make my setup work in my use case. Several things could have been
done differently. Suggestions on how to make this simpler to deploy and more
user friendly are welcome.

The same process should work also for Remarkable1 with few
minor modifications (e.g. to the device paths and screen sizes).

The secret ingredient here is `uinput` so this is not going to work as is for
osx nor windows. 

**This repository is currently a work in progress**

# Requirements

- a Remarkable2 tablet, I will refer to this as rm2
- a linux machine with an Intel video card, I will refer to this as host

# Steps

- Install `toltec` on rm2

- Install `netevent` and `vnsee` on rm2 through toltec

- Install `sshd` on host

- Setup key-based ssh access on both devices: user@host should be able to log in
  as root@rm2 and vice versa without typing passwords

- Make sure hosts runs a kernel that supports `uinput`; if it is a module make
  sure it gets loaded. Grant user sufficient rights to manage `uinput` devices.
  On my gentoo machine this amounted to adding user to the `input` group and
  loading the following udev rule
  ```
  KERNEL=="uinput", GROUP="input", MODE="0660"
  ```

- Add the following to the host's `hwdb` and refresh the database
  ```
  evdev:input:b0018v056Ap0000*
   LIBINPUT_DEVICE_GROUP=remarkable2
   EVDEV_ABS_00=::101
   EVDEV_ABS_01=::101
   ID_INPUT_TABLET=1
  
  evdev:input:b0000v0000p0000e0000-e0*
   LIBINPUT_DEVICE_GROUP=remarkable2
   EVDEV_ABS_00=::7
   EVDEV_ABS_01=::12
   EVDEV_ABS_35=::7
   EVDEV_ABS_36=::12
   LIBINPUT_CALIBRATION_MATRIX=0 1 0 1 0 0
   ID_INPUT_KEY=0
   ID_INPUT_TOUCHSCREEN=1
  ```

- Configure the host's X11 to use `libinput` drivers adding the following in
  `/etx/X11/xorg.conf.d/` then reload X11
  ```
  Section "InputClass"
  	Identifier      "Remarkable2 Touchscreen"
  	MatchProduct    "remarkable2-touchscreen"
  	Driver          "libinput"
  EndSection
  
  Section "InputClass"
  	Identifier      "Remarkable2 Digitizer"
  	MatchProduct    "remarkable2-digitizer"
  	Driver          "libinput"
  EndSection
  ```

- Add a new mode to the host's X11 and enable it on `VIRTUAL1`:
  ```
  xrandr --newmode 1872x1404 0 1872 1872 1872 1872 1404 1404 1404 1404 
  xrandr --addmode VIRTUAL1 1872x1404
  ```

- Create a file called `netevent.ne2` on rm2 containing these commands with the correct user@host
  ```
  device add digitizer /dev/input/by-path/platform-30a20000.i2c-event-mouse
  device add touchscreen /dev/input/by-path/platform-30a40000.i2c-event
  device rename digitizer remarkable2-digitizer
  device rename touchscreen remarkable2-touchscreen
  output add remote exec:ssh user@host netevent create
  use remote
  write-events on
  ```

All pieces of the configuration should now be in place. To start the system run
the following (These commands assume that the main output is `eDP1` and its
width is 1920, adapt them to your needs):
```
host$ xrandr --output VIRTUAL1 --right-of eDP1 --auto 
host$ x11vnc -repeat -forever -nocursor -nopw  -rotate 270 -clip 1872x1404+1920+0 -pipeinput UINPUT
rm2$ systemctl stop xochitl
rm2$ netevent daemon -s netevent.ne2 /var/run/netevent.sock &
rm2$ vnsee host
host$ xinput float "x11vnc injector"
```
The last command is there just to remove duplicates events while still retaining
the possibility to do fast refreshes when writing.

# TODO

- Refactor this into a single script to be run from host.

- It should be possible to use udev rules rather than hwdb; unfortunately
  libinput complains about missing capabilities. My udev foo is not strong
  enough here
 
- It should be possible to match both devices using the name we gave them
  `remarkable2-*` rather than vendor and product hex codes. This would be
  preferable since the touchscreen reports these numbers all to be zeros.
  Again my udev foo was not strong enough.
