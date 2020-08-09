[Home](index.md)

# Linux Boot Process

## Bootloader
---
Bootloader is used to:
- Boot other operating systems, and usually each operating system has a set of bootloaders specific for it. 
- Contains several ways to boot the kernel and commands for debugging and/or modifying the kernel environment. 

Since it is usually the first software to run after powerup or reset, bootloaders are highly processor and board specific.

On older computers with BIOS (Basic Input/Output System), a boot loader resides in the MBR (Master Boot Record), which occupies the first 512 bytes on a disk, but newer computers with UEFI (Unified Extensible Firmware Interface) store it in a special partition called EFI System Partition. A boot loader is loaded by BIOS or UEFI after a successful POST (Power-On Self-Test).

## GRUB2
---
GRUB is a bootloader capable of loading a variety of operating systems. Its function is to:

- Take over from BIOS at boot time
- Load itself
- Load the Linux kernel and associated kernel modules (or libraries) stored in a file referred to as the initrd / initramfs.
- Set kernel command-line parameters
- Turn over execution to the kernel. 

**Once the kernel takes over, GRUB has done its job and it is no longer needed**. 

GRUB supports multiple Linux kernels and allows the user to select between them at boot time using a menu. This is a very useful tool because there are instances where an application or system service fails with a particular kernel version. Many times, booting to an older kernel can circumvent issues such as these. 

**By default, three kernels are kept**: the newest and two previous, when yum or dnf are used to perform upgrades. The number of kernels to be kept before the package manager erases them is configurable in the `/etc/dnf/dnf.conf` or `/etc/yum.conf` files.

GRUB is dynamically configurable - this means that the user can make changes during the boot time, which include altering existing boot entries, adding new, custom entries, selecting different kernels, or modifying initrd. 

## Boot Process
---
1. **Locate and Load MBR**: When a system is first booted, or is reset, after the POST, the processor executes code at a well-known location (stored in the BIOS). The BIOS must determine which devices are candidates for boot. A boot device can be a floppy disk, a CD-ROM, a partition on a hard disk, a device on the network, or even a USB flash memory stick. Commonly, Linux is booted from a hard disk, where the **Master Boot Record (MBR)** contains the primary boot loader. MBR it is located in the first sector of the bootable disk (typically **/dev/hda, or /dev/sda**) and is only 512 bytes long. MBR contains a small piece of bootstrap code (446 bytes) called the **primary or Stage 1 boot loader**and a partition table (64 bytes) describing the primary and extended partitions. 

```bash
fdisk -l /dev/sda #Locate MBR - bootable partition (/boot or /) will have an *
dd if=/dev/sda bs=512 count=1 | hexdump -C #View MBR
dd if=/dev/sda bs=512 count=1 of=/tmp/mbr.img #Copy MBR
dd if=/tmp/mbr.img of=/dev/sda bs=512 count=1 #Restore MBR
```

2. **Load Stage 1 Bootloader**: MBR loads the Stage 1 Bootloader and executes it. Given how tiny the bootstrap code section of the MBR is (446 bytes), Stage 1 bootloader basically interrogates the partition table in the MBR  to look up another file (GRUB2 bootloader) from the disk and load it in RAM to perform the actual boot process. As such, this bootstrap code is often termed a “stage one bootloader”.

3. **Load Stage 2 Bootloader**: Stage 2 bootloader is located in /boot and runs the main body of the GRUB2 code. When GRUB2 loads, it presents information from a configuration file, normally located in `/boot/grub2/grub.cfg`. This is the stage where we get the option to select the operating system to be run, and start the system you’ve chosen. At this stage, user can interrupt the booting process and select specific kernel to boot into and pass additional parameters to the kernel if required. 

The task of a boot loader sounds simple: It loads the kernel into memory, and then starts the kernel with a set of kernel parameters. But consider the questions that the boot loader must answer:

- Where is the kernel?
- What kernel parameters should be passed to the kernel when it starts?
	
