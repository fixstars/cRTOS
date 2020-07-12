Installation
============

We assume you have down the following:
If you have difficult to do this, reference the Appendix A.
 1. Install a Linux on the target machine
 2. Replace the kernel with Linux 5.4 PREEMPT_RT
 3. Have installed gcc-9

First, clone the root repository, which is the one containing this README.md.

```
git clone 
```

Then you need to download all 5 repositories containing the cRTOS source code.
Run the following in the root repository to initialize the submodules.

```
git submodule --init update
```

From here I will give the general instruction on how to build each packages.
However, for detail configurations, please reference the documentations of each packages.

Jailhouse
---------

Jailhouse is used to boot the Nuttx sRTOS side by side with Linux.
The following instruction will teach you how to build and configure Jailhouse.

### Compile and Install

```sh
# Switch into the jailhouse directory
cd crtos-jailhouse

# Build jailhouse
make

# Install Jailhouse at system level
sudo make install
```

### Configuration

Jailhouse cell configuration files are stored in `configs/x86`.
The  2 sample cell configuration files for cRTOS are in that directory.
 * configs/x86/sysconfig.c
 * configs/x86/nuttx.c

However, even with the same machine as the tested machine, 
the required configuration still differs because of the x86 EFI and APCI features.

You will need to generate your own configuration files.
Luckily it will not be too difficult if the follow the instructions, "CAREFULLY".
In addition, you can always try to reference the sample configuration files.

#### Root call configuration file
First, generate the root cell configuration file.
This is done using the jailhouse command-line tool.
Generally cRTOS requires:
 * 128 MB of physical RAM to be reserved from Linux for Jailhouse.
 * 1.5 GB of physical RAM to be reserved from Linux for sRTOS.

```sh
sudo jailhouse config create -c ttyS0 --mem-hv 128M --mem-inmates 1536M configs/x86/sysconfig.c
```

You can tune the memory required to smaller amount,
but you need to change the memory settings in the following sections about cell memory accordingly.
sRTOS requires at least 512 MB of memory.

##### Memory regions for inmate cells

Open the generated file and navigates to the Inmate Memory regions.
We need to split the Inmate Memory into 2 two-cell jailhouse ivshmem devices' regions.
One for the standard inter-cell Ethernet.
One for the remote system call shadow device.

A two-cell ivshmem device has 4 memory regions. All of them have to be backed by physical region.
 * PAGE_SIZE state table
 * R/W region
 * Output Region for cell 0, Input Region for Cell 1
 * Input Region for cell 0, Output Region for Cell 1

The Ethernet device regions cal be created easily with built-in marcos.
However, the shadow device regions have to be hand crafted.

Inmate Memory defined in your root config should look like this.
(The value will vary accordingly to your system.)
```c
/* MemRegion: <N>-<N+0x60000000> : JAILHOUSE Inmate Memory */
{
  .phys_start = <N>,
  .virt_start = <N>,
  .size = 0x60000000,
  .flags = JAILHOUSE_MEM_READ | JAILHOUSE_MEM_WRITE,
},
```
This is the Inmate Memory can be used for the sRTOS cell.
All of the following ivshmem memory should be allocated from this region.

First, remove this entry in the memory region definition.

Then, you should create memory regions for the shadow device with:
 * PAGE_SIZE state table
 * 1GB R/W Region, marked with execute
 * 4x PAGE_SIZE Input and Output regions
Allocate the memory from the Inmate Memory manually.
Append these after the Inmate Memory region.

```c
{
  .phys_start = <N>,
  .virt_start = <N>,
  .size = 0x1000,
  .flags = JAILHOUSE_MEM_READ | JAILHOUSE_MEM_ROOTSHARED,
},
{
  .phys_start = <N + 0x1000>,
  .virt_start = <N + 0x1000>,
  .size =       0x40000000,
  .flags = JAILHOUSE_MEM_READ | JAILHOUSE_MEM_WRITE |
    JAILHOUSE_MEM_EXECUTE | JAILHOUSE_MEM_ROOTSHARED,
},
{
  .phys_start = <N + 0x40001000>,
  .virt_start = <N + 0x40001000>,
  .size = 0x4000,
  .flags = JAILHOUSE_MEM_READ | JAILHOUSE_MEM_WRITE | JAILHOUSE_MEM_ROOTSHARED,
},
{
  .phys_start = <N + 0x40005000>,
  .virt_start = <N + 0x40005000>,
  .size = 0x4000,
  .flags = JAILHOUSE_MEM_READ | JAILHOUSE_MEM_ROOTSHARED,
},
```

