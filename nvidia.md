# NVIDIA Driver

##### Install the driver

We are also going to set up our graphics drivers. Reason being Arch’s kernel is set to use nouveau drivers by default for nvidia cards, and some cards don’t work properly and will  cause a freeze/hang.

I assume you know which GPU you are using. Arch wiki has done a great job at documenting which drivers you need to install for your hardware.

We are using the dkms module, so that we don’t have to reinstall nvidia drivers for every different kernel, if we decide to try another kernel later. To install dkms modules we need the headers for our kernel (Already installed via pacstrap `linux-headers` and `linux-lts-headers`. For other kernels like e.g. `linux-zen`, install `linux-zen-headers`)

```text
sudo pacman -S nvidia-dkms nvidia-utils lib32-nvidia-utils opencl-nvidia lib32-opencl-nvidia nvidia-settings libglvnd lib32-libglvnd
```

We will also want to set nvidia drm kernel modules:

```text
nvim /etc/mkinitcpio.conf
MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)
```

##### Don't forget to rebuild initramfs with `sudo mkinitcpio -P`

We also need to make sure these are loaded during boot, so next we do this:

```text
sudo nvim /boot/loader/entries/arch.conf
options cryptdevice=UUID=XXXXXXX:archlv root=/dev/mapper/archvg-root quiet rw nvidia_drm.modeset=1 nvidia_drm.fbdev=1
```

It is also helpful to add the kernel modules

```text
sudo nvim /etc/modprobe.d/nvidia.conf
options nvidia_drm modeset=1 fbdev=1 hdmi_deepcolor=1
```

##### Beta driver
Install the `nvidia-beta-dkms` or `nvidia-open-beta-dkms` (if your card is supported by the open nvidia modules). Also Install `xorg-xwayland-git`, `nvidia-utils-beta` and `libva-nvidia-driver-git`.

Make sure you have `modset=1` AND `fbdev=1` in your `kernal params`. With the beta you can also add `hdmi_deepcolor=1` if your screen supports 10-bit colour.

Lastly, we need to make a pacman hook, so that any time the kernel is updated, it automatically adds the nvidia module. This will save us a LOT of headache later on.
Make sure the Target package set in this hook is the one you have installed in steps above (e.g. nvidia, nvidia-dkms, nvidia-lts or nvidia-ck-something).

```text
mkdir /etc/pacman.d/hooks
nvim /etc/pacman.d/hooks/nvidia.hook
[Trigger]
Operation=Install
Operation=Upgrade
Operation=Remove
Type=Package
Target=nvidia-dkms
Target=linux
# Change the linux part above and in the Exec line if a different kernel is used

[Action]
Description=Update Nvidia module in initcpio
Depends=mkinitcpio
When=PostTransaction
NeedsTargets
Exec=/bin/sh -c 'while read -r trg; do case $trg in linux) exit 0; esac; done; /usr/bin/mkinitcpio -P'
```

---

## Continue with the  [Postinstall.](postinstall.md)
