## Nvidia on Hyprland
There is no _official_ Hyprland support for Nvidia hardware. However, you might make it work properly following this page. 

## Foreword

{{< hint >}}Hardware Accelerated Rendering{{</ hint >}}
If your issue is with a program running _on_ Hyprland, check if it's possible to avoid relying on your GPU. 
Your mileage will vary, but this can be a useful application-specific workaround that allows you to avoid more invasive changes.

e.g. Discord: ![discord-hardware-acceleration](https://github.com/hyprwm/hyprland-wiki/assets/788471/ac1398fe-00ab-4b52-9c04-1189400fb0c5)
Firefox: ![firefox-hardware-acceleration](https://github.com/hyprwm/hyprland-wiki/assets/788471/9f69eda1-eb8e-41aa-a0d0-a463975441ab)

In Discord, this eliminated symptoms such as:
  1. Window content flickering intermittently or disappearing.
  2. Unusual text cursor behaviour, characters disappearing and reappearing as you're writing them.
{{< hint >}}

{{< hint >}}Optimus multi-GPU support
- To get multi monitor support on laptops with both integrated and "dedicated" GPU's, you'll need to remove the `optimus-manager` package if installed. Disabling the service does not work. You also need to change your BIOS settings from Hybrid graphics to Discrete graphics.{{</ hint >}}

## How to (possibly) get Hyprland to work on Nvidia
Below are some tips to try to help Nvidia systems run Hyprland properly:
1. Confirm that the kernel and bootloader are configured for Kernel Mode Setting.
2. Add Hyprland hints and "glue code" to improve compatibility.
3. Pick the best driver for your situation and install it.
4. Rebuild the kernel, boot and test.
5. Work around any remaining bugs.
   
(This order is intended to avoid building a kernel a moment too soon. Bonus points if you saw this coming!)

### Check bootloader and kernel support
- For people using [systemd-boot](https://wiki.archlinux.org/title/systemd-boot), add `nvidia_drm.modeset=1` to the end of the `options ...` line in `/boot/loader/entries/arch.conf`.
- For people using [grub](https://wiki.archlinux.org/title/GRUB), add `nvidia_drm.modeset=1` to the end of `GRUB_CMDLINE_LINUX_DEFAULT=` in `/etc/default/grub`, then run `# grub-mkconfig -o /boot/grub/grub.cfg`
- For others check out [kernel parameters](https://wiki.archlinux.org/title/Kernel_parameters) for how to add `nvidia_drm.modeset=1` to your specific bootloader.

Next, set up your kernel:
- In `/etc/mkinitcpio.conf` add `nvidia nvidia_modeset nvidia_uvm nvidia_drm` to your `MODULES` array.
- Create `/etc/modprobe.d/nvidia.conf` (if it does not exist). Add the line `options nvidia-drm modeset=1`.

### Configure environment hints and software to support hyprland
Add the below variable exports in your hyprland.config:
```sh
env = LIBVA_DRIVER_NAME,nvidia
env = XDG_SESSION_TYPE,wayland
env = GBM_BACKEND,nvidia-drm
env = __GLX_VENDOR_LIBRARY_NAME,nvidia
env = WLR_NO_HARDWARE_CURSORS,1
```

Install:
 - `qt5-wayland`, `qt5ct`, `qt6-wayland`, `qt6ct` provide QT APIs and a way to specify configuration for that version.
 -  `libva` is the Video Acceleration API for Linux
 -  `libva-nvidia-driver-git` (AUR) fixes crashes in some Electron-based applications, such as Unity Hub.

### Pick a driver 
You can choose between the proprietary [Nvidia drivers](https://wiki.archlinux.org/title/NVIDIA) or the open source [Nouveau driver](https://wiki.archlinux.org/title/Nouveau). Under the proprietary Nvidia drivers category, there are 3 of them: the current driver named 'nvidia' (or 'nvidia-dkms' to use with custom linux kernels) which is under active development, the legacy drivers 'nvidia-3xxxx' for older cards which Nvidia no longer actively supports, and the 'nvidia-open' driver which is currently an alpha stage attempt to open source a part of their close source driver for newer cards.

You may want to use the proprietary Nvidia drivers in some cases, for example: if you have a newer laptop or GPU, if you want more performance, if you want to play video games, if you need a wider feature set (for example, better power consumption on recent GPUs), etc. However, keep in mind that if the proprietary Nvidia drivers do not work properly on your computer, the Nouveau or nvidia-open driver might work fine while not having as much features or performance. For [older cards](https://wiki.archlinux.org/title/NVIDIA#Unsupported_drivers), in order to use Hyprland, you will probably need to use the Nouveau driver which actively supports them.

{{< hint type=important >}}Suspend functions are (as of 2024-01-28) broken on `nvidia-open-dkms` [due to a bug](https://github.com/NVIDIA/open-gpu-kernel-modules/issues/472), so make sure you're on `nvidia` or `nvidia-dkms` for the time being.{{< /hint >}}

Otherwise, what to consider?
- If your GPU is listed as supported by the `nvidia-open` or `Nouveau` driver and you're using a mainstream `linux` kernel, try using those first.
- If you're using e.g `linux-zen`, or some other custom kernel, remember that you'll probably need a [DKMS](https://askubuntu.com/q/408605) variant of your chosen driver.
- If performance or the latest features are critically important, you may be better served by the `nvidia` or `nvidia-dkms` drivers.

### Rebuild and reboot
Finally, rebuild the kernel and initramfs, and (if necessary) create a bootloader entry to load it:
- `# mkinitcpio --config /etc/mkinitcpio.conf --generate /boot/initramfs-custom.img`
- or `mkinitcpio -P linux` (if you're feeling particularly brave)

Reboot your computer. When you restart, Kernel Mode Setting should be active.
Launch Hyprland. 

It _should_ work now.

{{< hint >}}If you encounter crashes in Firefox, try disabling `env = GBM_BACKEND,nvidia-drm`.{{< /hint >}}
{{< hint >}}If screen sharing isn't working in Zoom, try disabling `env = __GLX_VENDOR_LIBRARY_NAME,nvidia`{{< /hint >}}

## Fixing random flickering (The Nuclear Method)

Do note though that this forces performance mode to be active, resulting in
increased power-consumption (from 22W idle on a RTX 3070TI, to 74W).

This may not even be needed for some users, only apply these 'fixes' if you
in-fact do notice flickering artifacts from being idle for ~5 seconds.

Make a new file at `/etc/modprobe.d/nvidia.conf` and paste this in:

```sh
options nvidia NVreg_RegistryDwords="PowerMizerEnable=0x1; PerfLevelSrc=0x2222; PowerMizerLevel=0x3; PowerMizerDefault=0x3; PowerMizerDefaultAC=0x3"
```

Reboot your computer and it should be working.

If it does not, try:
- Lowering your monitors' refresh rate, as this can stop the flickering altogether
- Installing the 535xx versions of the drivers, as later (545, 550) can cause flickering with XWayland
  - These are available for Arch via [the AUR here](https://aur.archlinux.org/packages?O=0&K=535xx)
- Using the [Nouveau](https://wiki.archlinux.org/title/Nouveau) or nvidia-open driver.

## Fixing suspend/wakeup issues

Enable the services `nvidia-suspend.service`, `nvidia-hibernate.service` and `nvidia-resume.service`, they will be started by systemd when needed.

Add `nvidia.NVreg_PreserveVideoMemoryAllocations=1` to your kernel parameters if you don't have it already.

For Nix users, the equivalent of the above is
```nix
# configuration.nix

boot.kernelParams = [ "nvidia.NVreg_PreserveVideoMemoryAllocations=1" ];

hardware.nvidia.powerManagement.enable = true

# Making sure to use the proprietary drivers until the issue above is fixed upstream
hardware.nvidia.open = false 

```