After that, you need to add the Ethernet device regions.
```
  JAILHOUSE_SHMEM_NET_REGIONS(<N + 0x40205000>, 0),
```

After modifying the memory regions, remember to increase the number of memory regions in the config header.
```c
/* Before */
struct jailhouse_memory mem_regions[<NUM OF MEM>];

/* After */
struct jailhouse_memory mem_regions[<NUM OF MEM + 7>];
```

##### Ivshmem Device

Having the backing memory allocated, we now need to create the PCI devices for ivhsmem.

Navigates to the PCI device regions of the configuration files.

Add 2 ivshmem devices to it.
You need to modify the <DOM> and <IOMMU> according to you system.
For the supermirco setup, the value is 0 for <DOM> and 1 for <IOMMU>.
Inquiry the Jailhouse mailing lit for more information.
(Seriously, I have no idea how to decide these values. I also inquired the mailing list.)
```
{ /* IVSHMEM (shadow) */
  .type = JAILHOUSE_PCI_TYPE_IVSHMEM,
  .domain = <DOM>,
  .iommu = <IOMMU>,
  .bdf = 0x0e << 3,
  .bar_mask = JAILHOUSE_IVSHMEM_BAR_MASK_MSIX,
  .num_msix_vectors = 16,
  .shmem_regions_start = <NUM OF MEM - 1>,
  .shmem_dev_id = 0,
  .shmem_peers = 2,
  .shmem_protocol = 0x0002,
},
{ /* IVSHMEM-NET */
  .type = JAILHOUSE_PCI_TYPE_IVSHMEM,
  .domain = <DOM>,
  .iommu = <IOMMU>,
  .bdf = 0x0d << 3,
  .bar_mask = JAILHOUSE_IVSHMEM_BAR_MASK_MSIX,
  .num_msix_vectors = 2,
  .shmem_regions_start = <NUM OF MEM + 3>,
  .shmem_dev_id = 0,
  .shmem_peers = 2,
  .shmem_protocol = JAILHOUSE_SHMEM_PROTO_VETH,
}
```

After modifying the PCI device regions, remember to increase the number of devices in the config header.

```c
/* Before */
struct jailhouse_pci_device pci_devices[<NUM of Device>];

/* After */
struct jailhouse_pci_device pci_devices[<NUM of Device> + 2];
```

#### sRTOS cell configurations

It is tedious to write a inmate cell configuration from scratch.
We will modify one out from the sample configuration file.
We will call it `nuttx.c`, placed in `configs/x86`

Open the file and do the following:
 * Clear out the memory regions
 * Clear out the PCI device regions
 * Clear out the PCI capability regions
 * Copy the irqchip regions from you root cell

##### CPU

The CPU setting is a CPU bit mask.
Give any of the processor you like to it.
Currently cRTOS's sRTOS only support Uni-processor.

For example, assigning core 3 (counting from 0):
```c
.cpus = {
  0x8,
},
```

##### LLC partitioning

With a Xeon processor, you are able to dedicate parts of the LLC to the sRTOS.
This is done via the cache_regions setting.

For example to give <S> slices of the LLC, starting from 0, to the sRTOS.
(Number of slice and slice size is processor dependent.)
```
.cache_regions = {
  {
    .start = 0,
    .size = <S>,
    .type = JAILHOUSE_CACHE_L3,
  },
},
```

Reference the Jailhouse documentation or mailing list for more information.

##### PIO regions

To give access to legacy x86 PIO devices.
You should add entries describing the devices regions in PIO regions.

As for cRTOS, you can leave this part empty, omitting this setting.

##### Memory regions

The sRTOS cell should also have the 8 memory region for the 2 ivshmem devices.
The regions of both root and sRTOS cell must have the same physical address. (Otherwise, how do we share memory? :) )

