Check if internet is avaiable and synchronize time

```
>ping -c 3 www.google.com
>timedatectl set-ntp true
```

check if UEFI is available

```
>ls /sys/firmware/efi
->or
>efivar -l
```
list harddrives
```
>lsblk
```
wipe data on harddrive (X representing your drive. mine is sda)
```
>gdisk /dev/sdX 
>x
>z
>y
>y
```
create new partitions
```
>cgdisk /dev/sdX

>[New] ->Press Enter
>First Sector: Leave this blank ->press Enter
>Size in sectors: 512M ->press Enter
>Hex Code: EF00 ->press Enter
>Enter new partition name: boot ->press Enter

>[New] ->Press Enter
>First Sector: Leave this blank ->press Enter
>Size in sectors: Leave this blank ->press Enter
>Hex Code: 8E00 ->press Enter
>Enter new partition name: Linux LVM ->press Enter
>[Write] ->type yes ->press Enter
>[Quit]
```
for EFI with GPT, boot needs to be fat32
```
>mkfs.fat -F32 /dev/sdX1
```
encryption setup
```
>cryptsetup luksFormat /dev/sdX2
->type YES
  >enter passphrase
```
```
>cryptsetup open --type luks /dev/sdX2 archlv
  >enter passphrase
```
```
>ls /dev/mapper/archlv
>pvcreate /dev/mapper/archlv
>vgcreate archvg /dev/mapper/archlv
```

```
>lvcreate -L2G archvg -n swap
>lvcreate -L10G archvg -n root
>lvcreate -l 100%FREE archvg -n home
```

```
>mkfs.ext4 /dev/mapper/archvg-root
>mkfs.ext4 /dev/mapper/archvg-home
>mkswap /dev/mapper/archvg-swap

>mount /dev/mapper/archvg-root /mnt
>mkdir /mnt/home
>mount /dev/mapper/archvg-home /mnt/home
>mkdir /mnt/boot
>mount /dev/sdX1 /mnt/boot
>swapon /dev/mapper/archvg-swap
```

make a backup of mirrorlist and update sync pacman
```
>cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.backup
>pacman -Sy
```
install base system
```
>pacstrap /mnt base base-devel linux linux-firmware linux-headers lvm2 sudo vim git wget zsh zsh-completions networkmanager pacman-contrib reflector htop firefox
```
Generating an fstab file and change relatime on all non-boot partitions to noatime (reduces wear if using an SSD)

```
>genfstab -p /mnt >> /mnt/etc/fstab
>vim /mnt/etc/fstab
```
enter the installation and change the root password

```
>arch-chroot /mnt
>passwd
```
add a default user

```
>useradd -m -g users -G wheel,storage,power -s /bin/zsh MYUSERNAME
>passwd MYUSERNAME
```

create a sudoers.d file

```
>visudo -f /etc/sudoers.d/01_MYUSERNAME
	##Override built-in defaults
	#Defaults rootpw
	Defaults insults
	
	##User specification
	# root and users in group wheel can run anything on any machine as any user
	MYUSERNAME ALL=(ALL) ALL
```

enable the TRIM service for SSD's

```
systemctl enable fstrim.timer
```

adjust timezone

```
>ln -s /usr/share/zoneinfo/Europe/Berlin /etc/localtime
>hwclock --systohc --utc
```
add multilib to pacman and rank mirrorlist
```
>vim /etc/pacman.conf ->uncomment
	[multilib]
	Include = /etc/pacman.d/mirrorlist
>pacman -Sy

>reflector -c "Germany" -f 12 -l 12 --verbose --save /etc/pacman.d/mirrorlist
```
language settings
```
>vim /etc/locale.gen
>locale-gen
>localectl set-locale LANG=en_US.UTF-8
>echo LANG=en_US.UTF-8 >> /etc/locale.conf
>echo LC_ALL= >> /etc/locale.conf
>export LANG=C
```
generate hostname and /etc/hosts
```
>hostnamectl set-hostname MYHOSTNAME
>vim /etc/hosts
	#<ip-address>	<hostname.domain.org>	<hostname>
	127.0.0.1		localhost
	::1             localhost
	127.0.1.1		MYHOSTNAME.localdomain	MYHOSTNAME
```
edit /etc/mkinitcpio.conf HOOKS 
```
>vim /etc/mkinitcpio.conf
  HOOKS=(base udev autodetect modconf block keyboard encrypt lvm2 filesystems fsck)
>mkinitcpio -p linux
```
install the bootloader and configure loader.conf
editor 0 prevents to edit bootoptions
```
>bootctl --path=/boot/ install
>vim /boot/loader/loader.conf ->clear
	default arch
	timeout 5
	editor 0
```
edit default arch.conf
in vim, exit insert mode and type :read ! blkid -s UUID -o value /dev/sdX2 and enter the UUID in arch.conf ->replace XXXXXXX

1. Position the cursor where you want to begin cutting. 
2. Press `v` to select characters, or uppercase `V` to select whole lines, or `Ctrl-v` to select rectangular blocks (use `Ctrl-q` if `Ctrl-v` is mapped to paste). 
3. Move the cursor to the end of what you want to cut. 
4. Press `d` to cut (or `y` to copy). 
5. Move to where you would like to paste. 
6. Press `P` to paste before the cursor, or `p` to paste after. 

**Copy and paste** is performed with the same steps except for step 4 where you would press `y` instead of `d`: 

- `d` stands for *delete* in Vim, which in other editors is usually called *cut* 
- `y` stands for *yank* in Vim, which in other editors is usually called *copy* 

