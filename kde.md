# Making it user friendly and adding a desktop environment

##### KDE

Install the plasma-meta meta-package `sudo pacman -S plasma-meta`
To enable support for Wayland in Plasma, also install the plasma-wayland-session package `sudo pacman -S plasma-wayland-session`
Enable the sddm service with `sudo systemctl enable sddm.service`
For basic functionality: `sudo pacman -S dolphin konsole kate kcalc ttf-dejavu ttf-liberation xdg-desktop-portal xdg-desktop-portal-gtk xdg-desktop-portal-kde xdg-desktop-portal-wlr`

##### Restart the system

Next configure KDE:

```text
System Settings > Workspace Behavior > Desktop Effects > Disable Translucency that behave bad for dark themes.
System Settings > Startup and Shutdown > Background Services > Disable Bluetooth, we don't need it
System Settings > Search > File Search > Deselect "Enable File Search"
System Settings > Regional Settings > Set Language and Formats
System Settings > Inputs Devices > Keyboard > Layouts > Check Configure layouts and add your keymap
```

Then follow https://community.kde.org/Distributions/Packaging_Recommendations

###### kdesu

`kdesu` may be used under KDE to launch GUI applications with root privileges. It is possible that by default kdesu will try to use `su` even if the root account is disabled. Fortunately one can tell kdesu to use `sudo` instead of `su`. Create/edit the file `~/.config/kdesurc`:

```text
[super-user-command]
super-user-command=sudo
```

or use the following command:

`kwriteconfig5 --file kdesurc --group super-user-command --key super-user-command sudo`

---

##### Replace PulseAudio with PipeWire + WirePlumber

`sudo pacman -S pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber`
`systemctl --user enable pipewire.socket pipewire-pulse.socket wireplumber.service`

---

## Next follow the [Nvidia instructions.](nvidia.md)
