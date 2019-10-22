```
>timedatectl set-ntp true
```

check if UEFI is available

```
>ls /sys/firmware/efi
or
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

>[New] Press Enter
>First Sector: Leave this blank ->press Enter
>Size in sectors: 512M ->press Enter
>Hex Code: EF00 press Enter
>Enter new partition name: boot ->press Enter

>[New] Press Enter
>First Sector: Leave this blank ->press Enter
>Size in sectors: Leave this blank ->press Enter
>Hex Code: 8E00 ->press Enter
>Enter new partition name: Linux LVM ->press Enter
>exit
```
for EFI with GPT, boot needs to be fat32
```
>mkfs.fat -F32 /dev/sdX1 
```
encryption setup
```
>cryptsetup luksFormat /dev/sdX2
>YES
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

rank mirrorlist
```
>cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.backup
>pacman -S reflector
>reflector -c "Germany" -f 12 -l 12 --verbose --save /etc/pacman.d/mirrorlist
```
install base system
```
>pacstrap /mnt base linux linux-firmware
```
```
>genfstab -p /mnt >> /mnt/etc/fstab
```
enter the installation
```
>arch-chroot /mnt
```
adjust timezone
```
>ln -s /usr/share/zoneinfo/Europe/Berlin /etc/localtime
>hwclock --systohc --utc
```
edit root password
```
>passwd
```
install additional packages
```
>pacman -S base-devel lvm2 sudo vim
```
add multilib to pacman
```
>vim /etc/pacman.conf
  [multilib]
  Include = /etc/pacman.d/mirrorlist
>pacman -Sy
```
language settings
```
>vim /etc/locale.gen
>locale-gen
>locale > /etc/locale.conf
```
generate hostname and /etc/hosts
```
>vim /etc/hostname
  >enter hostname
>vim /etc/hosts
  #<ip-address>		<hostname.domain.org>		<hostname>
  127.0.0.1       	localhost.localdomain		localhost
  ::1             	localhost.localdomain		localhost
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
>vim /boot/loader/loader.conf
  clear
  default arch
  timeout 5
  editor 0
```
edit default arch.conf
in vim, exit insert mode and type :read ! blkid /dev/sdX2 and enter the UUID in arch.conf
```
>vim /boot/loader/entries/arch.conf
  title		Arch Linux (ENCRYPTED)
  linux 	/vmlinuz-linux
  initrd 	/initramfs-linux.img
  options cryptdevice=UUID=XXXXXXXXXXX:archlv root=/dev/mapper/archvg-root quiet rw
>:read ! blkid /dev/sdX2
  enter UUID above
```
install intel-ucode and add it to arch.conf
```
>pacman -S intel-ucode
>vim /boot/loader/entries/arch.conf
  initrd	/intel-ucode.img
  initrd	/initramfs-linux.img
```
make the ethernet device avaiable at boot and install NetworkManager
```
>ip link
>systemctl enable dhcpcd@interface.service
pacman -S networkmanager
systemctl enable NetworkManager.service
```
exit and restart into new installation
```
>exit
>umount -R /mnt
>reboot
```