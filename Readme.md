# Requirements:

An Arch Linux, Debian Linux or Ubuntu installation. This can be a virtual machine that supports USB passthrough (e.g. using Oracle VirtualBox).
A USB thumb drive or SATA disk with a capacity of at least 4 GB.
In the next examples, replace /dev/sdX with the device name to be prepared as an installation medium for the ix2-200. Replace it with the correct device (which can be found by running `dmesg` after connecting a USB thumb drive).

SATA disks must be initialized before they are usable. Any existing RAID superblocks have to be removed. First, run `cat /proc/mdstat` to identify and separate any Linux software RAID arrays from the disk intended to be an installation medium from any RAID arrays of the used system itself. Run the following commands only on RAID arrays associated with the disk to be prepared (replace /dev/mdN with the relevant array):

```
mdadm --stop /dev/mdN
mdadm --zero-superblock /dev/sdb*
```
If it is not possible to stop and remove some of the superblocks, run:

```
dd if=/dev/zero of=/dev/sdb bs=1M count=16
```
Reboot the system used to prepare the installation disk and any old RAID arrays should not be detected anymore.

Use fdisk to partition the disk or USB thumb drive:
```
fdisk /dev/sdX
Use o to create a clean DOS partition table.
Use n to create a new partition, type: primary, number: 1, first sector: default, last sector: +256M.
Use n to create a new partition, type: primary, number: 2, first sector: default, last sector: +8G (or the rest of the remaining free disk space if smaller).
Use p to see the new partition table.
If ID and Type are not ‘83 Linux’ for both partitions: Use t to change a partition’s type. Type must be 83.
Use w to write the changes to disk.
```
Create and mount the file systems, and download and extract Arch Linux ARM using the following commands. If bsdtar is unavailable, tar can be used instead with the same parameters. Warnings about ignored unknown extended header keywords can be ignored.
```
mkfs.ext2 /dev/sdX1
mkfs.ext4 /dev/sdX2
e2label /dev/sdX1 BOOT
e2label /dev/sdX2 ROOT

mount /dev/sdX2 /mnt
mkdir /mnt/boot
mount /dev/sdX1 /mnt/boot

cd /mnt
curl -JLO http://os.archlinuxarm.org/os/ArchLinuxARM-kirkwood-latest.tar.gz
bsdtar -xpf ArchLinuxARM-kirkwood-latest.tar.gz
rm ArchLinuxARM-kirkwood-latest.tar.gz
```
Now a U-Boot image of the initramfs has to be created. This makes it readable by the boot loader of the ix2-200.

```
apt-get install u-boot-tools # If Debian or Ubuntu is used to create the installation medium
pacman -S --noconfirm --needed uboot-tools # If Arch Linux is used to create the installation medium
mkimage -A arm -O linux -T ramdisk -C gzip -a 0x00000000 -e 0x00000000 -n "Arch Linux ARM initrd" -d /mnt/boot/initramfs-linux.img /mnt/boot/uInitrd

```
Network support has to be enabled on the installation medium. eth0 is an internal network card that is not used by the ix2-200. eth1 is the actual Gigabit Ethernet adapter wired to the network port on the rear of the unit. eth1 must be enabled and eth0 must be disabled.

```
sed 's/eth0/eth1/g' /mnt/etc/systemd/network/eth0.network > /mnt/etc/systemd/network/eth1.network
rm /mnt/etc/systemd/network/eth0.network
```
DHCP will be used to assign an IP address to eth1. /mnt/etc/systemd/network/eth1.network can be edited to assign a static IP address instead. An example configuration:

```
[Match]
Name=eth1

[Network]
DHCP=no
DNSSEC=no
Address=192.168.1.2/24
Gateway=192.168.1.1
DNS=192.168.1.1
```

Run when using a SATA disk for the installation:

```
cat << 'EOF' >> /mnt/etc/fstab
/dev/sdb1  /boot  ext2  defaults,noatime  0  0
EOF
```
Run when using a USB thumb drive for the installation:

