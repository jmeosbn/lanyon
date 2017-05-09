title:        Windows 8.1 - EFI install
categories:   efi windows macbook
date:         2014-06-16 17:40

I recently wanted to test something on Windows, and not wanting to use a virtual machine,
I decided to install Windows 8.1 on my MacBook.

All Intel based Macs use EFI with a GUID partition table (GPT), instead of the
traditional Wintel BIOS/MBR combination. Until recently, this has meant that Windows can
only be installed by emulating an MBR disk, typically by using Apple's Boot Camp to
create the Windows partition.

Since Windows 8 now supports installation under EFI[^win7efi], and Boot Camp is fairly rigid about
the placement of a Windows partition, I wanted to try installing Windows 8.1 using EFI and
GPT just as I do with Mac OS X and Ubuntu.

[^win7efi]: Windows 7 actually supports efi booting from CD, but the efi loader and
    related files were not included in the CD's filesystem (there
    [is a hack](http://forums.bit-tech.net/showthread.php?t=209045) to work around
    this, but it doesn't seem to work on my MacBook).

<!-- more -->


*Note: this procedure gets rapidly more complicated under only slightly unusual
circumstances (such as having a non Windows OS installed, or more than one disk). Since
this information involved some trial and error to discover, I post it here more for the
record than as an endorsement of something that's worth doing (it's not too hard, but
seriously, Ubuntu is free and much easier to install!).*

* TOC
{:toc}


## Why the Windows install will fail

The issues below will prevent the installation of Windows 8 (and 8.1) from succeeding
when installing under EFI on a Mac or other EFI based system. The lack of clear error
messages actually makes an EFI installation seem impossible, despite the EFI boot loader
being present on the Windows disc. However, the fixes are fairly straightforward once you
understand the issues.

- When installing from a USB key[^usbkey], that key must use an MBR partition table
- The target for the installation must use a GUID partition table (GPT)
- The target disk must not have a hybrid GPT/MBR partition table, such as that created by Boot Camp, Disk Utility, or GParted[^gparted].  A GPT with *protective* MBR is fine though, and protects the GUID partitions on unsupported systems
- Windows will normally fail to install on systems with more than one GPT disk

[^usbkey]: Write a new MBR partition table to a USB key and add a single FAT32 partition.
    Then copy the Windows install files from the disc to the USB
     key using `rsync -aPh <source>/. <key>`.

[^gparted]: Disk Utility only adds a hybrid MBR when creating a new partition table and
    will retain any protective MBR already present. Annoyingly, GParted will re-add a hybrid
    MBR whenever it modifies a disk that uses GPT, and this will prevent Windows from
    starting under EFI until the hybrid MBR is removed.


## A new Windows partition

The first task is to make some free space available, and then create a new NTFS partition
for Windows to install to (size should be at least 20GB). It's easier to do this now,
before trying to install Windows, for reasons that will become apparent.

If you'll be resizing any Mac HFS+ partitions then you should probably use Disk Utility
on OS X. Likewise, for resizing ext3/4 and other file systems unknown to OS X, use
[GParted][] (booting from the [gparted live cd] if no Linux system is installed).

Note: On a Mac, the boot manager can be accessed by holding the `alt` key during startup.

[GParted]: http://gparted.org/index.php


## Preparing a GUID Partition Table

You can check the GUID partition table (or create one from an existing MBR) using
`gdisk` - which can be [installed on Mac OS X][gdisk-osx] or used from most Linux
distros, such as the [gparted live cd].

[gparted live cd]: http://gparted.org/livecd.php
[gdisk-osx]: http://sourceforge.net/projects/gptfdisk/files/gptfdisk/0.8.10/gdisk-binaries/gdisk-0.8.10.pkg/download

1. From a terminal, run `sudo gdisk /dev/<target-disk-id>`[^diskid]
2. `gdisk` should display `GPT: present`, but *must not* list `MBR: hybrid`
3. If the disk has a GPT without a hybrid MBR, quit by typing `q` then `enter`

[^diskid]: On Linux systems, the disk will normally be named using the form `sda`, where
    the `a` represents the first disk. On Mac OS X, the form is `disk0`, where the `0`
    represents the first disk.

If the target disk is listed with `MBR: hybrid`, then Windows will fail to install under
EFI. The best way to correct the GPT is to replace the hybrid MBR using `gdisk`.

1. Still in `gdisk`, use `x` to enter the expert menu
2. Enter `n` to create a new *protective* MBR
3. Check the protective[^protective] MBR and GPT by using `o` and `p` respectively
4. You can abort before writing any changes by using `q` to quit
5. Once satisfied, write out the new table to disk by using `w`

