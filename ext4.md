# Here are the Ext4 instructions

## For Btrfs see [btrfs.md](btrfs.md)

### Prepare Harddrive

List harddrives with `lsblk` or `fdisk -l`

Wipe data on harddrive *(sd`X` representing your drive. mine is sda)*: `gdisk /dev/sdX`
Then type `x`, then `z`, then `y` and `y`

##### Create new partitions

`cgdisk /dev/sdX`

```text
[New] ->Press Enter
First Sector: Leave this blank ->press Enter
Size in sectors: 512M ->press Enter
Hex Code: EF00 ->press Enter
Enter new partition name: boot ->press Enter

[New] ->Press Enter
First Sector: Leave this blank ->press Enter
Size in sectors: Leave this blank ->press Enter
Hex Code: 8E00 ->press Enter
Enter new partition name: Linux LVM ->press Enter
[Write] ->type yes ->press Enter
[Quit]
```

*(for EFI with GPT, `/boot` needs to be fat32)*
`mkfs.fat -F32 /dev/sdX1`

---

### Encryption Setup

Create the LUKS encrypted container:
`cryptsetup luksFormat /dev/sdX2` *(type `YES` and enter `passphrase`)*

Open the LUKS container:
`cryptsetup open --type luks /dev/sdX2 archlv` *(enter `passphrase`)*

---

### Preparing the logical volumes

*(List the logical volume with: `ls /dev/mapper/archlv`)*

`pvcreate /dev/mapper/archlv`
`vgcreate archvg /dev/mapper/archlv`

*(Optional: **First** create swap partition with: `lvcreate -L 8G archvg -n swap`)*

`lvcreate -l 100%FREE archvg -n root`

---

### Format your filesystems on each logical volume

`mkfs.ext4 /dev/mapper/archvg-root`


*(Optional: mount the swap partition with: `mkswap /dev/mapper/archvg-swap` and `swapon -d /dev/mapper/archvg-swap`)*

---

### Root File System Setup

##### Mount your filesystems

`mount /dev/mapper/archvg-root /mnt` or  `mount /dev/archvg/root /mnt`
`mkdir /mnt/boot/ && mkdir /mnt/efi`

**Mount ESP to `/efi` outside `/boot` with:** `mount /dev/sdX1 /mnt/efi`

---

##### Install the base + some useful packages

###### I am using zsh as default shell, neovim as default editor

`pacstrap -K /mnt base base-devel linux linux-headers linux-lts linux-lts-headers linux-firmware sof-firmware lvm2 efibootmgr sudo neovim git wget curl rsync zsh networkmanager pacman-contrib reflector htop`

---

##### Generate an fstab file by UUID

`genfstab -U /mnt >> /mnt/etc/fstab`

**Check `/mnt/etc/fstab` correctness and add `/efi/EFI/arch /boot none defaults,bind 0 0` to mount the EFI mountpoint at boot since we mounted ESP outside of `/boot`. Also change `relatime` on all non-boot partitions to `noatime` (reduces wear if using an SSD) and Optional: add Ramdisk tmp**

`vim /mnt/etc/fstab`

###### So you should have something similar to this

```text
# Static information about the filesystems.
# See fstab(5) for details.

# <file system> <dir> <type> <options> <dump> <pass>

# /dev/mapper/archvg-root
UUID=b1566d2d-96db-4efb-8098-06cbdc2ba17d	/         	ext4      	rw,noatime	0 1

# /dev/sda1
UUID=3203-97C0      	/efi      	vfat      	rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=iso8859-1,shortname=mixed,utf8,errors=remount-ro	0 2

# If swap was created /dev/mapper/archvg-swap
UUID=e0729c70-df94-44c9-849a-1f1cfedc5db8	none      	swap      	defaults,pri=-2	0 0

# Mount the EFI mountpoint at boot since we mounted ESP outside of `/boot`
/efi/EFI/arch	/boot	none	defaults,bind	0 0

# Ramdisk tmp
tmpfs     /tmp         tmpfs  defaults,noatime,mode=1777  0 0
```

---

## Continue with the [Install instructions](base.md)