```
cat << 'EOF' >> /mnt/etc/fstab
/dev/sdc1  /boot  ext2  defaults,noatime  0  0
EOF
```
Preparation of the installation medium is now complete. Run the following commands to unmount the installation medium safely:

```
cd /
sync
umount -R /mnt
```

Disconnect the prepared installation medium.

# Step 2: ix2-200 preparation

The requirements for this method are:

A Phillips #1 screwdriver.
A USB to TTL serial adapter with support for +3.3V. Recommended serial adapters include the CP2102 and any FTDI-based adapters. Avoid adapters made by Prolific, since their drivers are crippled on purpose. Expect BSODs with Prolific adapters when running certain terminal software (e.g. PuTTY) or when a counterfeit chip is used. Note that it is impossible for a consumer to know whether a chip is counterfeit or not. Tested software includes PuTTY and Tera Term.
Access to the main board of the ix2-200 is required. Follow this procedure to remove the main board:

Remove the 4 large screws from both HDD bays on the bottom of the case.
Remove the HDD bays from the case.
Remove the 4 small screws on the rear of the case.
Remove the plastic bracket from the rear of the case.
Remove the 2 small screws under the front side of the two front rubber feet of the case.
Remove the main board assembly by sliding it from the rear to the front of the case.
The ix2-200’s serial connector

Locate the serial connector on the mainboard. The connector has the label JP1. Connect the serial adapter to this interface using the following pinout (JP1 = pin 1):

Pin 1: Do NOT connect. This pin provides +3.3V and is not required for USB UART serial adapters.
Pin 2: TxD. Connect to RxD of the serial adapter.
Pin 3: GND. Connect to GND of the serial adapter.
Pin 4: RxD. Connect to TxD of the serial adapter.
Open in the terminal software the relevant serial port using the following parameters:

Baud rate / speed: 115200
Data bits: 8
Parity: None
Stop bits: 1
Flow control: XON/XOFF
Power on the ix2-200 (without any installed disks) by connecting the power adapter while monitoring the terminal software. Press enter at the autoboot prompt to abort the boot process and to remain in the boot loader.

In the shell of U-Boot, run:
```
printenv
```

Copy all output text to a file on the SSH client named “nandconfig.txt”. This is a backup of the ix2-200’s original NAND configuration. Keep it safe in case you ever need to recover it.

Now run the following commands to configure the ix2-200. These commands will allow the ix2-200 to boot Arch Linux ARM from the installation medium and from the disks after it has been installed. Do NOT copy/paste all commands since this will result in incomplete executed commands due to the lack of flow control on the serial interface in U-Boot. A copy/paste of each line separately should be OK. In Tera Term, it is also possible to copy/paste all commands if the paste delay per line is configured as 250 ms.

```
setenv ethaddr 'AA:00:00:00:00:01'
setenv mainlineLinux 'yes'
setenv arcNumber '1682'

setenv bootargs_console 'console=ttyS0,115200'
setenv bootargs_mtdparts 'mtdparts=orion_nand:640k(u-boot)ro,16k(u-boot-env),-(iomega-firmware)ro'
setenv bootargs_root 'root=LABEL=ROOT rw'

setenv memoffset_kernel '0x01000000'
setenv memoffset_initrd '0x08004000'

setenv bootargs_combine 'setenv bootargs ${bootargs_console} ${bootargs_mtdparts} ${bootargs_root}'
setenv bootlinux 'bootm ${memoffset_kernel} ${memoffset_initrd}'

setenv usb_load_firstdevice 'ext2load usb 0:1 ${memoffset_kernel} /uImage; ext2load usb 0:1 ${memoffset_initrd} /uInitrd'
setenv usb_load 'run bootargs_combine; usb reset; run usb_load_firstdevice; run bootlinux'

setenv sata_load_disk1 'ext2load ide 0:1 ${memoffset_kernel} /uImage; ext2load ide 0:1 ${memoffset_initrd} /uInitrd'
setenv sata_load_disk2 'ext2load ide 1:1 ${memoffset_kernel} /uImage; ext2load ide 1:1 ${memoffset_initrd} /uInitrd'
setenv sata_load 'run bootargs_combine; ide reset; run sata_load_disk1; run sata_load_disk2; run bootlinux'

setenv bootcmd 'run usb_load; run sata_load'
saveenv
```