If the target disk is listed with `MBR: MBR only`, then `gdisk` should have automatically
created a GUID partition table in memory[^gptwarning]. If the MBR partition table isn't
required to support older systems or legacy boot loaders, then use the menu commands
shown above to check the existing partitions have been added correctly before writing out
the tables to disk.

[^protective]: A *protective* MBR usually lists only one partition, that spans the entire
    disk to prevent non-GPT aware tools from identifying the disk as being uninitialised or
    empty. A *hybrid* MBR exploits the protective MBR to make up to four primary partitions
    available to systems that don't support GPT disks. The use of a hybrid MBR is
    discouraged, and not supported by Windows when booting under EFI.

[^gptwarning]: It may be that `gdisk` can't write out the GPT due to lack of free spaceat
    the start of the disk. In this case you can try to resize the first partition to start a
    few megabytes later.

[`homebrew`]: http://brew.sh/


## Install Windows to a GPT disk

Start the Windows installation using your preferred media, select the custom install
option, then continue on to the partition selection screen and select the empty NTFS
partition [created previously](#a-new-windows-partition). With a *single non-hybrid* GPT
disk, Windows should install without complaint.

Note: While you can use the installer to format or delete existing partitions, using it
to create a new partition from free space will likely create additional system partitions
which you may prefer not to have littering your disk. If you don't want to reboot at this
point to create an NTFS partition, `diskpart` can be used to avoid the additional
partition clutter.

With some free space ready to be partitioned, type `Shift + F10` to bring up a command
prompt and use the following commands.

    # start the diskpart utility
    diskpart

    # select the target disk
    list disk
    sel disk <target disk>
    list part

    # create a new partition using the selected disk's free space
    create part pri
    format fs=ntfs label=Windows quick

    # exit diskpart
    exit


## Manually mount the EFI partition

With more than one GPT disk, Windows will fail to install as it only expects to have one
EFI system partition in which to store its boot configuration files. To work around this,
use `diskpart` to manually mount the EFI partition from the target installation disk,
then tell Windows to use that partition by using `bcdedit`.

Type `Shift + F10` to bring up a command prompt.

	# start diskpart
	diskpart

	# mount the efi volume
	list vol
	sel vol <efi volume>
	assign letter=s
	exit

	# set the EFI partition with bcdedit
	bcdedit /sysstore s:


## Delete the previous boot configuration

If you've previously had Windows installed using EFI, then the previous boot loader and
boot configuration data store (BCD) will likely still be installed in the EFI system
partition. To prevent previous Windows entries from showing up as duplicates in the
Windows boot manager, you can delete the previous BCD store before installing Windows.

Assuming the EFI partition is mounted to `S:\`, the following command will delete the BCD store.

	# delete boot store from a previous windows install
	del s:\EFI\Microsoft\Boot\BCD

*If you forget to do this until Windows is installed, and a previous configuration did
exist, then a system selection screen will be shown after each startup of Windows. To
remove the additional entry, use `bcdedit /delete {id-of-boot-entry}`.*


## Continue installation of Windows

Back in the installer, you should now be able to select the new partition and continue
the installation as normal. When Windows reboots, it may be necessary to use the system's
EFI boot manager to select the Windows partition and complete installation.


## Why the Recovery Console will fail

With Windows installed, you may at some point need to use the recovery console. On EFI
systems with more than one GPT disk, you'll find that most of the recovery tools, such as
System Restore, will fail due to the same issue that causes the installation to fail.

From _Troubleshoot -> Advanced Options -> Command Prompt_, you can manually mount the
correct EFI partition using `diskpart` as
[described above](#manually-mount-the-efi-partition). Once done, simply use the graphical
menu to select *System Restore* or another recovery option.


## Legacy boot manager

Windows 8 uses a graphical boot manager that's only shown once the system is mostly
booted. Apart from taking longer to switch between systems, there is no access to
advanced boot options such as safe mode etc., without first being able to start the
graphical boot manager or recovery environment.

Previously, advanced options could be accessed by holding `F8` during startup, but
Windows no longer checks for this key[^f8key] as it adds a second or so to the boot
time. If you'd prefer to have the menu available instead, then the `legacy` boot menu
policy can be enabled using `bcdedit`.

	# allow F8 during boot to access advanced options
	bcdedit /set bootmenupolicy legacy

[^f8key]: The check is supposedly still there, but is so short that it can't be activated.

Alternatively, the advanced options can be shown for the next boot only, or for every boot
(without needing to hold `F8`).

	# show advanced options for the next boot only
	bcdedit /set onetimeadvancedoptions on

	# always show the boot menu before system startup
	bcdedit /set {bootmgr} displaybootmenu yes
