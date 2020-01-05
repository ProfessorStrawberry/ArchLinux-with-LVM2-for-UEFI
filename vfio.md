Requiered network tools (not sure about bridge-utils)

```
>sudo pacman -S ebtables dnsmasq iptables
```

https://www.happyassassin.net/2014/07/23/bridged-networking-for-libvirt-with-networkmanager-2014-fedora-21/

### Bypassing the IOMMU groups (ACS override patch)

If you find your PCI devices grouped among others that you do not  wish to pass through, you may be able to seperate them using Alex  Williamson's ACS override patch. Make sure you understand [the potential risk](https://vfio.blogspot.com/2014/08/iommu-groups-inside-and-out.html) of doing so. 

You will need a kernel with the patch applied. The easiest method to acquiring this is through the [linux-vfio](https://aur.archlinux.org/packages/linux-vfio/)AUR package. 

```
>sudo pacman -S linux-vfio
```

In addition, the ACS override patch needs to be enabled with  kernel command line options.  The patch file adds the following  documentation:

```
pcie_acs_override =
        [PCIE] Override missing PCIe ACS support for:
    downstream
        All downstream ports - full ACS capabilties
    multifunction
        All multifunction devices - multifunction ACS subset
    id:nnnn:nnnn
        Specfic device - full ACS capabilities
        Specified as vid:did (vendor/device ID) in hex
```

The option `pcie_acs_override=downstream` is typically sufficient.



```
>sudo pacman -S qemu libvirt ovmf virt-manager
```

```
/etc/libvirt/qemu.conf
nvram = [
	"/usr/share/ovmf/x64/OVMF_CODE.fd:/usr/share/ovmf/x64/OVMF_VARS.fd"
]
```

You can now [enable](https://wiki.archlinux.org/index.php/Enable) and [start](https://wiki.archlinux.org/index.php/Start) `libvirtd.service` and its logging component `virtlogd.socket`.

You may also need to [activate the default libvirt network](https://wiki.libvirt.org/page/Networking#NAT_forwarding_.28aka_.22virtual_networks.22.29).



### With vfio-pci loaded as a module

[linux](https://www.archlinux.org/packages/?name=linux) kernel does not include vfio-pci as a built-in module and therefore  needs to be loaded en configured separately like so.  First, the vendor-device ID pairs must be specified as default  parameters passed to vfio-pci whenever it is inserted into the kernel.

```
/etc/modprobe.d/vfio.conf
options vfio-pci ids=10de:13c2,10de:0fbb
```

This, however, does not guarantee that vfio-pci will be loaded before other graphics drivers. To ensure that, we need to statically bind it  in the kernel image alongside with its dependencies. That means adding,  in this order, `vfio_pci`, `vfio`, `vfio_iommu_type1`, and `vfio_virqfd` to [mkinitcpio](https://wiki.archlinux.org/index.php/Mkinitcpio):

```
/etc/mkinitcpio.conf
MODULES=(... vfio_pci vfio vfio_iommu_type1 vfio_virqfd ...)
```

**Note:** If you also have another driver loaded this way for [early modesetting](https://wiki.archlinux.org/index.php/Kernel_mode_setting#Early_KMS_start) (such as `nouveau`, `radeon`, `amdgpu`, `i915`, etc.), all of the aforementioned VFIO modules must precede it.

Also, ensure that the modconf hook is included in the HOOKS list of `mkinitcpio.conf`:

```
/etc/mkinitcpio.conf
HOOKS=(... modconf ...)
```

Since new modules have been added to the initramfs configuration, you must [regenerate the initramfs](https://wiki.archlinux.org/index.php/Regenerate_the_initramfs). Should you change the IDs of the devices in `/etc/modprobe.d/vfio.conf`, you will also have to regenerate it, as those parameters must be  specified in the initramfs to be known during the early boot stages.



### Setting up the guest OS

The process of setting up a VM using `virt-manager` is mostly self-explanatory, as most of the process comes with fairly comprehensive on-screen instructions.

If using `virt-manager`, you have to add your user to the `libvirt` [user group](https://wiki.archlinux.org/index.php/User_group) to ensure authentication.