Now power down and re-assemble the ix2-200. Continue with step 3.

# Step 3: Install Arch Linux ARM using the prepared installation medium

If in step 1 a USB thumb drive was prepared, install two disks with similar capacities in the HDD bays (if not already installed) and connect the thumb drive to any of the USB ports of the ix2-200.

If in step 1 a SATA disk was prepared as an installation medium, place the installation medium in HDD bay 2. Place another SATA disk with a similar capacity in HDD bay 1.

Now power up the ix2-200. It should boot from the prepared installation medium. Monitor your DHCP server on any newly assigned leases if DHCP was configured, or monitor the static IP that you configured manually. Once it has booted up, establish an SSH connection. Use alarm as username and alarm as password. Once logged in, use su root to get root access using the password root.

Start by loading some kernel modules. These have to be loaded now to avoid any reboots during the installation process. Run the following command:
```
modprobe raid1
```
Note that by loading the ‘raid1’ module, other required modules will automatically be loaded.

Arch Linux ARM needs to be fully up-to-date before new packages can be installed. See the Arch Linux Wiki for more information. Run the following commands to update Arch Linux ARM, and to install some additional packages that are needed to complete the installation:

```
pacman-key --init && pacman-key --populate archlinuxarm
pacman -Syu --noconfirm uboot-tools mdadm gptfdisk parted rsync arch-install-scripts
```

Arch Linux ARM will be installed to the disk in HDD bay 1 (/dev/sda). After the installation a RAID1 array will be constructed. Start by destroying any old RAID superblocks if one or both of the disks were used previously. First, see if any such blocks are available:

```
mdadm --assemble --scan
ls -al /dev/md*
```
If they are, they should be removed. Note that you should only run the –zero-superblock command on /dev/sdb? if you use a USB thumb drive to prepare the ix2-200.

Run the following commands:

```
mdadm --stop /dev/md*
mdadm --zero-superblock /dev/sda?
mdadm --zero-superblock /dev/sdb?
```
Now partition the disk in HDD bay 1. Run the following command:

```
gdisk /dev/sda
```
Use o to create a clean GPT partition table Use n to add partition 1, default first sector, +256M size, type hex code FD00 (Linux RAID). Use n to add partition 2, default first sector, +32G size (or how large you want your system partition to be), type hex code FD00 (Linux RAID). Use n to add partition 3, default first sector, default size (which will use most of the remaining disk space), type hex code FD00 (Linux RAID). Use r to enter recovery/transformation mode. Use p to see the current GPT partition table. Use h to create a hybrid MBR. Only add GPT partition 1 (which will be /boot, ext2) to the hybrid MBR. Do not allow the EFI GPT partition entry to be placed first in the MBR (n). Use MBR hex code 83 for the partition type in the MBR. Bootable flag does not have to be set (n). Do not use the unused partition space(s) to protect partitions (n). Use o at the recovery/transformation command prompt to see the new hybrid MBR. Use w and y to write all changes to disk.

Finally, run partprobe to enforce the kernel to use the new partition table layout without a reboot.

Create and initialize an incomplete RAID1 array. Run the following commands:

```
mdadm --create /dev/md0 --level=1 --raid-devices=2 --metadata=0.90 --run /dev/sda1 missing
mdadm --create /dev/md1 --level=1 --raid-devices=2 --run /dev/sda2 missing
mdadm --create /dev/md2 --level=1 --raid-devices=2 --run /dev/sda3 missing

mkfs.ext2 /dev/md0
mkfs -t ext4 -E lazy_itable_init=0,lazy_journal_init=0 /dev/md1
mkfs -t ext4 -E lazy_itable_init=0,lazy_journal_init=0 /dev/md2

e2label /dev/md0 BOOT
e2label /dev/md1 ROOT
e2label /dev/md2 DATA
```
The way the ext4 partitions are formatted will take some time (~10 minutes for a 2 TB disk), but it will save disk I/O (and RAID1 build time, due to reducing random read access) after the installation. See this page for more information.