It should contains the following:
 * PAGE_SIZE state table of shadow device.
 * 1GB R/W region of shadow device (R/W, ROOT SHARED, EXECUTE, LOADABLE) mapped at virtual address 0x0.
 * Input and output region of shadow device.
 * VM communication region
 * Ivshmem Ethernet device regions

In summary:
```
/* cRTOS Shadow memory */
{
  .phys_start = <N>,
  .virt_start = <N>,
  .size = 0x1000,
  .flags = JAILHOUSE_MEM_READ | JAILHOUSE_MEM_ROOTSHARED,
},
{
  .phys_start = <N + 0x1000>,
  .virt_start = 0,
  .size = 0x40000000,
  .flags = JAILHOUSE_MEM_READ | JAILHOUSE_MEM_WRITE |
    JAILHOUSE_MEM_EXECUTE | JAILHOUSE_MEM_LOADABLE |
    JAILHOUSE_MEM_ROOTSHARED,
},
{
  .phys_start = <N + 0x40001000>,
  .virt_start = <N + 0x40001000>,
  .size = 0x4000,
  .flags = JAILHOUSE_MEM_READ | JAILHOUSE_MEM_ROOTSHARED,
},
{
  .phys_start = <N + 0x40005000>,
  .virt_start = <N + 0x40005000>,
  .size = 0x4000,
  .flags = JAILHOUSE_MEM_READ| JAILHOUSE_MEM_WRITE | JAILHOUSE_MEM_ROOTSHARED,
},
/* communication region */ {
  .virt_start = <N + 0x40010000>,
  .size       = 0x00001000,
  .flags = JAILHOUSE_MEM_READ | JAILHOUSE_MEM_WRITE |
    JAILHOUSE_MEM_COMM_REGION,
},
JAILHOUSE_SHMEM_NET_REGIONS(<N + 0x40205000>, 1),
```

After modifying the memory regions, remember to set the correct number of regions in the config header as you do it in root cell configuration.

##### Ivshmem Device

Having the backing memory allocated, we now need to create the PCI devices for ivhsmem.

Navigates to the PCI device regions of the configuration files.

Add 2 ivshmem devices to it. For <DOM> and <IOMMU> setting from root cell configuration.
```
{ /* Shadow */
  .type = JAILHOUSE_PCI_TYPE_IVSHMEM,
  .domain = <DOM>,
  .iommu = <IOMMU>,
  .bdf = 0x0e << 3,
  .bar_mask = JAILHOUSE_IVSHMEM_BAR_MASK_MSIX,
  .num_msix_vectors = 16,
  .shmem_regions_start = 0,
  .shmem_dev_id = 1,
  .shmem_peers = 2,
  .shmem_protocol = 0x0002,
},
{ /* IVSHMEM-NET */
  .type = JAILHOUSE_PCI_TYPE_IVSHMEM,
  .domain = <DOM>,
  .iommu = 1<IOMMU>,
  .bdf = 0x0d << 3,
  .bar_mask = JAILHOUSE_IVSHMEM_BAR_MASK_MSIX,
  .num_msix_vectors = 2,
  .shmem_regions_start = 18,
  .shmem_dev_id = 1,
  .shmem_peers = 2,
  .shmem_protocol = JAILHOUSE_SHMEM_PROTO_VETH,
},
```

After modifying the PCI device regions, remember to set the correct number of regions in the config header as you do it in root cell configuration.

##### Physical Device

To have a debug serial console for Nuttx, we will map a physical PCI-e to serial adapter to sRTOS cell.

Currently the only supported device are MCS99xx PCI-e to serial adapters.

First, you need to found out the PCI bus address of the device.
```sh
lspci

# For example the device is at [03:00]
03:00.0 Serial controller: MosChip Semiconductor Technology Ltd. PCIe 9912 Multi-I/O Controller
03:00.1 Serial controller: MosChip Semiconductor Technology Ltd. PCIe 9912 Multi-I/O Controller
03:00.2 Parallel controller: MosChip Semiconductor Technology Ltd. PCIe 9912 Multi-I/O Controller
```

