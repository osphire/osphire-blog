+++
date = '2025-09-27T14:51:44+02:00'
draft = true
title = 'Hyprland and Nvidia Can Be Friends'
+++

# Hyprland and Nvidia can be friends
If you have done any search on the topic, chances are you have heard some snide remarks on Nvidia and Linux. It is notoriously finicky to get to play nice with your setup.
Now combine it with Hyprland, Wayland-enthousiasts current favourite child, and the rather common but surprisingly misunderstood dual-GPU setup most higher-end laptops come with, and you have a recipe for suffering.
The Hyprland wiki too famously waves away all mentions of using Nvidia hardware as being not supported.
Yet, all this does not discourage you. The prospect of a pretty little Hyprland setup entices you ever so slightly too much.
If this is you, I respect that.
You are up to it. You are courageous- you have style!
Before I turned into a bitter old lady trying to actually get this to work, I was like you.
It can't be that hard right?
In retrospect it was all pretty doable, but in the process I ended up wasting a lot of time scouring the internet and trying things out. Many to no avail.
But in the end I got it to work!
Stability is great, and it is very usable as a daily driver.
So here I am, ready to share my secrets with you, so you do not have to waste the days on this I had to.

# The system
This guide is for a computer running **two GPUs**:
- an integrated GPU: low power, built into your CPU
- a dedicated GPU (high power draw, but more raw power for more demanding graphical tasks. I will be referring to then using the abbreviations **iGPU** and **dGPU** respectively for brevity.
I assume you are running some sort of display manager, and you want to use Hyprland.
You can make this type of setup work for other graphical sessions, but some of the environment variables will have to be changed.

This guide is for **Arch Linux**, but you should be able to make it work on other distros as well. You'll just have to figure out some other package names, and replace `pacman` with your package manager of choice, and load kernel modules through a different method from `mkinitcpio`.



# Nvidia drivers
First, we have to get the Nvidia part of the system working.
We'll be using the proprietary Nvidia drivers.
If you prefer using the nouveau drivers- good on you! You probably know what substitutions to make in the rest of this guide.
We'll be using the DKMS versions of the drivers:
```bash
pacman -S --needed \
nvidia-dkms \ # Actual drivers
linux-headers \ # We need this if we use dkms version
nvidia-prime \ # optional: includes prime-run, which we will make our own version of later.
nvidia-utils \ # includes nvidia-smi
libva-nvidia-driver \ # Hardware acceleration for video
libva-utils \
egl-wayland 
```

Enable modeset by making the file `/etc/modprobe.d/nvidia.conf` with the contents 
```conf {filename="/etc/modprobe.d/nvidia.conf"}
options nvidia_drm modeset=1
```
This should no longer be necessary with newer cards, but adding it doesn't hurt.

Now edit `/etc/mkcpininitio.conf` and add to the `MODULES` in this order:
```conf {filename="/etc/mkcpininitio.conf"}
MODULES=(... i915 nvidia nvidia_modeset nvidia_uvm nvidia_drm ...)
```
If you already have have other kernel modules, you can just add append them to the ones listed, as long as their relative ordering is the same!
Don't forget to rebuild using `sudo mkinitcpio -P`.



# Tools of the trade
If you run into issues, or want to tweak things, here are a couple of good tools to use:
- `nvidia-smi` shows you which programs are running using which GPU. Note that if the dGPU is currently suspended, calling `nvidia-smi` **will wake up the dGPU**.
- We can check if the dGPU is currenty turned on by checking `cat /sys/bus/pci/devices/0000:01:00.0/power/runtime_status`. You might have to adjust the numbers based on where your dGPU is plugged in. You can figure this out using ` lspci -d ::03xx`. I recommend making a little script which calls this for you, as you'll probably end up using this a lot. This will say 'active' when the dGPU is turned on. Not just when it is actually being used by a program-- when it is turned on. You might have some nasty programs checking or probing it occasionally which keep it online, while `nvidia-smi` does not actually show any programs that use it. Keep this in mind during troubleshooting.

# The set-up we want



## Which gpu do we run Hyprland on?
Hyprland is pretty lightweight. If you can run it on the iGPU, that is ideal.
It is also completely possible to run a Hyprland itself on the iGPU, but a game or other graphics-intensive program on the dGPU. Arguably, it might even be beneficial if you are trying to squeeze out every last bit of performance :)
However, depending on your hardware, it might be desirable to run Hyprland on your dGPU as well. My machine, a Lenovo Legion laptop (as well as many more laptops much like it), has unfortunately hard-wired all of the external display connectors exclusively to the dGPU. 
This means that I need to run Hyprland on the dGPU to be able to use external monitors.
If I run it on the iGPU and connect an external monitor, it just stays black.
The [Hyprland wiki suggests just running Hyprland on the dGPU all the time](https://wiki.hypr.land/FAQ/#my-external-monitor-is-blank--doesnt-render--receives-no-signal-laptop), but I feel that this solution is rather extreme.
Why don't we just allow you to choose which device to run Hyprland on?
In a beautiful reality where Nvidia drivers just worked, unicorns shit rainbows, and essential workers are paid a living wage, we should just be able to switch it while we are running Hyprland.
In this gritty reality, we have to face the reality that this is not possible. Once a process starts on a certain GPU, **it is practically impossible to hot-potato Hyprland from one GPU do another**.

### Selecting the GPU to run Hyprland on at boot time.
The option that is left is that we have to decide it at boot time.
If you are a sane person, you are launching Hyprland through a Display manager.
Display manangers like `greetd` give you the option to choose a specific *session* to start. This is useful if you are using different graphical sessions (shout-out to my KDE plasma buddies!), but we can also use it to run Hyprland with some different options.
The USWM-managed version of Hyprland actually uses this same trick!
So, we will make two special Hyprland sessions:
One that runs it on the iGPU, and one that runs it on the dGPU.
Sessions are really just XDG desktop files, so they'll probably look familiar to you.
I create two very similar entries in `/usr/share/wayland-sessions/`:

```desktop {filename="/usr/share/wayland-sessions/hyprland-igpu.desktop"}
[Desktop Entry]
Name=Hyprland (igpu)
Comment=Hyprland running on the igpu cannot run multi-monitor.
Exec=/usr/local/bin/hyprland-igpu.sh
Type=Application
DesktopNames=Hyprland
Keywords=tiling;wayland;compositor;
```

```desktop {filename="/usr/share/wayland-sessions/hyprland-nvidia-gpu.desktop"}
[Desktop Entry]
Name=Hyprland (Nvidia)
Comment=Hyprland running on dgpu can display over multiple monitors.
Exec=/usr/local/bin/hyprland-nvidia-gpu.sh
Type=Application
DesktopNames=Hyprland
Keywords=tiling;wayland;compositor;
```

As you can see, what this session really does is launch a wrapper script (which we will define later). This is because some display managers (like `greetd`'s `tuigreet`) **do not like environment variables in their `Exec`**-- and we want to propagate settings not just to Hyprland, but also to any programs started in it!


# Getting everything to run on the iGPU
First, we need to be sure that we have rendering backends, configurations, and drivers for all the apps we need for both the iGPU and the dGPU.
If you are missing one, you might find yourself in a precarious situation where one set of programs incessantly uses the dGPU instead of the iGPU (I've been there), or vice versa.
If you are using the proprietary Nvidia drivers, you will probably already have everything set up for the Nvidia side. What remains is the Intel iGPU:
```bash
pacman -S
# Abstraction layer to allow vulkan to use mesa devices such as an igpu
vulkan-mesa-layers
# Hardware acceleration for video
intel-media-driver
# So we can check iGPU utilisation etc in btop or other dedicated programs
intel-gpu-tools
xf86-video-intel
vulkan-tools
mesa-utils
# for entries in /usr/share/vulkan/icd.d/
vulkan-intel
```

## Environment Variables
What we need to do now is set a whole load of environment variables to let programs know which devices and which library implementations to use.
- `__GLX_VENDOR_LIBRARY_NAME=mesa`: Hyprland uses OpenGL ES, meaning we should tell it to which library that should use. 

### Aquamarine device selection
Some special attention must be given to *Aquamarine*, Hyprland's lightweight rendering framework. We can set an environment variable `AQ_DRM_DEVICES` to say which device it must run on
(fun fact: DRM here does not stand for *Digital Rights Management*, as you might have encountered in video games, but for **Direct Rendering Manager**).
As all devices in Linux are also files, we can point to it as its path under `/dev/dri/`. If you look in this directory, you will find two inconspicuously named files for the different GPUs: `card1` and `card0`, maybe even a `card2`. But which one to use?
You can manually identify which one is which by cross-referencing the numbers in the output of `lspci -d ::03xx` and `ls -l /dev/dri/by-path/` 

```bash
$ ls -l /dev/dri/by-path/
---
lrwxrwxrwx - root 27  9月 08:34 pci-0000:00:02.0-card -> ../card1
lrwxrwxrwx - root 27  9月 08:34 pci-0000:00:02.0-render -> ../renderD128
lrwxrwxrwx - root 27  9月 08:34 pci-0000:01:00.0-card -> ../card0
lrwxrwxrwx - root 27  9月 08:34 pci-0000:01:00.0-render -> ../renderD129
```

```bash
$ lspci -d ::03xx
---
00:02.0 VGA compatible controller: Intel Corporation Raptor Lake-S UHD Graphics (rev 04)
01:00.0 VGA compatible controller: NVIDIA Corporation AD107M [GeForce RTX 4060 Max-Q / Mobile] (rev a1)
```

Now we see that `card1` is actually the iGPU, and `card0` is the dGPU!

Unfortunately, these number assignments do not always stay the same. While right now the iGPU can be `card1` and the dGPU `card0` on the next boot they might be swapped!
We need a way to consistently refer to the iGPU and the dGPU, which we can do using udev rules, as also suggested on [the Hyprland Wiki](https://wiki.hypr.land/Configuring/Multi-GPU/#creating-consistent-device-paths-for-specific-cards). 

We'll create a udev rule which automatically does the steps we did before, and creates a symlink to the correctly identified graphics device. We'll do one for both the iGPU and the dGPU, just for good measure.
Create `/etc/udev/rules.d/intel-igpu.rules` with the following content:
```udev {filename="/etc/udev/rules.d/intel-igpu.rules"}
KERNEL=="card*", \
  KERNELS=="0000:00:02.0", \
  SUBSYSTEM=="drm", \
  SUBSYSTEMS=="pci", \
  SYMLINK+="dri/intel-igpu"
```

Now for the dGPU, `/etc/udev/rules.d/nvidia-gpu.rules` with the following content:
```udev {filename="/etc/udev/rules.d/intel-igpu.rules"}
KERNEL=="card*", \
  KERNELS=="0000:01:00.0", \
  SUBSYSTEM=="drm", \
  SUBSYSTEMS=="pci", \
  SYMLINK+="dri/nvidia-gpu"
```

And now reload udev:

```bash
sudo udevadm control --reload
sudo udevadm trigger
```

You should now see two new symlinks in `/dev/dri`.
We can use these symlinks to set the aquamarine environment variable.
`AQ_DRM_DEVICES=/dev/dri/intel-igpu`

## Environment variables in sessions
So, where do you put these environment variables?
The title of this section kinda spoils the fun, but bear with me~
The simplest place would be to put them in the Hyprland config itself. Any settings here can be seen by Hyprland, and any programs started by it.
Our use case has one point of conflict for this solution though; we do not *always* want to run Hyprland on the iGPU.
But wait! Recall the different sessions we set up? We want to be able to run it on the dGPU as well to support multi-monitor. If we do so, then we also need some different environment variables. We can combine this by putting the environment variables in the wrapper scripts that we use to launch a session.
Because we want these variables to also be propagated into any programs launched under Hyprland, we want to `export` them.

```bash {filename="/usr/local/bin/hyprland-igpu.sh"}
#!/bin/bash
# Specifically run hyprland on the igpu.
# Also set up every other program to launch under the igpu by default,
# unless instructed otherwise.

# Which specific gpu device to run hyprland on.
# the device is a symlink to the igpu, set up from udev rules.
# (see the other modules in this git)
export AQ_DRM_DEVICES=/dev/dri/intel-igpu
# OpenGL: used by hyprland itself too!
export __GLX_VENDOR_LIBRARY_NAME=mesa
# libva driver is the intel VA-API.
export LIBVA_DRIVER_NAME=iHD
export VDPAU_DRIVER=va_gl

export KWIN_DRM_DEVICES=/dev/dri/intel-igpu

# Attempt at stopping apps from activating the dgpu for like 5 seconds
# from https://bbs.archlinux.org/viewtopic.php?pid=2098846#p2098846
# These must be overwritten when you try to run something on the nvidia gpu!
# These are for opengl backend
export __EGL_VENDOR_LIBRARY_FILENAMES=/usr/share/glvnd/egl_vendor.d/50_mesa.json
# For nvidia: /usr/share/glvnd/egl_vendor.d/10_nvidia.json
# and for vulkan
export VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/intel_icd.x86_64.json
# For nvidia: /usr/share/vulkan/icd.d/nvidia_icd.json

Hyprland
```

And with that, we should be finished!
Note that pc boots with gpu turned on- give it some time to suspend (~10s).

# But..! Now there is a pause when I launch some apps?
If you have followed this humble little guide so far, you will find that the dGPU is suspended by default, and all programs will run run on the iGPU.
We're not done quite yet though! You might notice a distinct pause when starting some programs, as well as your dGPU flaring up during this brief interval, albeit only momentarily before suspending again.
This happens when the dGPU is suspended, but launching the program causes it to be probed.
The problem now is that this happens not only when you want to explicitly run something on the dGPU, but also when you want to run something on the iGPU.
That's... Not ideal!
This problem exists mainly for OpenGL and Vulkan applications. Examples are games, but also every-day programs like Nautilus, 
When launching an application using them, a compatibility layer tries to find devices it could run on.
For Vulkan, the available configurations are placed in `/usr/share/vulkan/icd.d/`. You need ones for both your igpu and dgpu here! If you are missing the entries for Intel,install `vulkan-intel`.
The same goes for GL in `/usr/share/glvnd/egl_vendor.d/`.
Both backends iterate through all available options. Unfortunately the do not stop when they have found one, they iterate through all of them. And guess what- When probing the dGPU configuration, it has to wake up from sleep first, causing a delay!
There's a pretty easy workaround for this though. Using environment variables, we can specifically say which configurations it must consider.
For GL, we can simply supply `__EGL_VENDOR_LIBRARY_FILENAMES=/usr/share/glvnd/egl_vendor.d/50_mesa.json`, and for Vulkan `VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/intel_icd.x86_64.json`.
We have to set these globally, but beware! If we now simply `prime-run some-application`, it will not be able to find the configurations for the dGPU!

Now, `prime-run` is no magic program, unless you consider wrapper scripts wizardry-- in which case, understanding this blog post up to here makes you a wizard too! Looking at the source, we see that it really is just a [tiny wrapper with some environment variabkes](https://gitlab.archlinux.org/archlinux/packaging/packages/nvidia-prime/-/blob/main/prime-run?ref_type=heads).
```bash {filename="/usr/bin/prime-run"}
#!/bin/bash
__NV_PRIME_RENDER_OFFLOAD=1 \
__VK_LAYER_NV_optimus=NVIDIA_only \
__GLX_VENDOR_LIBRARY_NAME=nvidia \
"$@"
```

Mixing ingredients... It is more like alchemy!
So why don't we concoct our own little potion?
We just need a couple more environment variables, so that the globally exported configuration files are temporarily overruled by ones that refer to our Nvidia card.
Let's create a script in a folder that is added to our path (I have a `~/scripts/` folder that I add to my path in `.bashrc`, but putting it in `~/.local/bin/` also works fine), called `dgpu-run`.

```bash {filename="~/.local/bin/dgpu-run"}
#!/bin/bash
# By default everything is expected to run on the igpu.
# I have hard-coded some driver configs in the root env so that it doesnt
# constantly wake up the dgpu. This means we have to specifically tell that there are actually nvidia implementations for vulkan and opengl available.
__EGL_VENDOR_LIBRARY_FILENAMES=/usr/share/glvnd/egl_vendor.d/10_nvidia.json \
VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/nvidia_icd.json \
__NV_PRIME_RENDER_OFFLOAD=1 \
__VK_LAYER_NV_optimus=NVIDIA_only \
__GLX_VENDOR_LIBRARY_NAME=nvidia \
"$@"
```
Now, we can request any program to start on the dGPU exactly the same way you would interact with `prime-run`!



TODO: I still have this problem for electron apps like Vesktop!




# Troubleshooting

## Caveat: dGPU auto-suspend and external monitors
Plugging in an external monitor  kept the gpu awake at all time, even if the monitor is turned off or nothing is displayed!
This holds no matter how you connect it-
through a usb-c hub (Usb-c hub works fine if no hdmi cable is plugged into it), or direct HDMI cable.
Although again- this is probably a limitation of my make of laptop, where the Nvidia card is hard-wired to the external ports.
There is one more downside- if you have woken up the dgpu by pluggin in a monitor, it will not go to sleep again on that boot- even of you unplug the monitor. You will need to reboot to restore suspending functionality.


## TLP - a one-stop shop for battery saving... Right?
TLP was constantly checking on my Nvidia card. That kept it awake.
I have seen it go to `suspended`, but rarely, and inconsistently.
Then I found [this forum post](https://forums.developer.nvidia.com/t/ryzen-5500u-and-runtime-d3-issue/287383/14) linked from [here](https://bbs.archlinux.org/viewtopic.php?id=294113&p=2).
Uninstalling it made sure that finally my card was suspended!
Also noticeable: suddenly pc was very quiet.
And somehow, cpu temperature was still like 20C at rest?
In short: **avoid using tlp in hybrid iGPU-dGPU set-ups, as it stops the graphics card from suspending**