Now mount the prepared disk and copy the contents of the installation medium to it:

```
mount /dev/md1 /mnt
mkdir /mnt/boot
mount /dev/md0 /mnt/boot
rsync -aAXq --exclude={"/dev/*","/proc/*","/sys/*","/tmp/*","/run/*","/mnt/*","/lost+found"} / /mnt
```
Now chroot into the new installation:
```
arch-chroot /mnt
```
It is required to switch to a kernel line that supports device trees to support the buttons and LEDs on the front of the ix2-200, and to be able to power down the system in a safe manner. A warning will be shown about that the system will not be able to boot without user intervention when the pacman command is executed. This warning can be ignored, since the required user intervention is in the commands that follow. Run the following commands:

```
pacman -S --noconfirm --ask 4 linux-kirkwood-dt
cat /boot/zImage /boot/dtbs/kirkwood-iomega_ix2_200.dtb > /boot/zImage-dtb
mkimage -A arm -O linux -T kernel -C none -a 0x02000000 -e 0x02000000 -n "Arch Linux ARM kernel" -d /boot/zImage-dtb /boot/uImage
```
Add mdadm_udev to HOOKS in /etc/mkinitcpio.conf. This allows the boot process of the new system to be aware of the system RAID configuration. An example of the modified line:

```
HOOKS="base udev autodetect modconf block filesystems keyboard fsck mdadm_dev"
```
Publish information about the RAID1 array for automated assembly at boot. Run:
```
cat << 'EOF' >> /etc/mdadm.conf
ARRAY /dev/md0 devices=/dev/sda1,/dev/sdb1
ARRAY /dev/md1 devices=/dev/sda2,/dev/sdb2
ARRAY /dev/md2 devices=/dev/sda3,/dev/sdb3
EOF
```
To make the new installation aware of its filesystems running on the RAID array, update /etc/fstab by running:

```
sed -iE '/^\/dev\/sd.1/d' /etc/fstab
cat << 'EOF' >> /etc/fstab
/dev/md0  /boot        ext2  defaults,noatime  0  0
/dev/md1  /            ext4  defaults,noatime,barrier=1  0  0
/dev/md2  /mnt/r1data  ext4  defaults,noatime,barrier=1  0  0
EOF
```
Create the directory for the large data partition, in which it can be mounted after boot:

```
mkdir /mnt/r1data
```
Now it is time to create an initramfs, which embeds /etc/mdadm.conf and /etc/fstab to notify the kernel at boot that the system uses a RAID1 array. Run the following commands:

```
mkinitcpio -p linux-kirkwood-dt
mkimage -A arm -O linux -T ramdisk -C gzip -a 0x00000000 -e 0x00000000 -n "Arch Linux ARM initrd" -d /boot/initramfs-linux.img /boot/uInitrd
```
Access to U-Boot’s configuration parameters can be enabled in Linux in case changes are required at a later time. Do this by running:
```
cat << 'EOF' >> /etc/fw_env.config
/dev/mtd1  0x0000  0x4000  0x4000
EOF
```
Now cleanly exit the chroot environment, and unmount the drive on which Arch Linux ARM is installed:

```
exit
sync
umount -R /mnt
```
Now power down the ix2-200 by running shutdown now and by unplugging the power after the disks are switched off. The installation is complete. If you used a USB thumb drive, remove it now. The USB thumb drive can be used as a recovery volume in the future, in case the ix2-200 is unable to boot.

If you used a SATA disk as the installation medium, leave it connected to be used as a second disk for the RAID1 set after the next boot. If you want to use another disk as the second disk, this is the time to replace it. The ix2-200 is configured to boot from the disk in HDD bay 1 before trying the disk in HDD bay 2. This was the reason why the earlier instructions noted that the installation medium SATA disk should be in HDD bay 2 for this step.

# Step 4: Make the RAID1 array complete

