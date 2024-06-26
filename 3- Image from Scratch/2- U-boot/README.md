# U-boot

U-Boot is **the most popular boot loader in** **linux** **based embedded devices**. It is released as open source under the GNU GPLv2 license. It supports a wide range of microprocessors like MIPS, ARM, PPC, Blackfin, AVR32 and x86.

## Setup U-boot

### Download U-Boot

```bash
git clone git@github.com:u-boot/u-boot.git
cd u-boot/
git checkout v2022.07
```

### Configure U-Boot Machine

In this section we will **configure** u-boot for several Machine

```bash
# In order to find the machine supported by U-Boot
ls configs/ | grep [your machine] 
```

#### Vexpress Cortex A9 (Qemu)

In **U-boot directory** Assign this value

```bash
# Set the Cross Compiler into the environment
# To be used by the u-boot
export CROSS_COMPILE=<Path To the Compiler>/arm-cortexa9_neon-linux-musleabihf-
export ARCH=arm

# load the default configuration of ARM Vexpress Cortex A9
make vexpress_ca9x4_defconfig
```

#### Raspberry Pi

First we need to download the **compiler** for the architecture set for **RPI**

```bash
# update the package manager
sudo apt-get update

# the following compiler support 32 bit architecture
sudo apt-get install gcc-arm-linux-gnueabihf-

# in case of using architecture 64 Download the following compiler
sudo apt-get install gcc-aarch64-linux-gnu
```

##### Raspberry Pi 3

In **U-boot directory** Assign this value

```bash
# Set the Cross Compiler into the environment
# To be used by the u-boot
export CROSS_COMPILE=arm-linux-gnueabihf-
export ARCH=arm

# load the default configuration of rpi 3
make rpi_3_defconfig
```

##### Raspberry Pi 4

In **U-boot directory** Assign this value

```bash
# Set the Cross Compiler into the environment
# To be used by the u-boot
export CROSS_COMPILE=arm-linux-gnueabihf-
export ARCH=arm

# load the default configuration of rpi 3
make rpi_4_32b_defconfig
```

**In case of 64 architecture**

```bash
export CROSS_COMPILE=aarch64-linux-gnu-
export ARCH=arm64

# depends what raspberry pi in used
make rpi_4_b_defconfig
```

#### Beaglebone

In **U-boot directory** Assign this value

```bash
# Set the Cross Compiler into the environment
# To be used by the u-boot
export CROSS_COMPILE=<Path To the Compiler>/arm-cortexa9_neon-linux-musleabihf-
export ARCH=arm

# load the default configuration of ARM Vexpress Cortex A9
make am335x_evm_defconfig
```

### Configure U-Boot 

In this part we need to configure some u-boot configuration for the specific board chosen up.

```bash
make menuconfig
```

**The customer requirement are like following**:

- [ ] Support **editenv**.
- [ ] Support **bootd**.
- [ ] Store the environment variable inside file call **uboot.env**.
- [ ] Unset support of **Flash**
- [ ] Support **FAT file system**
  - [ ] Configure the FAT interface to **mmc**
  - [ ] Configure the partition where the fat is store to **0:1**

## SD Card

In this section it's required to have SD card with first partition to be FAT as pre configured in **U-boot Menuconfig**.

### ! ONLY FOR WHO WORK WITH QEMU !

In order to Emulate SD card to attached to Qemu following steps will be followed: (ONLY FOR NON PHYSICAL SD):

```bash
# Change directory to the directory before U-Boot
cd ..

# Create a file with 1 GB filled with zeros
dd if=/dev/zero of=sd.img bs=1M count=1024
```

### ! ONLY FOR WHO WORK WITH HW !

Plug your SD card to the computer

```bash
# To show where your sd card is mounted
lsblk 
```

`WARNING: The following command will completely erase all data on the specified SD card.`

```bash
### WARNING ####
sudo dd if=/dev/zero of=/dev/mmblk<x> bs=1M

# is not always mmblck to make sure use the command lsblk

# Assign the Block device as global variable to be treated as MACRO
export DISK=/dev/mmblck<x>
```



### Configure the Partition Table for the SD card

```bash
# for the VIRTUAL SD card
cfdisk sd.img

# for Physical SD card
cfdisk /dev/mmblck<x>
```

| Partition Size | Partition Format | Bootable  |
| :------------: | :--------------: | :-------: |
|    `200 MB`    |     `FAT 16`     | ***Yes*** |
|    `300 MB`    |     `Linux`      | ***No***  |

**write** to apply changes

### Loop Driver FOR Virtual usage ONLY

To emulate the sd.img file as a sd card we need to attach it to **loop driver** to be as a **block storage**

```bash
# attach the sd.img to be treated as block storage
sudo losetup -f --show --partscan sd.img

# Running the upper command will show you
# Which loop device the sd.img is connected
# take it and assign it like down bellow command

# Assign the Block device as global variable to be treated as MACRO
export DISK=/dev/loop<x>
```

### Format Partition Table

As pre configured from **cfdisk command** first partition is **FAT**

```bash
# Formating the first partition as FAT
sudo mkfs.vfat -F 16 -n boot ${DISK}p1
```

 As pre configured from cfdisk Command second partition is linux