To allocate the device in the sRTOS cell configuration file:
 1. Search the root cell configuration file for the device by PCI bus address.
 2. Then copy the following items to the corresponding regions sRTOS cell configuration file.

 * memory regions
 * PCI device regions
 * PCI capability regions
 * PIO regions related to the PCI bus address

Then, modify the starting index of PCI capabilities in the PCI device region to the new index in the new configuration file.
Also remember to increase each region's size in the header.

An example to pass an device is give in Appendix C.

#### Host kernel command-line

The host has to be booted with part of the physical memory reserved (hypervisor and Inmate Memory).
To achieve this, append the following to the kernel command-line.
```
intel_iommu=off memmap=<Size>$<Start address>
```

For the actual address and size of your machine, reference the root cell configuration.
There should be information about kernel command-line.

In Ubuntu, edit the `/etc/default/grub` and `grub-update`. ($ have to be escaped twice)
```
GRUB_CMDLINE_LINUX="console=tty intel_iommu=off memmap=<Size>\\\$<Start address>"
```

### Testing

After booting with the new kernel command-line.
It's time to test the setup and configurations.

```sh
# Goto the jailhouse source directory

# Build the newly writte configurations
make

# Start the jailhouse hypervisor and root cell
insmod /lib/modules/`uname -r`/extra/driver/jailhouse.ko
jailhouse enable ./configs/x86/sysconfig.cell
```

Debug messages will be output via `/dev/ttyS0`.
This should success fully "insert" the jailhouse hypervisor under host Linux, making host Linux as the root cell.

If your using a machine with buggy PCI devices causing errors, reference Appendix D for more informations.
(The supermicro motherboard, we tested, has such bug.)

Nuttx and apps
--------------

### Configuration and Build

There is an default configuration for cRTOS.
We will use it.
```sh
cd crtos-nuttx
make distclean
tools/configure.sh qemu-intel64:crtos
```

This configuration should work out of the box if you have managed the memory exactly as the section above.
However, if you shrink the memory region because some reasons. You need to configure the memory size.
Edit `.config` or use `make menuconfig`:
```sh
CONFIG_TUX_USER_ADDR_START=0x8000000
CONFIG_TUX_USER_ADDR_SIZE=<Half of the memory availble to sRTOS>
```

Also, the default config assumes you have MCS99xx serial device attached to the sRTOS cell.
It will use the first MCS99xx serial as the debug console.

For more detailed configuration, reference the sRTOS_Config.md.

### Loading as sRTOS cell

After a successful build, you can try to load it as the sRTOS cell in jailhouse.
```sh
jailhouse cell create ../crtos-jailhouse/configs/x86/nuttx.cell
jailhouse cell load nuttx ./nuttx.bin
jailhouse cell start nuttx
```

Supporting drivers
------------------

This driver are used by Linux side Proxy/loader as interfaces to access the ivshemem devices in cRTOS.
There are three drivers:
 * uio_ivshmem: For general purpose access to ivshmem devices
 * ivshmem-net: Access the ivshmem device as Ethernet devices.
 * shadow: Access the ivshmem device as interface for remote system call.

### Build

```
cd crtos-drivers
make
```

### Insert

In normal operation only the `shadow` and `ivshmem-net` are needed.
Simple insmod command will do the job of loading these modules.
```
insmod shadow.ko
insmod ivshmem-net.ko
```

The Ethernet device will be enumerated as enp0s13.
Assign IP address `172.16.0.1/24` to it.

Loader
------

The loader is used to load a Linux executable to the sRTOS side.
It also resides as a proxy process for remote system calls after loading.

### Build

```
cd crtos-loader
make
```

Appendix A. Environment preparation
-----------------------------------

### Install Ubuntu 16.04

Download Ubuntu 16.04 from official site and install as normal.

https://releases.ubuntu.com/16.04/

### Install gcc-9

The easiest way is to use the `ubuntu-toolchain-r` PPA.

```sh
sudo add-apt-repository ppa:ubuntu-toolchain-r/test
sudo apt update
sudo apt install gcc-9
```

### Compile Linux kernel 5.4 with PREEMPT_RT