Power up the ix2-200, establish an SSH connection, and gain root access using su root. The ix2-200 will have the same IP that was used during the installation, unless you used DHCP and have a jumpy DHCP configuration. In that case, monitor the assigned leases on your DHCP server.

Make the RAID1 array complete by preparing and adding the second disk.

Copy GPT partition layout from the first disk to the second disk
```
sgdisk -b /root/raid1.gpt /dev/sda
sgdisk --zap-all /dev/sdb
sgdisk -l /root/raid1.gpt /dev/sdb
partprobe
```

Add the second disk to the existing RAID1 array
```
mdadm /dev/md0 -a /dev/sdb1
mdadm /dev/md1 -a /dev/sdb2
mdadm /dev/md2 -a /dev/sdb3
```
Verify that rebuild of the RAID1 array has started.
Also see: https://raid.wiki.kernel.org/index.php/Mdstat
```
watch cat /proc/mdstat
```
Further configuration and use of the system, and reboots will not disrupt the rebuild process. Expect the rebuild to take up more or less one hour for every 400 GB of RAID1 array capacity.

# Step 5: Automate kernel and initramfs management

Arch Linux ARM can update the kernel when new software is installed through its package management system. Several actions must be performed every time this happens to allow the boot loader to load the new boot files:

The new kernel must be converted to uImage.
The newly generated initrd must be converted to uInitrd.
Create the pacman hook and a script it runs that will generate new U-Boot images, anytime the Linux kernel is updated:

```
mkdir -p /etc/pacman.d/hooks
cat << 'EOF' > /etc/pacman.d/hooks/generate-ubootfiles.hook
[Trigger]
Operation = Install
Operation = Upgrade
Type = Package
Target = linux-kirkwood-dt

[Action]
Description = Make the new kernel and initrd bootable by U-Boot
When = PostTransaction
Exec = /usr/local/bin/generate-ubootfiles
Depends = uboot-tools
EOF

cat << 'EOF' > /usr/local/bin/generate-ubootfiles
#!/bin/sh
#Creates U-Boot images from the boot files

zImage=/boot/zImage
dtb=/boot/dtbs/kirkwood-iomega_ix2_200.dtb
zImagedtb=/boot/zImage-dtb
uImage=/boot/uImage
uImageAddr=0x02000000

initrd=/boot/initramfs-linux.img
uInitrd=/boot/uInitrd
uInitAddr=0x00000000

if [ ! -e $zImage -o ! -e $initrd ]; then 
  echo "distribute-bootfiles: Error! $zImage and/or $initrd is missing."
  echo "distribute-bootfiles: New U-Boot files will NOT be created!"
  exit 1
fi

#Create uImage
cat $zImage $dtb > $zImagedtb
mkimage -A arm -O linux -T kernel -C none -a $uImageAddr -e $uImageAddr -n "Arch Linux ARM kernel" -d $zImagedtb $uImage > /dev/null

#Create uInitrd
mkimage -A arm -O linux -T ramdisk -C gzip -a $uInitAddr -e $uInitAddr -n "Arch Linux ARM initrd" -d $initrd $uInitrd > /dev/null
EOF

chmod +x /usr/local/bin/generate-ubootfiles
```
Test whether the script works by running /usr/local/bin/generate-ubootfiles. Everything works OK if there is no output.

Note that this only automates creation of the necessary U-Boot files when the kernel package is updated through pacman. The update script must be run manually if something else updated either the kernel or the initramfs (including the user, by running mkinitcpio -P, for example).

This concludes the ix2-200 specific installation steps. You can finalize the Arch Linux ARM installation (e.g. configuring the time zone) by consulting the Arch Linux Wiki.


# (OPTIONAL): Install Samba and add a share

This is a very brief example implementation.

Install Samba using the following command:

```
pacman -S --noconfirm samba
```
Create /etc/samba/smb.conf. A minimalistic example with ACL support:
```
[gobal]
   workgroup = WORKGROUP
   server string = ix2-200 Samba Server

   log file = /var/log/samba/%m.log
   max log size = 50

   security = user

   vfs objects = acl_xattr
   map acl inherit = yes
   store dos attributes = yes
```
See /etc/samba/smb.conf.default for more configuration options.

