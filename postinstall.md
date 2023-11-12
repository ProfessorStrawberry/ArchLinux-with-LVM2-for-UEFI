# POST INSTALLATION TWEAKS (IMPORTANT – YOU WILL WANT TO DO THESE)

##### Install paru or any AUR wrapper

What the AUR is, is a collection of USER created packages for Arch  Linux users to pull from. These can be game installers, programs  compiled from git repositories, beta drivers, or other programs that  aren’t included in Arch’s main repos. That being said, it is VERY smart  to LOOK at the PKGBUILD of a package before installing it, to see what  it does, where it installs things, and if the auther has added any notes about installing it. If you find a package on the AUR you want to  install, you use the AUR wrapper the same way as pacman. AUR packages can be found  here: https://aur.archlinux.org/

```text
git clone https://aur.archlinux.org/paru.git
cd paru
makepkg -si
```

Many AUR packages compile from source, so you will want to speed up  compile times. I have a few small tips for that as well. First, you will need to know the amount of cores your processor has, and will need to  know if your processor supports Hyperthreading or SMT. For example, if  you own an i7 4770k, you have 4 cores and support hyperthreading,  essentially giving you 8 cores. Ryzen 1700x 8 cores, 16 threads – counts as 16 cores. If your processor does NOT support SMT/hyperthreading,  such as an AMD FX-8350, you would just need to know the core number.  FX-8350 has 8 cores.

First install ccache: `sudo pacman -S ccache`

Next lets enable ccache and set our makeflags for makepkg. Find `BUILDENV=` remove the `!` in front of `ccache`. Find `MAKEFLAGS=`

```text
sudo nvim /etc/makepkg.conf
BUILDENV=(!distcc color ccache check !sign)
MAKEFLAGS="-jXX -lYY"
```

Replace XX with your number of cores +1 and YY with your number of cores. At the time of this edit, I currently am using a Intel Core i7 8700k, which is why I use -j13 -l12.

Next we need to make sure ccache and makeflags are set at all times  in case we compile something without using a package manager:

```text
nvim ~/.zshrc
export PATH="/usr/lib/ccache/bin/:$PATH"
export MAKEFLAGS="-j13 -l12"
```

---

##### Bluetooth

Install bluez and bluez-utils and enable/start bluetooth.service

(Might need to add user to lp group)

```text
sudo pacman -S bluez bluez-utils
sudo systemctl enable bluetooth.service
```

---

##### Additional Packages

###### TODO: List all necessary or useful packages

mpv
yt-dlp
filezilla
firefox
flatpak
libreoffice-fresh
neofetch
qbittorrent
upower
ark

packagekit-qt5
gst-libav
gst-plugin-pipewire
gst-plugins-base
gst-plugins-good
gst-plugins-bad
gst-plugins-ugly
gstreamer
gstreamer-vaapi
x264
x265
xvidcore

avahi
curl
dnsutils
firewalld
iptables-nft
net-tools
netctl
networkmanager
networkmanager-openvpn
network-manager-applet
nm-connection-editor
nss-mdns
wget
whois

cups
cups-pdf
cups-filters
cups-pk-helper
foomatic-db
foomatic-db-engine
foomatic-db-gutenprint-ppds
foomatic-db-ppds
foomatic-db-nonfree
foomatic-db-nonfree-ppds
ghostscript
gsfonts
gutenprint
print-manager
python-pillow
python-pip
python-pyqt5
python-reportlab
system-config-printer
