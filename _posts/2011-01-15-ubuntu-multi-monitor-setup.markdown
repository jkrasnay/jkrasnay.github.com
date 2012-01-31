---
layout: post
title: Ubuntu Multi-monitor Setup
---

My current computing platform is a ThinkPad T61p running Ubuntu. I
recently picked up a Mini Dock and a pair of 24" Samsung monitors. This
post describes how I got the multi-monitor setup working.

The ThinkPad has an NVidia Quadro FX 570M GPU.  Your best reference for
configuring dual displays on Linux with NVidia GPUs is the [NVidia Linux driver
README](http://us.download.nvidia.com/XFree86/Linux-x86_64/260.19.29/README/index.html)

There are a number of ways to get multiple monitors working on an X
server. I decided to use NVidia's proprietary TwinView approach. Under
TwinView, X thinks there's one large display space, and the NVidia
driver divides this up into multiple monitors.

The easiest way to get this working is to install the Ubuntu
`nvidia-settings` package and fiddle with the GUI application it
installs under the *System > Administration* menu. The problem is that
there doesn't seem to be a way to clearly switch between two modes when
I'm docked (the two external displays) or un-docked (the laptop's
panel). Luckily, like most things in Linux, we can fix things up with a
bit of scripting.

## Building nv-control-dpy

The `nv-control-dpy` utility gives us the functionality we need to
dynamically control our display configuration. This utility is actually
just a sample program that NVidia distributes with the driver source to
illustrate how to use their API. We can build `nv-control-dpy` from the
the Ubuntu source package. The following instructions are based on a [a
post to the Ubuntu forums
site](http://ubuntuforums.org/showthread.php?p=5809670), but have been
tweaked to work on Ubuntu 10.10.

{% highlight bash %}
sudo apt-get install build-essential libxext-dev libx11-dev
cd
mkdir nvidia-source
cd nvidia-source/
apt-get source nvidia-settings
cd nvidia-settings-*/
cd src/libXNVCtrl
make
cd ../../samples
make
{% endhighlight %}

At this point I got a compile error when compiling
`nv-control-events.c`. 

    nv-control-events.c:705: error: ‘NV_CTRL_FLATPANEL_DITHERING_MODE’ undeclared here (not in a function)

I didn't need this particular program; however, running `make
nv-control-dpy` as suggested in the Ubuntu forums post didn't work
either. The solution was to edit `nv-control-events.c` and comment out
the offending line:

    //MAKE_ENTRY(NV_CTRL_FLATPANEL_DITHERING_MODE),

Run `make` again, and you should find `nv-control-dpy` in the
`_out/Linux_x86_64` directory. I suggest you copy it `/usr/local/bin`.


## Switching Display Modes

X detects the installed displays when it starts. This can be a challenge for
laptop users whose display configuration changes depending on whether
or not they're docked. The way I've handled this is to remove all
TwinView-related settings from `/etc/X11/xorg.conf` (`TwinView`,
`MetaModes`, etc.), and to set things up dynamically at runtime.

Suppose we've rebooted our laptop un-docked. X will be unaware of our
external displays. We can remedy this by running `nv-control-dpy
--probe-dpys`. On my system the output is as follows:

    Using NV-CONTROL extension 1.24 on :0.0
    Connected Display Devices:
      DFP-0 (0x00010000): LEN

    Display Device Probed Information:

      number of GPUs: 1
      display devices on GPU-0 (Quadro FX 570M):
        CRT-0 (0x00000001): Samsung SyncMaster
        DFP-0 (0x00010000): LEN
        DFP-1 (0x00020000): Samsung SyncMaster

To the NVidia driver, `DFP-0` refers to the laptops attached panel,
while `CRT-0` is the external monitor connected to the dock's VGA port
and `DFP-1` is the external monitor connected to the dock's DVI port.

Now that the driver sees the new displays, we have to "associate" them
(I'm not exactly sure what that means, but we have to do it for things
to work.) If you look at the output of `nv-control-dpy --probe-dpys`
you'll notice that each display has an associated hexadecimal number. To
associated multiple displays, you have to call `nv-control-dpy
--set-associated-dpys X`, where `X` is the result of or-ing these
numbers together. On my system the proper command is `nv-control-dpy
--set-associated-dpys 0x00030001`.

Now we must create a "meta mode" that describes a configuration that
incorporates the external displays. In my case, I want them displayed
side-by-side, with the internal laptop display disabled. Here is the
appropriate setting:

    nv-control-dpy --add-metamode "DFP-0: NULL, CRT-0: nvidia-auto-select +0+0, DFP-1: nvidia-auto-select +1920+0"

Refer to the [NVidia
documentation](http://us.download.nvidia.com/XFree86/Linux-x86_64/260.19.29/README/index.html)
for a full treatment of meta modes.

Finally, we use the `xrandr` utility to switch between configurations.
`xrandr` is not directly aware of meta modes, but we can switch based on
the fact that one of the configurations has a display resolution of
1920x1200 and the other a resolution of 3840x1200:

    xrandr -s 3840x1200


## Tying It All Together

I put the required commnds together in a shell script that toggles the
display configuration:

{% highlight bash %}
#!/bin/sh
nv-control-dpy --probe-dpys
nv-control-dpy --set-associated-dpys 0x00030001
nv-control-dpy --add-metamode "DFP-0: NULL, CRT-0: nvidia-auto-select +0+0, DFP-1: nvidia-auto-select +1920+0"

if xrandr | grep 'current 1920'; then
    xrandr -s 3840x1200
else
    xrandr -s 1920x1200
fi
{% endhighlight %}

In Gnome, you can bind this to a keystroke by running *System >
Preferences > Keyboard Shortcuts*. Click *Add*, and specify the shell
script with a name and the full path to the script, and click *Apply*.
Finally, bind the new shortcut to a key. (I used `Ctrl-Alt-```.)