```bash
# format the created partition by ext4
sudo mkfs.ext4 -L rootfs ${DISK}p2
```

## Test U-Boot

Check the **u-boot** and the **sd card** are working

### Vexpress-a9 (QEMU)

Start Qemu with the **Emulated SD** card

```bash
qemu-system-arm -M vexpress-a9 -m 128M -nographic \
-kernel u-boot/u-boot \
-sd sd.img
```

### Raspberry Pi

Add all **close source** file provided my Raspberry depend for each version to the **SD card in FAT Partition**

```
1. Bootcode.bin 
2. Fixup.dat
3. cmdline.txt
4. config.txt
5. start.elf
6. u-boot.bin {generated from u-boot}
```

Edit file `config.txt` as following

```
kernel=u-boot.bin
enable_uart=1
```

### Beaglebone

Add following file generated by U-Boot in the SD card FAT partition

```
1. MLO
2. u-boot.bin
```

## Initialize TFTP protocol

### Ubuntu

```bash
#Switch to root
sudo su
#Make sure you are connected to internet
ping google.com
#Download tftp protocol
sudo apt-get install tftpd-hpa
#Check the tftp ip address
ip addr `will be needed`
#Change the configuration of tftp
nano /etc/default/tftpd-hpa
	#write inside the file
    tftf_option = “--secure –-create”
#Restart the protocal
Systemctl restart tftpd-hpa
#Make sure the tftp protocol is running
Systemctl status tftpd-hpa
#Change the file owner
cd /srv
chown tftp:tftp tftp 
#Move your image or file to the server
cp [File name] /srv/tftp
```

### Create Virtual Ethernet For QEMU

This section for Qemu emulator users only **no need for who using HW**

Create a script `qemu-ifup` 

```bash
#!/bin/sh
ip a add 192.168.0.1/24 dev $1
ip link set $1 up
```

#### Start Qemu

In order to start Qemu with the new virtual ethernet

```bash
sudo qemu-system-arm -M vexpress-a9 -m 128M -nographic \
-kernel u-boot/u-boot \
-sd sd.img \
-net tap,script=./qemu-ifup -net nic
```

## Setup U-Boot IP address

```bash
#Apply ip address for embedded device
setenv ipaddr [chose]
#Set the server ip address that we get from previous slide
setenv serverip [host ip address]

#### WARNING ####
#the ip address should has the same net mask

```

## Load File to RAM

First we need to know the ram address by running the following commend

```bash
# this commend will show all the board information and it start ram address
bdinfo
```

### Load from FAT

```bash
# addressRam is a variable knowen from bdinfo commend
fatload mmc 0:1 [addressRam] [fileName]
```

### Load from TFTP

```bash
# addressRam is a variable knowen from bdinfo commend
tftp [addressRam] [fileName]
```
## U-Boot Scripting
You can create the script by using any text editor *(vim)* to write the commands and configurations desired, and then create a **uboot script image** file using **`mkimage tool`**

### Create uboot-script 
Create the `uboot-script` with the desired configurations and environment variables needed ... with a final extension of *.txt*
``` bash
setenv bootargs "console=ttyAMA0,38400n8 root=/dev/mmcblk0p2 rootfstype=ext4 rw rootwait init=/sbin/init"

setenv imageFat "if mmc dev; then echo Device is working ...; else echo Device is not working!; fi"

setenv load_fromFat "fatload mmc 0:1 $kernel_addr_r zImage; fatload mmc 0:1 $fdt_addr_r vexpress-v2p-ca9.dtb; bootz $kernel_addr_r - $fdt_addr_r"

setenv load_fromTFTP "tftp $kernel_addr_r zImage; tftp $fdt_addr_r vexpress-v2p-ca9.dtb; bootz $kernel_addr_r - $fdt_addr_r"

setenv ipaddr 192.168.0.100
setenv serverip 192.168.0.1

setenv bootcmd "if run imageFat; then run load_fromFat; else run load_fromTFTP; fi"

bootz $kernel_addr_r - $fdt_addr_r"
```

### Convert To Uboot Image
``` bash
# File name is uboot_script.txt 
mkimage -T script -C none -n "uboot_script" -d uboot_script.txt uboot_script.img
```
- **-T** --> File Type
- **-C** --> Compression Type
- **-n** --> Image Name
- **-d** --> Input File then Output file

### Copy Image To Boot Partition
``` bash
# From: uboot_script.img location
# To: SD_Card boot partition
sudo cp .../uboot_script.img /media/ibrahim/boot
```

### Load Script Into Uboot
We have to set an `environment variable` to a specific **DRAM address**, to store the *yet to be loaded* `uboot_script`
``` bash
setenv uboot_scriptaddr 0x60000020

setenv bootcmd fatload mmc 0:1 $uboot_scriptaddr uboot_script.img; source $uboot_scriptaddr

# If the command is working without any syntax errors ... don't forget to saveenv ... to save the uboot_scriptaddr step and bootcmd step
```
> :white_check_mark: **`run bootcmd`**, If the image is loaded successfully .. you should see the image containing the uboot script written, being loaded *(with the commands and configurations written being loaded)* ... and since we added the `bootz` command ... the system would boot successfully!