```
>vim /boot/loader/entries/arch.conf
	title	Arch Linux (ENCRYPTED)
	linux 	/vmlinuz-linux
	initrd 	/initramfs-linux.img
	options cryptdevice=UUID=XXXXXXX:archlv root=/dev/mapper/archvg-root quiet rw
```
install intel-ucode and add it to arch.conf
```
>pacman -S intel-ucode
>vim /boot/loader/entries/arch.conf
	initrd	/intel-ucode.img
	initrd	/initramfs-linux.img
```
## Nvidia

Now before we reboot, we are also going to set up our graphics drivers. Reason being Arch’s kernel is set to use nouveau drivers by default for nvidia cards, and some cards don’t work properly and will  cause a freeze/hang.

I assume you know which GPU you are using. Arch wiki has done a great job at documenting which drivers you need to install for your hardware.

We are using the dkms module so that we don’t have to reinstall nvidia drivers for every different kernel, if we decide to try another kernel later. To install dkms modules we need the headers for our  kernel (installed via pacstrap linux-headers)

I have an Nvidia GTX 1080 Ti, so my latest drivers will just be nvidia. I also want the multilib drivers and all dependencies required  for the nvidia package, so I will install all of the following: 

```
>pacman -S nvidia-dkms libglvnd nvidia-utils opencl-nvidia lib32-libglvnd lib32-nvidia-utils lib32-opencl-nvidia nvidia-settings
```

We will also want to set nvidia drm kernel modules:


```
>vim /etc/mkinitcpio.conf
	MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)
```

 We also need to make sure these are loaded during boot, so next we do this:

```
>vim /boot/loader/entries/arch.conf
	options cryptdevice=UUID=XXXXXXX:archlv root=/dev/mapper/archvg-root quiet rw nvidia-drm.modeset=1
```

Lastly, we need to make a pacman hook, so that any time the kernel is  updated, it automatically adds the nvidia module. This will save us a LOT of headache later on.

```
>mkdir /etc/pacman.d/hooks
>vim /etc/pacman.d/hooks/nvidia.hook
	[Trigger]
	Operation=Install
	Operation=Upgrade
	Operation=Remove
	Type=Package
	Target=nvidia
	Target=linux
	# Change the linux part above and in the Exec line if a different kernel is used

	[Action]
	Description=Update Nvidia module in initcpio
	Depends=mkinitcpio
	When=PostTransaction
	NeedsTargets
	Exec=/bin/sh -c 'while read -r trg; do case $trg in linux) exit 0; esac; done; /usr/bin/mkinitcpio -P'

```

enable the NetworkManager.service to have internet after rebooting

```
>systemctl enable NetworkManager.service
```
exit and restart into new installation
```
>exit
>umount -R /mnt
>reboot
```



# Making it user friendly and adding a desktop environment


Installing X server display manager now install X, which is our display manager.

```
>sudo pacman -S xorg-server xorg-apps xorg-xinit xorg-twm xorg-xclock xterm ttf-dejavu
```

Now we need to test if X runs. If you get a screen with a few terminals and a clock, it works! You  can type “exit” in the terminals to drop back to the command line.


```
>startx
>exit
```

Installing a desktop manager


```
>sudo pacman -S plasma sddm
>sudo systemctl enable sddm.service
```

After this, you should be able to reboot and safely log into your system!

# POST INSTALLATION TWEAKS (IMPORTANT – YOU WILL WANT TO DO THESE):

Once you are in KDE, if you use NVIDIA you will want to get rid of some screen tearing. Open a terminal, run:

```
>sudo nvidia-xconfig
>sudo nvidia-settings
	->Click X Server Display Configuration
	->For each monitor:
	->click “Advanced”
	->check Force Composition Pipeline
	->check Force Full Composition Pipeline
	->Then click Save to X Configuration File, and quit.
```

```
sudo pacman -S acpi
```

install trizen (AUR wrapper)

```
git clone https://aur.archlinux.org/trizen.git
cd trizen
makepkg -si
```

What the AUR is, is a collection of USER created packages for Arch  Linux users to pull from. These can be game installers, programs  compiled from git repositories, beta drivers, or other programs that  aren’t included in Arch’s main repos. That being said, it is VERY smart  to LOOK at the PKGBUILD of a package before installing it, to see what  it does, where it installs things, and if the auther has added any notes about installing it. If you find a package on the AUR you want to  install, you use yay the same way as pacman. AUR packages can be found  here: https://aur.archlinux.org/

Many AUR packages compile from source, so you will want to speed up  compile times. I have a few small tips for that as well. First, you will need to know the amount of cores your processor has, and will need to  know if your processor supports Hyperthreading or SMT. For example, if  you own an i7 4770k, you have 4 cores and support hyperthreading,  essentially giving you 8 cores. Ryzen 1700x 8 cores, 16 threads – counts as 16 cores. If your processor does NOT support SMT/hyperthreading,  such as an AMD FX-8350, you would just need to know the core number.  FX-8350 has 8 cores.

First install ccache:


```
>sudo pacman -S ccache
```



Next lets enable ccache and set our makeflags for makepkg. Find BUILDENV= remove the ! in front of ccache. Find MAKEFLAGS=

```
>sudo vim /etc/makepkg.conf
	BUILDENV=(!distcc color ccache check !sign)
	MAKEFLAGS="-j17 -l16"
```

Replace 17 with your number of cores +1, and 16 with your number of  cores. At the time of this edit, I currently am using a Ryzen 1700x, which is why I use -j17 -l16.

Next we need to make sure ccache and makeflags are set at all times  in case we compile something without using a package manager:

```
>vim ~/.zshrc
	export PATH="/usr/lib/ccache/bin/:$PATH"
	export MAKEFLAGS="-j17 -l16"
```

Disable root (To still be able to `sudo su` use  `sudo -i` )

```
# passwd -l root
```

 install [kdesudo](https://aur.archlinux.org/packages/kdesudo/)AUR, which has the added advantage of tab-completion for the command following. 