# Here are the Btrfs instructions

## For Ext4 see [ext4.md](ext4.md)

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

`mkfs.btrfs /dev/mapper/archvg-root`
`mount /dev/mapper/archvg-root /mnt`

*(Optional: mount the swap partition with: `mkswap /dev/mapper/archvg-swap` and `swapon -d /dev/mapper/archvg-swap`)*

---

### Root File System Setup

##### Create Btrfs subvolumes

```text
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@snapshots
```

Unmount system partition with: `umount /mnt`

##### Mount Btrfs subvolumes

```text
mount -o compress=lzo,discard,noatime,nodiratime,subvol=@ /dev/mapper/archvg-root /mnt
mkdir /mnt/home
mkdir /mnt/.snapshots
mount -o compress=lzo,discard,noatime,nodiratime,subvol=@home /dev/mapper/archvg-root /mnt/home
mount -o compress=lzo,discard,noatime,nodiratime,subvol=@snapshots /dev/mapper/archvg-root /mnt/.snapshots
```

##### Create nested subvolumes for special folders

```text
mkdir -p /mnt/var/cache/pacman
btrfs subvolume create /mnt/var/cache/pacman/pkg
btrfs subvolume create /mnt/var/log
btrfs subvolume create /mnt/var/tmp
```

##### Mount your filesystems

`mkdir /mnt/boot/ && mkdir /mnt/efi`

**Mount ESP to `/efi` outside `/boot` with:** `mount /dev/sdX1 /mnt/efi`

---

**Make sure to include `btrfs-progs` into pacstrap with: `pacstrap /mnt btrfs-progs` or install with pacman: `pacman -S btrfs-progs`**

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

UUID=...  /            btrfs  rw,noatime,nodiratime,compress=lzo,ssd,discard,space_cache,subvolid=257,subvol=/@,subvol=@  0 0
UUID=...  /home        btrfs  rw,noatime,nodiratime,compress=lzo,ssd,discard,space_cache,subvolid=258,subvol=/@home,subvol=@home  0 0
UUID=...  /.snapshots  btrfs  rw,noatime,nodiratime,compress=lzo,ssd,discard,space_cache,subvolid=259,subvo=/@snapshots,subvol=@snapshots 0 0

# If swap was created
UUID=...  /none        swap   defaults  0 0


# Mount the EFI mountpoint at boot since we mounted ESP outside of `/boot`
/efi/EFI/arch	/boot	none	defaults,bind	0 0

# Ramdisk tmp
tmpfs     /tmp         tmpfs  defaults,noatime,mode=1777  0 0
```

---

## Continue with the [Install instructions](base.md)

---

##### Install, configure and enable Snapper
```text
sudo pacman -S snapper
sudo umount /.snapshots
sudo rm -r /.snapshots
sudo snapper -c root create-config /
sudo mount -o compression=lzo,discard,noatime,nodiratime,subvol=@snapshots /dev/mapper/archvg-root /.snapshots
sudo systemctl start snapper-timeline.timer
```
