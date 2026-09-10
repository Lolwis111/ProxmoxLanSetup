# ProxmoxLanSetup

## Hardware setup

- Xeon E5-2698v3
- Asus X99-E WS
- 4x 16GB DDR4
- 4x Nvidia Quadro M2000
- 1x NVMe boot drive 1TB
- 2x USB Controller card
- 1x Intel X520-DA2 networking (optional)

### BIOS Settings

- Enable VM Extension (Intel VT-x / AMD-V
- Enable VM IOMMU (Intel VT-d / AMD-Vi)
- Give each device own iommu group (Asus Board: ACS Option on)

## Software Setup

### Setup IOMMU

add to /etc/kernel/cmdline:
```intel_iommu=on```

then run:
```proxmox-boot-tool refresh```

### Load Modules

load the following modules (e.g. via /etc/modules):
```
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```

### Unload Modules

blacklist gpu drivers:
```
echo "options vfio_iommu_type1 allow_unsafe_interrupts=1" > /etc/modprobe.d/iommu_unsafe_interrupts.conf
echo "options kvm ignore_msrs=1" > /etc/modprobe.d/kvm.conf

echo "blacklist nouveau" >> /etc/modprobe.d/blacklist.conf
echo "blacklist nvidia" >> /etc/modprobe.d/blacklist.conf
```

### Find GPUs

```
lspci | grep -i vga
```

sample output:
```
05:00.0 VGA compatible controller: NVIDIA Corporation GM206GL [Quadro M2000] (rev a1)
07:00.0 VGA compatible controller: NVIDIA Corporation GM206GL [Quadro M2000] (rev a1)
0b:00.0 VGA compatible controller: NVIDIA Corporation GM206GL [Quadro M2000] (rev a1)
0d:00.0 VGA compatible controller: NVIDIA Corporation GM206GL [Quadro M2000] (rev a1)
```

find details about GPUs (replace 82:00 with id):
```
lspci -n -s 82:00 -v
```

sample output (id in this case 10de:1430):
```
07:00.0 0300: 10de:1430 (rev a1) (prog-if 00 [VGA controller])
        Subsystem: 10de:1190
        Flags: bus master, fast devsel, latency 0, IRQ 170, NUMA node 0, IOMMU group 46
        Memory at f8000000 (32-bit, non-prefetchable) [size=16M]
        Memory at a0000000 (64-bit, prefetchable) [size=256M]
        Memory at b0000000 (64-bit, prefetchable) [size=32M]
        I/O ports at 8000 [size=128]
        Expansion ROM at f9000000 [disabled] [size=512K]
        Capabilities: [60] Power Management version 3
        Capabilities: [68] MSI: Enable+ Count=1/1 Maskable- 64bit+
        Capabilities: [78] Express Legacy Endpoint, IntMsgNum 0
        Capabilities: [100] Virtual Channel
        Capabilities: [258] L1 PM Substates
        Capabilities: [128] Power Budgeting <?>
        Capabilities: [420] Advanced Error Reporting
        Capabilities: [600] Vendor Specific Information: ID=0001 Rev=1 Len=024 <?>
        Capabilities: [900] Secondary PCI Express
        Kernel driver in use: vfio-pci
        Kernel modules: nvidiafb, nouveau, nova_core

07:00.1 0403: 10de:0fba (rev a1) (prog-if 00 [HDA compatible])
        Subsystem: 10de:1190
        Flags: bus master, fast devsel, latency 0, IRQ 36, NUMA node 0, IOMMU group 46
        Memory at f9080000 (32-bit, non-prefetchable) [size=16K]
        Capabilities: [60] Power Management version 3
        Capabilities: [68] MSI: Enable- Count=1/1 Maskable- 64bit+
        Capabilities: [78] Express Endpoint, IntMsgNum 0
        Capabilities: [100] Advanced Error Reporting
        Kernel driver in use: vfio-pci
        Kernel modules: snd_hda_intel
```

add ids to vfio options:
```echo "options vfio-pci ids=10de:1430 disable_vga=1"> /etc/modprobe.d/vfio.conf```


apply all changes:
```update-initramfs -u -k all```

then:
```reboot```

## VM Config
### Virtual Hardware
- 4 vCores per VM (pin cores for more performance)
- 12 GB RAM (balloning=0 !!)
- passthrough one of the GPUs from earlier
- passthrough one USB Controller (or assign individual ports)
- add network bridge with VirtIO
- 100 GB vDisk with VirtIO SCSI Single Controller
- *Important*: set BIOS to OVMF (UEFI)
### Windows
- Install Windows 10 as normal
- Load VirtIO Driver ISO to detect harddisk and get network going
