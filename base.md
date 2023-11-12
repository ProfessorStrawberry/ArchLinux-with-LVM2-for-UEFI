# Here are the base install instructions

## Boot the live environment

Check if internet is avaiable with: `ping -c 3 archlinux.org` and synchronize time with internet: `timedatectl set-ntp true`

Check if UEFI is available with: `ls /sys/firmware/efi` or `efivar -l`

---

## First Setup the filesystem with [Btrfs](btrfs.md) or [Ext4](ext4.md)

---

### Further instructions after setting up the filesystem

---

Enter the installation with `arch-chroot /mnt`

Change the root password with `passwd`

---

##### TRIM support

Add `issue_discards = 1` to `/etc/lvm/lvm.conf`

Append `:allow-discards` to the cryptdevice boot entry `cryptdevice=UUID=68b25e3a-89fb-4a1b-96b9-8baf3ed62a58:archvg-root:allow-discards`

Enable the TRIM service for SSD's with: `systemctl enable fstrim.timer`

###### Optional adjust the timer to do daily TRIM

---

##### Adjust timezone

Set the time zone: `ln -s /usr/share/zoneinfo/Europe/Berlin /etc/localtime`

Run hwclock(8) to generate /etc/adjtime: `hwclock --systohc --utc`

---

##### Localization

Edit /etc/locale.gen `nvim /etc/locale.gen` and uncomment `en_US.UTF-8 UTF-8` and other needed UTF-8 locales. Generate the locales by running: `locale-gen`

Create the locale.conf(5) file, and set the `LANG` variable accordingly: `echo LANG=en_US.UTF-8 >> /etc/locale.conf`

If you set the console keyboard layout, make the changes persistent in vconsole.conf(5): `echo KEYMAP=us >> /etc/vconsole.conf`

---

##### Network configuration

Generate hostname: `hostnamectl set-hostname MYHOSTNAME`
`echo MYHOSTNAME >> /etc/hostname`

Generate /etc/hosts:
`nvim /etc/hosts`

```text
# Static table lookup for hostnames.
# See hosts(5) for details.
#
# <ip-address>  <hostname.domain.org>   <hostname>
##
## The following lines are desirable for IPv4 capable hosts
127.0.0.1       localhost
#192.168.1.10   foo.mydomain.org        foo
#192.168.1.13   bar.mydomain.org        bar
#146.82.138.7   master.debian.org       master
#209.237.226.90 www.opensource.org
### sourceforge adjustment for Germany
213.203.218.122 dl.sourceforge.net
## 127.0.1.1 is often used for the FQDN of the machine
127.0.1.1       MYHOSTNAME.localdomain 	MYHOSTNAME
#
## The following lines are desirable for IPv6 capable hosts
::1             localhost ip6-localhost ip6-loopback
ff02::1         ip6-allnodes
ff02::2         ip6-allrouters
```

Enable `NetworkManager.service` to have internet after rebooting: `systemctl enable NetworkManager.service`

---

##### Pacman adjustments

Add `multilib` to pacman `nvim /etc/pacman.conf` and uncomment:

```text
[multilib]
Include = /etc/pacman.d/mirrorlist
```

Update pacman with: `pacman -Syu`

Make a backup of mirrorlist
`cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.backup`

Make a rank for mirrorlist (adjust to your location)
`reflector -c "Germany" -f 12 -l 12 --verbose --save /etc/pacman.d/mirrorlist`

---

##### Configuring mkinitcpio HOOKS in `/etc/mkinitcpio.conf` to work with `encrypt`:

Edit `/etc/mkinitcpio.conf HOOKS` with: `nvim /etc/mkinitcpio.conf`
`HOOKS=(base udev keyboard keymap autodetect consolefont modconf block encrypt lvm2 filesystems fsck)`

Update the initramfs with: `mkinitcpio -P`

---

##### Boot loader + Microcode

For AMD processors, install the `amd-ucode` package. For Intel processors, install the `intel-ucode` package.

`pacman -S refind amd-ucode intel-ucode`

Copy linux image to ESP:

```text
mkdir -p /efi/EFI/arch
cp -a /boot/vmlinuz-linux /efi/EFI/arch/
cp -a /boot/initramfs-linux.img /efi/EFI/arch/
cp -a /boot/initramfs-linux-fallback.img /efi/EFI/arch/
cp -a /boot/vmlinuz-linux-lts /efi/EFI/arch/
cp -a /boot/initramfs-linux-lts.img /efi/EFI/arch/
cp -a /boot/initramfs-linux-lts-fallback.img /efi/EFI/arch/
cp -a /boot/{amd,intel}-ucode.img /efi/EFI/arch/
```

Install refind with: `refind-install`

Then we need to edit `/boot/refind_linux.conf`:

```text
"Boot with default options"  "cryptdevice=UUID=XXXXXXX:archlv:allow-discards root=/dev/archvg/root rw add_efi_memmap initrd=\EFI\arch\intel-ucode.img initrd=\EFI\arch\initramfs-%v.img"
"Boot with fallback initramfs"    "cryptdevice=UUID=XXXXXXX:archlv root=/dev/archvg/root rw add_efi_memmap initrd=\EFI\arch\intel-ucode.img initrd=\EFI\arch\initramfs-%v-fallback.img"
"Boot to terminal"   "cryptdevice=UUID=XXXXXXX:archlv root=/dev/archvg/root rw add_efi_memmap systemd.unit=multi-user.target"
```

Note: Use backslashes `\` for initrd and forward slashes `/` for other attributes.

###### To get the UUID in `cryptdevice=UUID=:archlv`

In nvim, exit insert mode and type `:read ! blkid -s UUID -o value /dev/sdX2` and enter the UUID ->replace XXXXXXX

```text
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

###### Optional: For `Btrfs` add `rootflags=subvol=@` so it looks like this

```text
"Boot with default options"  "cryptdevice=UUID=XXXXXXX:archlv:allow-discards root=/dev/archvg/root rootflags=subvol=@ rw add_efi_memmap initrd=\EFI\arch\intel-ucode.img initrd=\EFI\arch\initramfs-%v.img"
"Boot with fallback initramfs"    "cryptdevice=UUID=XXXXXXX:archlv root=/dev/archvg/root rootflags=subvol=@ rw add_efi_memmap initrd=\EFI\arch\intel-ucode.img initrd=\EFI\arch\initramfs-%v-fallback.img"
"Boot to terminal"   "cryptdevice=UUID=XXXXXXX:archlv root=/dev/archvg/root rootflags=subvol=@ rw add_efi_memmap systemd.unit=multi-user.target"
```

Edit `/efi/EFI/refind/refind.conf` in order to work with %v in `refind_linux.conf`:

```text
...
extra_kernel_version_strings linux-lts,linux
...
```

So this way we have to configure the boot entries only once for multiple kernels.

Lastly bind mount the ESP

###### Do not bind mount the ESP to `/boot` before using `refind-install`, else it will fail

`mount --bind /efi/EFI/arch /boot`

---

##### Exit and restart into new installation

```text
exit
umount -R /mnt
reboot
```

---

## Continue with the [User setup.](user.md)
