title:        Making a Pi Disk Image
categories:   arm pi
date:         2014-06-30 21:13

After setting up the Raspberry Pi, it's a good idea to make your own installation image
from it. This mainly involves using the `dd` command to clone the SD card into a file.
However, there are some extra steps that can be used to produce a smaller file, and to
provide a helpful login message to anyone using the image.

<!-- more -->


### Preparation on the running system

After setting up your Raspberry Pi - but before shutting it down - prepare the installed
system for imaging.

Edit the login message to detail setup tasks required on logon:

```sh
echo -e "\e[47;30;1m
Please generate new host keys by using the following commands:
  rm /etc/ssh/ssh_host_*
  dpkg-reconfigure openssh-server

Set user and root passwords with 'sudo passwd $(logname) && sudo passwd'

You should also use 'sudo raspi-config' to change the hostname
and expand the filesystem to fill the SD card.

Use the following command to suppress this message in future:
  touch ~/.hushlogin
\e[0m" | sudo tee /etc/motd

# ensure the message is shown
rm -f /home/$(logname)/.hushlogin
```

Clean up the system by removing any unneeded files and private user data.

```sh
# login as root
sudo -s

# install secure-delete
apt-get install secure-delete

# clean up downloaded packages, log, caches, etc.
apt-get clean
find /var/log -type f -delete
du -sch /var/cache/*
df -h /

# backup files from your home directory before removal
cd /home/$(logname)
zip -FS -ry1 /media/KEY/pihome.zip .

# remove any other unneeded or private user data
nano /etc/network/interfaces
nano /etc/wpa_supplicant/wpa_supplicant.conf

# list of temp files in home dir
tmpfiles=".*_history .lesshst *.log .cache/ .hushlogin"
# list of config files in home dir
dotfiles=".config/ .local/ .git* .gem/ .npm/ .ssh__/"
# remove transient and config files for root and user home
for u in /home/$(logname) /root; do echo &&
  cd $u && pwd && rm -rfv $tmpfiles $dotfiles
  ls -A
done
```

Zero fill the unused SD card space for better compression. This can instead be done with an
image file[^gnome-disk] if you're concerned wearing out the card, or just want better speed.

[^gnome-disk]: The easiest way I've found to mount the image as writable, is to first create loopback devices using `gnome-disk-image-mounter -w`.  You can also calculate the partition offsets and use `mount` directly.

```sh
# zero fill the swap file
swapoff -a
dd if=/dev/zero of=/var/swap bs=1M count=100
mkswap /var/swap

# zero fill the filesystem
sfill -z -l -l -f -v /boot /

# shutdown and remove the card
poweroff
```

Here's the basic command to copy an SD card into a file. The size is correct for the
current Raspbian image at [raspberrypi.org](http://raspberrypi.org), and can be confirmed
by viewing the size of the original uncompressed image: `unzip -l raspbian.zip`, or
by using `fdisk`[^fdisk].

```sh
# location and size of SD card
dev=/dev/rdisk2
size=2825

# create the image
date=$(date +%Y%m%d)
name=raspbian-minimal
sudo dd if=$dev of=$name.img count=$size bs=1m

# compress the image using 7z (without BCJ filter)
7z a -mf- $date-$name.img.7z $name.img
```


[^fdisk]: Use `fdisk` to view the partition table info of the SD card, then calculate the
    total amount to copy using `bc`:

        # confirm the above values
        sudo fdisk $dev  # append '-l' flag on linux

        # on OS X, add the last partition start to its size
        bc -l <<< '(122880+5662720)/2/1024'

        # on Linux, use the last partition block
        bc -l <<< '(5785599+1)/2/1024'

<!--
If the image is going to be compressed straightaway, it's also possible to read and
compress in one operation.  This will be much slower than reading the SD card normally.

```sh
# read the sd card and compress using 7z
sudo dd if=$dev count=$size bs=1m | 7z a -mf- -si$name.img $date-$name.img.7z
```
 -->

<!--

The above method is sufficient to take a backup of a card that's had some initial setup,
but hasn't yet had it's filesystem expanded and does not require mounting the filesystem
for further modifications. Otherwise, skip to the more detailed instructions to cleanup
the image and determine it's correct size.

[ubuntu]: http://www.ubuntu.com/download/desktop/



```sh
# reduce size of the main partition in gparted
# (to minimise the sd card read/write time)

# set device path for sd card
dev=/dev/sdc

# check the filesystems
umount ${dev}?
fsck -pfv ${dev}?

# extract partition info
partedm() { usage="Usage: partedm dev part field unit"
            [ $# -ne 4 ] && echo $usage && return
            parted -m $1 unit $4 print |
            grep "^$2" | cut -d: -f$3 | tr -d TGMKB; }
size=$((($(partedm $dev 2 3 b)+1) /1024/1024))
boot=$(partedm $dev 1 2 b)
main=$(partedm $dev 2 2 b)
echo "total size = $size MB
 boot starts at  $boot bytes
 main starts at $main bytes"

# copy the sd card - using calculated size of image
dd if=$dev of=$name.img bs=1M count=${size}

# mount image file and zero fill free space
for part in boot main; do
  mkdir -p $part
  mount -o loop,offset=${!part} $name-basic.img $part &&
    sfill -z -l -l -f -v $part
done

# zero fill the swap file
dd if=/dev/zero of=main/var/swap bs=1M count=100
mkswap main/var/swap

# unmount (after making any final changes)
umount boot main && rm -rf boot main

# compress the image using 7z (without BCJ filter)
7z a -mf- $name.img.7z $name.img
```

 -->


<!--
# make image on Mac OS X
dev=/dev/rdisk2
fdisk $dev
bc -l <<< '(122880+3481600)/2/1024'
dd if=$dev of=$name.img bs=1m count=$size
-->

<!-- OFFSET=`fdisk -lu $IMAGE | grep -m 1 Linux$ | awk '{ print $2 *512 }'` -->