Enable and start the file server by running the following commands:
```
systemctl enable smb
systemctl start smb
```
To add an SMB user account named ‘user’, run the following commands:

```
useradd -c "Test User" -M -d / -s /usr/bin/nologin user
smbpasswd -a user
```
To create a directory to share with ‘user’, run the following commands:

```
mkdir /mnt/r1data/usershare$
chown -R user:user /mnt/r1data/usershare$
```
To share the created directory using SMB with ‘user’, edit /etc/samba/smb.conf. Add the following new lines to the bottom of the file:

```
[usershare$]
   comment = Some user's share
   path = /mnt/r1data/usershare$
   valid users = user
   writable = yes
```

Changes to /etc/samba/smb.conf should be processed automatically. systemctl reload smbd forces Samba to reload its configuration.

# Appendix A: Replace a failed RAID disk

Remember that HDD bay 1 is /dev/sda and HDD bay 2 is /dev/sdb. ‘/dev/sdX’ is used as a placeholder in the commands for the disk that will be replaced and the replacement disk.

If the disk is still accessible, it must be marked as faulty. This prevents data corruption at boot if the old disk is accidentally placed back after its removal. Run the following commands:
```
mdadm --fail /dev/md0 /dev/sdX1
mdadm --fail /dev/md1 /dev/sdX2
mdadm --fail /dev/md2 /dev/sdX3
mdadm --zero-superblock /dev/sdX*
sgdisk --zap-all /dev/sdX
```
Remove the faulty disk and add a new disk with a similar capacity. The ix2-200 supports hotswap, so it is not necessary to power down the system.

After swapping in a new disk, run:
```
sgdisk --zap-all /dev/sdX
sgdisk -l /root/raid1.gpt /dev/sdX
mdadm /dev/md0 -a /dev/sdX1
mdadm /dev/md1 -a /dev/sdX2
mdadm /dev/md2 -a /dev/sdX3
```
Verify that rebuild of the RAID1 array has started by running:
```
cat /proc/mdstat
```
# Appendix B: Unbrick an ix2-200

The stock ix2-200 boot order is as follows, starting from power on:

The Kirkwood SoC executes the embedded ROM code. Connected pins determine the boot order. The boot order is UART first, SPI second.
The SPI flash memory (also referred to as NAND flash memory) is accessed. Das U-Boot is loaded and executed.
Das U-Boot reads the Linux kernel and initramfs from SPI flash memory into RAM, and boots the Linux kernel.
The Linux kernel reads its initramfs from RAM, and mounts partitions on the disks containing the ix2-200’s Iomega software. From here, the operating system is started. If the reset button is pushed during boot up, the kernel provided by Iomega will use an ‘EMC Imager’ (a connected USB drive or a network server) to re-initialize the disks, which is the procedure described in Appendix C.
If Das U-Boot, the kernel or initramfs are not available in SPI for whatever reason (such as a bad flash), the system will be unable to boot. There are several ways to unbrick an ix2-200.

The easiest method would be to use the embedded ROM code to boot U-Boot using UART, and from there reflash the SPI flash memory using TFTP. Unfortunately, version 1.11 of the Kirkwood ROM has issues with UART booting, which is the version the ix2-200 has. Some users claim that they were able to use the Kirkwood-specific kwboot tool to boot over UART, but this did not work for me on two different ix2-200 devices.

Flashing U-Boot directly to the SPI flash chip on the board is possible. However, it can be a bit cumbersome since specialized tools are required due to the small size of the chip..

An approachable (and affordable) alternative is using the ix2-200’s ARM JTAG 20 connector. This is a standard debug connector available on many ARM devices. In the case of the ix-200, this connector allows full control of the device’s CPU and RAM. This connector should be able to interface directly with SPI flash memory as well, but my ix-200 stubbornly refused this. However, a workaround using U-Boot is possible.

Since I did not have a JTAG adapter available, I had to improvise. Booting and restoring a bricked ix2-200 is a staged process:

Physically connect all relevant devices.
Prepare the Raspberry Pi to logically interface with the ix2-200.
Load U-Boot to the ix2-200’s RAM and boot it using JTAG.
Give commands to U-Boot using UART (serial connection).
Command U-boot to load each required image (U-Boot itself, the Linux kernel, and initramfs) from TFTP to RAM, and from there write it to SPI flash.
The following is needed:

A Phillips #1 screwdriver.
A Raspberry Pi with a 26/40 pin GPIO header and an Ethernet adapter. 1 GB of RAM is recommended. I used a Pi 3 Model B. The Pi 4 is untested, which is why the provided scripts do not support it. All older Pi generations should work. Of course, the used Pi must be bootable, and I recommend to use a microSD card for simplicity. Interfacing with the Pi can be done using a HDMI monitor and USB keyboard, or using SSH.
Arch Linux ARM installed on the Pi, which was the only tested distribution. These instructions can be adjusted for other Pi distributions, but you should have a fair bit of Linux knowledge. This guides assumes that a fresh Arch Linux ARM image is used.
6x female-male jumper wires. The female side must fit the Pi’s GPIO header.
3x female-female jumper wires. These must fit the Pi’s GPIO header.
An UTP network cable to connect the ix2-200 to the same network as the Raspberry Pi.
The ix2-200 U-Boot recovery files, provided in this archive.
The ix2-200 Raspberry Pi JTAG scripts in this archive.
Start by disassembling the ix2-200.

Remove the 4 large screws from both HDD bays on the bottom of the case.
Remove the HDD bays from the case.
Remove the 4 small screws on the rear of the case.
Remove the plastic bracket from the rear of the case.
Remove the 2 small screws under the front side of the two front rubber feet of the case.
Remove the main board assembly by sliding it from the rear to the front of the case.
Disconnect the fan connector from the main board.
Remove the 4 small screws which secure the main board to the main board assembly. Note that the angled PCB containing the SATA connectors is also part of the mainboard.
Gently slide the main board away from the assembly. First slide it backwards, then out. Remember this, because this step has to be reversed when reassembling the device.
Turn the main board upside down. The result should be this:

The ix2-200’s main board, placed upside down

The JTAG connector is on the bottom, labeled J8. Note that your main board might have a small piece of transparent tape on the other side, partially over the GPIO connector. This is to protect the main board from making direct contact with one of the assembly’s HDD tray supports. Remove this tape. It can be reattached/replaced later, or removed entirely as long as you are careful during assembly and placing hard drives. There should always be some space between the main board and the support.

The GPIO connector is not populated, but open. The male part of the jumper wires will be pushed through the holes. Alternatively, you can solder a header. Make sure the header is on the same side as the serial connector, and that on the rear nothing sticks out too much (or use electrical tape), to prevent contact with the HDD tray support.

The jumper wires I used. Note the colors, which are used in the pinout: Jumper wires

This is the pinout for connecting a Raspberry Pi to an ix2-200 main board for JTAG and serial communication: Pi - ix2-200 JTAG and UART connection pinout

Note that each connector in the layout has its label drawn next to it. The position and orientation of the labels are the same as on their respective boards. This can be used to orientate the connectors. Only connect the pins as shown, or you risk magical blue smoke coming from your Pi and ix2-200’s main board.

The Pi connected to the ix2-200 should look as follows: Jumper wires

Now connect both devices to the same network, which should have an internet connection. Finally, apply power to the Pi.

Run the following on the Pi as root to update a fresh Arch Linux ARM installation:

```
wifi-menu # Only needed if a wireless network connection is required
pacman-key --init && pacman-key --populate
pacman -Syu --needed --noconfirm base-devel minicom inetutils tftp-hpa
```
Prepare the TFTP server as root:
```
mkdir -p /srv/tftp

curl -JLO --output-dir /srv/tftp https://kiljan.org/downloads/ix2-200/uboot.2009-09-03
curl -JLO --output-dir /srv/tftp https://kiljan.org/downloads/ix2-200/uboot.2009-09-08

curl -JLO --output-dir /tmp https://kiljan.org/downloads/ix2-200/ix2-boot.tgz
tar -C /srv/tftp -xf /tmp/ix2-boot.tgz zImage initrd
systemctl start tftpd
```
Now run as a normal user (which has rights to use sudo):
```
mkdir -p ~/src/openocd
cd ~/src/openocd
curl -JLO https://github.com/archlinux/svntogit-community/raw/packages/openocd/trunk/PKGBUILD
```
Edit PKGBUILD.