The answers are (typically) that the kernel and its parameters are usually somewhere on the root filesystem. It sounds like the kernel parameters should be easy to find, except that the kernel is not yet running, so it can’t traverse a filesystem to find the necessary files. Worse, the kernel device drivers normally used to access the disk are also unavailable. We will see below how this is addressed using the `initramfs`
	
**The 2nd stage bootloader performs the following activities:**

	1. Search for the compressed kernel image (vmlinuz) in the /boot directory and load it into memory.
	2. Extract the contents of the **initial RAM disk, initrd or initramfs** (which is a file containing loadable kernel modules). initramfs file contains a mini-root filesystem that has the kernel modules necessary when the system is booting. It is located in /boot and there is a unique initramfs file for each kernel. It’s main purpose is to provide kernel modules required to mount the root filesystem. It consists of a set of directories bundled in an archive that is extracted at boot time. *This initrd serves as a temporary root file system in RAM and allows the kernel to fully boot without having to mount any physical disks. Since the necessary modules needed to interface with peripherals can be part of the initrd, the kernel can be very small, but still support a large number of possible hardware configurations.*


To hand over the control to the kernel, the bootloader has to achieve two major things.
	- **Load the kernel and initramfs into memory**: GRUB will not load the kernel (/boot/vmlinuz) at any random location; it will always be loaded at a special location which varies as per the Linux distribution/version and CPU architecture of the system. **vmlinuz is an archive file, and is made from three parts**:
	
	```bash
	vmlinuz =  Header + Kernel Setup Code + vmlinux (actual compressed kernel)
	vm = virtual memory, z = zipped file
	```
vmlinuz is a compressed file of the actual kernel’s binary vmlinux. You cannot decompress this file with gunzip/bunzip or even with tar. **The kernel extracts itself with the help of the header of the vmlinuz file.**

	- **Set kernel command-line parameters**: These are parameters like root device name, mount options like ro or rw, the initramfs name, the initramfs size, etc. and the are passed by GRUB/the bootloader to the kernel. `cat /proc/cmdline` can show the kernal command line parameters. In this file, the `root` parameter specifies the location of the root filesystem; without it, the kernel cannot find init and therefore cannot perform the user space start. On most modern systems, this location is specified as UUID, which is a type of unique serial number. To view a list of devices and the corresponding filesystems and UUIDs on your system, use the `blkid` command.

4. **Start systemd**: After the kernel is booted, the root file system is pivoted, where the initrd root file system is unmounted and the real root file system is mounted (as specified in the “root=” in grub.conf file) in read only mode, and starts the **systemd** process with a PID of 1.
5. **Initialize system and Launch Services**: systemd initializes the system and launches all the services that were once started by the traditional init (/etc/init.d) process. systemd process reads the configuration file `/etc/systemd/system/default.target` which defines the services that systemd starts, and brings the system to the state defined by the system target.

## What is initrd/initramfs?
Back at the dawn of UNIX, devices were few and kernels were small. Most devices were physically attached to a computer all the time and a kernel was typically built to support exactly the installed hardware. As time went by and the number and type of devices proliferated, device support was shifted into loadable kernel modules. Kernel modules are like a library of functions with well-defined entry points that allow a kernel to use a device whose support was not compiled in to the kernel. This allowed a kernel to be loaded and then load support only for devices that were actually installed in the system. But without full disk and filesystem support, how do you find and load these modules from disk? The answer is the initial RAM disk or initrd or initramfs, which is a file containing loadable kernel modules. It is usually compressed and is loaded into RAM by the boot loader. The kernel is given access to it as if it were a mounted file system.

The initial RAM disk (initrd) is an initial root file system that is mounted before the real root file system - the creation of a preliminary root file system would contain just enough in the way of loadable modules to give the kernel access to the rest of the hardware. Intial RAM Disk refers to a simple idea of an “early userspace” root filesystem that can be used to get at least the minimum functionality loaded in order to let the boot process continue. Instead of using initrd, some Linux filesystem will also use initramfs, which is a successor of initrd.

