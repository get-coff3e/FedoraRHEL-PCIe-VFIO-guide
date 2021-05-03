# Fedora-34-VFIO-guide
A helpful guide to guide through the process of enabling PCIe devices into vfio

Most of the original content is from https://github.com/ekistece/Fedora-33-VFIO-guide and improved upon! Most of the instructions are not very different from there.

# Getting the requirements
You need:
 - Fedora 34 Workstation on your system
 - Two PCIe devices in different IOMMU groups

# Enabling IOMMU
This is crucial, you need to enable the IOMMU option in your motherboard's BIOS.
```sh
#!/bin/bash
for d in /sys/kernel/iommu_groups/*/devices/*; do
  n=${d#*/iommu_groups/*}; n=${n%%/*}
  printf 'IOMMU Group %s ' "$n"
  lspci -nns "${d##*/}"
done
```
Then save this as **ls-iommu.sh** to helpfully list the IDs of all devices in a neater manner.
Run it and check if your card is in their own groups. In my case:

Guest GPU:
```
IOMMU Group 18:
	26:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Ellesmere [Radeon RX 470/480/570/570X/580/580X/590] [1002:67df] (rev e7)
	26:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Ellesmere HDMI Audio [Radeon RX 470/480 / 570/580/590] [1002:aaf0]
```

You need to take note of their IDs. (Ex: The GPUs are **1002:67df** and **1002:aaf0**.)
For now, I'll call them **GPU_ID1** **GPU_ID2** and **USB_ID**

# Setting up VFIO
VFIO is the driver used for detaching devices from the host, and letting them be used by KVM to passthrough. This process will not be the same for other linux distributions, so its best to look it up for your specific distro!

First add the drivers to your initramfs with Dracut.
```sh
$ sudo nano /etc/dracut.conf.d/vfio.conf
```
Add this to that file:
```
add_drivers+=" vfio vfio_iommu_type1 vfio_pci vfio_virqfd "
```
And remake the initramfs.
```sh
$ sudo dracut -f
```

Then we need to edit your kernel parameters in GRUB.
```sh
$ sudo nano /etc/default/grub
```

Remember the GPU ID's from before? Now it's time to use them.

For AMD use:
```
... quiet amd_iommu=on amd_iommu=pt rd.driver.pre=vfio-pci vfio-pci.ids=GPU_ID1,GPU_ID2
```

For Intel use:
```
... quiet intel_iommu=on intel_iommu=pt rd.driver.pre=vfio-pci vfio-pci.ids=GPU_ID1,GPU_ID2
```

Regenerate your GRUB config and reboot.
```sh
$ sudo grub2-mkconfig -o /etc/grub2-efi.cfg
```

After the reboot, don't panic, your guest GPU should be disabled and your specified USB controller should too.
Check if they are bound to the VFIO and PCI-STUB drivers before proceeding.
```sh
lspci -nnv
```
Look for this in the output:
```
Kernel driver in use: vfio-pci
```

# Creating the VM
The easiest way is to create one is with Virtual Machine Manager (virt-manager). Be careful at the last step to choose UEFI (OVMF) instead of BIOS. We need OVMF for this to work. Using BIOS is possible, but not what we're trying to do here as it requires some different steps.

When you have it created, open up the VM config and attach your passed through devices (Add Hardware > PCI Host Device). Check the IDs if you are not sure which ones they are, but usually it will display the card name.

Note that it's unlikely the card will have any use inside of the virtual machine until the driver software is installed or initialized. for GPUs, a black screen is normal, so you'll have to use the SPICE (or virt-viewer) remote connection to make sure the operating system boots correctly, or install the proper driver software.

If you are gonna use the KVM switch and passed through GPU right away, remove the **Video** and **Graphics** virtual devices from your VM config. It's recommended you do this after installation, especially on AMD Radeon cards as these can cause some unwanted issues. 


# My Windows 10 VM locks up after a certain amount of time!
Docs will be updated to show a fix on this