Add bcm2835gpio to the _features variable.
Add armv7h or aarch64 if you are using a 64-bit installation to the _features variable. When in doubt, use uname -m to see your architecture, or simply add both values.
Now run as the same user:

Reassembly of the ix2-200 can be done by following the disassembly steps in reverse. For attaching the main board back to the main board assembly, I advise to start with first putting it in place using the two screws of the SATA PCB, follow with the outer screw of the main PCB, and finish with the inner screw of the main PCB. This makes it easier to align the main board with the assembly.

# Appendix C: Restore the original ix2-200 firmware

You will need this ix2-200 recovery image, which is based on the last official Iomega ix2-200 firmware. Place ‘ix2-boot.tgz’ on a FAT32 formatted USB thumb drive in these directories:

```
/emctools/ix2-200_images
/emctools/ix2-200d_images
```
Now run in Arch Linux ARM the following commands:
```
fw_setenv mainlineLinux
fw_setenv arcNumber
fw_setenv bootcmd 'run flash_load'
```
Alternatively, run the following commands in U-Boot using a USB to TTL serial adapter:
```
setenv mainlineLinux
setenv arcNumber
setenv bootcmd 'run flash_load'
saveenv
```
Power down the ix2-200. Place one or two (preferably clean) disks in its bays and connect the prepared USB flash drive to any of the USB ports.

Now press and hold the reset button on the back of the ix2-200. Power up the ix2-200 while keeping the reset button pressed. After ~90 seconds the USB thumb drive will be read intensively, which can be observed if the USB thumb drive has an activity LED. Release the reset button when this happens.

The ix2-200 will power itself down after the recovery is complete. Remove the USB flash drive and power the ix2-200 on. The original firmware will be restored. Check your DHCP server to see which IP address is assigned to the ix2-200. The default settings are as follows.

Web interface: http://hostnameorip/

No username and password will be set for web access.

# Appendix D: Reading system temperature and adjusting fan speed

Install some packages:
```
pacman -S --noconfirm lm_sensors hddtemp
```
To read the mainboard temperature, run:

```
sensors
```
Alternatively, to get pure temperature values, run:

```
cat /sys/class/hwmon/hwmon0/temp{1,2}_input
```
To read the HDD temperature for each disk, run:
```
hddtemp /dev/sd?
```
Fan speed is controlled through pwm1 in sysfs (/sys/class/hwmon/hwmon0/pwm1*). For example, to turn the fan on at the maximum speed, run:

```
echo -n 1 > /sys/class/hwmon/hwmon0/pwm1_enable
echo -n 255 > /sys/class/hwmon/hwmon0/pwm1
```

See here for more information about ix2-200 fan speed control, including a useful script for configuring a fan curve.

# Appendix E: Enable the crypto accelerator

The chipset of the ix2-200 contains a Marvell Cryptographic Engines And Security Accelerator (CESA), which is useful to offload the CPU when using a VPN.

To prepare the crypto accelerator, run:
```
pacman -S --noconfirm --ask 4 linux-kirkwood-dt-headers cryptodev-dkms openssl-cryptodev
```
To start the crypto accelerator immediately and at boot, run:

```
echo "cryptodev" > /etc/modules-load.d/cryptodev.conf
modprobe cryptodev
```
To temporarily disable the crypto accelerator, run:
```
rmmod cryptodev
```
To see the supported algorithms, run:
```
cat /proc/crypto|grep -B 2 "marvell_cesa"
```
To run a benchmark:
```
openssl speed -evp aes-256-cbc
```
Programs using OpenSSL should use the crypto accelerator automatically once it is loaded.