The initrd function allows you to create a small Linux kernel with drivers compiled as loadable modules. These loadable modules give the kernel the means to access disks and the file systems on those disks, as well as drivers for other hardware assets. Because the root file system is a *file system* on a disk, the initrd function provides a means of bootstrapping to gain access to the disk and mount the real root file system

The **dracut** utility is used to build an initramfs, and **/etc/dracut.conf** is its configuration file.

## Managing GRUB2 Configuration
- `/boot/grub2/grub.cfg`: This file contains the configuration of the GRUB 2 menu items. **This file should NOT be edited** — it is automatically generated by the `grub2-mkconfig` command (listed below), and is first generated during Linux install, is regenarated when a new kernel is installed. 
- `/etc/default/grub`: This is the input file for GRUB2 and it usually includes additional environmental settings such as backgrounds and themes. After editing this file, update the main configuration file as shown below.

```bash
#Editing GRUB Configuration
grub-mkconfig -o /boot/grub2/grub.cfg
```                                               

If you do not specify the `-o` flag, the output will be written to stdout (the screen), which is not useful.                                                                       
- `/etc/grub.d/` - It contains GRUB scripts. These scripts are building blocks from which the grub.cfg file is built. When the relevant GRUB command is executed, the scripts are read in a certain sequence and grub.cfg is created.

## systemd
systemd is the default Linux initialization system and service manager. Linux requires an initialization system during its boot and startup process. At the end of the boot process, the Linux kernel loads systemd and passes control over to it and the startup process begins. During this step, the kernel initializes the first user space process, the systemd init process with process ID 1, and then goes idle unless called again. systemd prepares the user space and brings the Linux host into an operational state by starting all other processes on the system.

Note is that **/sbin/init is a symlink to  /lib/systemd/systemd**, i.e. systemd takes over the init process. systemctl is used for most basic tasks:

```bash
systemctl start|stop|restart <service_name> #Starting, stopping, restarting a service
systemctl enable|disable <service_name> # Enabling or disabling a system service at system boot
systemctl status <service_name> #Check the status of a service
```

## Rebooting and Shutting Down
* **Shutdown:**The preferred method to shut down or reboot the system is to use the `shutdown` command. This sends a warning message, and then prevents further users from logging in. The init process will then control shutting down or rebooting the system. You may specify a time string (which is usually `now` or `hh:mm` for hour/minutes) as the first argument. Additionally, you may set a wall message to be sent to all logged-in users before the system goes down. Example,sudo shutdown -h now “Server is shutting down.”
* **Reboot**: `reboot` or `shutdown -r` causes the system to reboot.

## Reset Forgotten Root Password
1. In the GRUB menu, select the kernel and hit `e` to edit:
2. Append `rd.break` at the end of the linux kernel command line and press `Ctrl-X` to reboot. This will break the boot process just before the control is handed from initramfs to the actual system:
3. We are now presented with the root shell, with the root file system mounted as read-only on /sysroot. To verify if the the root filesystem is mounted as read-only run `mount | grep -i sysroot`:
4. Remount sysroot as read-write `mount -o remount,rw /sysroot`. Note, there is a space between `rw` and `/sysroot`
5. Switch to a **chroot** jail with the `chroot /sysroot` command. This ‘changes the root’ to another location. Once a new root is declared via chroot, any references that a user or application makes to ‘/’ will resolve to the new directory. This is a pretty effective way to restrict access to the real root and therefore the real file system. In fact, sometimes that act of chrooting is referred to a jailing or a chrooted shell is referred to as a jail shell.
6. Now use the `passwd root` command to change the root password.
7. When you run the passwd command to do the password reset, the associated shadow file (/etc/shadow) is modified with an incorrect SELinux context. Run the `touch /.autorelabel` command which will create a hidden file named .autorelabel under the root directory. On the next boot, the SELinux subsystem will detect this file, and then relabel all of the files on that system with the correct SELinux contexts.
8. Type `exit` twice to reboot.