You will need the following.
 * Linux source
 * PREEMPT_RT patch

```sh
# Make a workspace
cd ~
mkdir linux-5.4
cd linux-5.4

# Clone the Linux source
git clone --depth=1 --branch v5.4.39 --single-branch  https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git linux-kernel

# Download the PREEMPT_RT patch
wget http://cdn.kernel.org/pub/linux/kernel/projects/rt/5.4/patch-5.4.39-rt23.patch.xz

# Apply the PREEMPT_RT patch
cd linux
xzcat ../patch-5.4.39-rt23.patch.xz |patch -p
```

We will compile them into dpkgs for easier management in Ubuntu.
```sh
# Build the packages
make -j16 bindeb-pkg

# Install the packages
cd ..
dpkg -i ./*.deb

```

Appendix B. Known problems
-----------------------------------
* In some rare cases, the BIOS/EFI's decision on device mapping might resulting not having such large contiguous hold on physical memory. In that case, consider adding more RAM to your system.

Appendix C. Example of passing an PCI device to RTOS cell
-----------------------------------

For example, to pass a MCS99xx PCI-e to serial adapter on [03:00].
(DO NOT copy-paste into your config! This is just a mere example, the address and index will vary with you system!)

```c
.mem_regions = {
  ...
  /* MemRegion: fb100000-fb100fff : 0000:03:00.2 */
  {
    .phys_start = 0xfb100000,
    .virt_start = 0xfb100000,
    .size = 0x1000,
    .flags = JAILHOUSE_MEM_READ | JAILHOUSE_MEM_WRITE,
  },
  /* MemRegion: fb500000-fb500fff : 0000:03:00.2 */
  {
    .phys_start = 0xfb500000,
    .virt_start = 0xfb500000,
    .size = 0x1000,
    .flags = JAILHOUSE_MEM_READ | JAILHOUSE_MEM_WRITE,
  },
  /* MemRegion: fb501000-fb501fff : 0000:03:00.2 */
  {
    .phys_start = 0xfb501000,
    .virt_start = 0xfb501000,
    .size = 0x1000,
    .flags = JAILHOUSE_MEM_READ | JAILHOUSE_MEM_WRITE,
  },
  /* MemRegion: fb502000-fb502fff : 0000:03:00.1 */
  {
    .phys_start = 0xfb502000,
    .virt_start = 0xfb502000,
    .size = 0x1000,
    .flags = JAILHOUSE_MEM_READ | JAILHOUSE_MEM_WRITE,
  },
  /* MemRegion: fb503000-fb503fff : 0000:03:00.1 */
  {
    .phys_start = 0xfb503000,
    .virt_start = 0xfb503000,
    .size = 0x1000,
    .flags = JAILHOUSE_MEM_READ | JAILHOUSE_MEM_WRITE,
  },
  /* MemRegion: fb504000-fb504fff : 0000:03:00.0 */
  {
    .phys_start = 0xfb504000,
    .virt_start = 0xfb504000,
    .size = 0x1000,
    .flags = JAILHOUSE_MEM_READ | JAILHOUSE_MEM_WRITE,
  },
  /* MemRegion: fb505000-fb505fff : 0000:03:00.0 */
  {
    .phys_start = 0xfb505000,
    .virt_start = 0xfb505000,
    .size = 0x1000,
    .flags = JAILHOUSE_MEM_READ | JAILHOUSE_MEM_WRITE,
  },
},

.pio_regions = {
  ...
  /* Port I/O: d000-d007 : 0000:03:00.2 */
  PIO_RANGE(0xd000, 0x8),
  /* Port I/O: d010-d017 : 0000:03:00.2 */
  PIO_RANGE(0xd010, 0x8),
  /* Port I/O: d020-d027 : 0000:03:00.1 */
  PIO_RANGE(0xd020, 0x8),
  /* Port I/O: d030-d037 : 0000:03:00.0 */
  PIO_RANGE(0xd030, 0x8),
},
.pci_devices = {
  ...
  /* PCIDevice: 03:00.0 */
  {
    .type = JAILHOUSE_PCI_TYPE_DEVICE,
    .iommu = 1,
    .domain = 0x0,
    .bdf = 0x300,
    .bar_mask = {
      0xfffffff8, 0xfffff000, 0x00000000,
      0x00000000, 0x00000000, 0xfffff000,
    },
    .caps_start = 0,
    .num_caps = 5,
    .num_msi_vectors = 8,
    .msi_64bits = 1,
    .msi_maskable = 0,
    .num_msix_vectors = 0,
    .msix_region_size = 0x0,
    .msix_address = 0x0,
  },
  /* PCIDevice: 03:00.1 */
  {
    .type = JAILHOUSE_PCI_TYPE_DEVICE,
    .iommu = 1,
    .domain = 0x0,
    .bdf = 0x301,
    .bar_mask = {
      0xfffffff8, 0xfffff000, 0x00000000,
      0x00000000, 0x00000000, 0xfffff000,
    },
    .caps_start = 5,
    .num_caps = 4,
    .num_msi_vectors = 8,
    .msi_64bits = 1,
    .msi_maskable = 0,
    .num_msix_vectors = 0,
    .msix_region_size = 0x0,
    .msix_address = 0x0,
  },
  /* PCIDevice: 03:00.2 */
  {
    .type = JAILHOUSE_PCI_TYPE_DEVICE,
    .iommu = 1,
    .domain = 0x0,
    .bdf = 0x302,
    .bar_mask = {
      0xfffffff8, 0xfffffff8, 0xfffff000,
      0x00000000, 0x00000000, 0xfffff000,
    },
    .caps_start = 5,
    .num_caps = 4,
    .num_msi_vectors = 1,
    .msi_64bits = 1,
    .msi_maskable = 0,
    .num_msix_vectors = 0,
    .msix_region_size = 0x0,
    .msix_address = 0x0,
  },
},

.pci_caps = {
  /* PCIDevice: 03:00.0 */
  {
    .id = PCI_CAP_ID_MSI,
    .start = 0x50,
    .len = 0xe,
    .flags = JAILHOUSE_PCICAPS_WRITE,
  },
  {
    .id = PCI_CAP_ID_PM,
    .start = 0x78,
    .len = 0x8,
    .flags = JAILHOUSE_PCICAPS_WRITE,
  },
  {
    .id = PCI_CAP_ID_EXP,
    .start = 0x80,
    .len = 0x14,
    .flags = 0,
  },
  {
    .id = PCI_EXT_CAP_ID_VC | JAILHOUSE_PCI_EXT_CAP,
    .start = 0x100,
    .len = 0x10,
    .flags = 0,
  },
  {
    .id = PCI_EXT_CAP_ID_ERR | JAILHOUSE_PCI_EXT_CAP,
    .start = 0x800,
    .len = 0x40,
    .flags = 0,
  },
  /* PCIDevice: 03:00.1 */
  /* PCIDevice: 03:00.2 */
  {
    .id = PCI_CAP_ID_MSI,
    .start = 0x50,
    .len = 0xe,
    .flags = JAILHOUSE_PCICAPS_WRITE,
  },
  {
    .id = PCI_CAP_ID_PM,
    .start = 0x78,
    .len = 0x8,
    .flags = JAILHOUSE_PCICAPS_WRITE,
  },
  {
    .id = PCI_CAP_ID_EXP,
    .start = 0x80,
    .len = 0x14,
    .flags = 0,
  },
  {
    .id = PCI_EXT_CAP_ID_ERR | JAILHOUSE_PCI_EXT_CAP,
    .start = 0x100,
    .len = 0x40,
    .flags = 0,
  },
},
```

Appendix D. Removal of buggy PCI devices
----------------------------------------

Sometimes, you might encounter buggy PCI devices or bridges causing jailhouse to crash.
The solution is to:
 1. Remove it physically.
 2. Disable it using sysfs.

To disable it via sysfs, do the following:
```
# get the PCI bus address of the device (in our case it is an PCI-PCI bridge at 0a:00)
lspci

0a:00.0 PCI bridge: ASMedia Technology Inc. Device 1182

# remove it from the kernel
echo 1 > /sys/bus/pci/devices/0000:0a:00.0/remove
```
