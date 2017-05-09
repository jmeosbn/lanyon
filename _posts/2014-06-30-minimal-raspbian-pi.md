title:        Minimal Raspbian Pi
categories:   arm pi dev
date:         2014-06-30 23:42

A collection of notes for setting up a clean and minimal install of the official [Raspbian]
image from [raspberrypi.org].

[Raspbian]: http://raspbian.org
[raspberrypi.org]: http://raspberrypi.org/downloads

<!-- See [Minimal Raspbian Net Installer](#minimal-raspbian-net-installer) if you prefer to
build a minimal image from scratch using the
[Raspbian unattended netinstaller](https://github.com/hifi/raspbian-ua-netinst). -->

### Comparison of compressed images

`826 MB` - Original 2014-06-20-wheezy-raspbian  
`284 MB` - Additional packages added, desktop extras removed  
`156 MB` - Desktop environment removed, `udisks-glue` installed  
`184 MB` - As above, with `mono` installed


<!-- more -->

### Short summary of changes

After some initial setup, some common development tools and other utilities were added.
The desktop extras were then removed, followed by the desktop itself. To handle
auto-mounting of USB drives, `udisks-glue` was installed as a service using `supervisor`.
Being quite large, `Mono` was installed after room had been made for it. Wiring Pi is
built for `gpio` access, and for dropbox sync, `dbox` is used.


<!--

## Disable password based logon

Disable password logins for remote ssh access (public key only).

### TODO:

 -->

_Note: less common changes are listed towards the end of the post_

* TOC
{:toc}


## Basic Pi Tips

You can copy an image to the SD card, without first extracting it:

```sh
# Replace image archive and device name with correct values
7z e -so raspbian.img.7z '*.img' | sudo dd of=/dev/sdc bs=1M

# 7z should support most compression formats
7z e -so raspbmc.img.gz '*.img' | sudo dd of=/dev/sdc bs=1M

```


To backup an entire SD card to an image file, use `dd` again:

```sh
# Replace device name with correct values
sudo dd if=/dev/sdc of=raspbian.img bs=1M
```

For details on making an optimised `dd` image, see
[Making a Pi disk image](/blog/making-a-pi-disk-image/).


## First login tasks

The first task after booting the Raspberry Pi, is to set up the environment. For a
headless Pi (one without a monitor or keyboard connected), you may want to use a
serial connection to login until WiFi and zeroconfig [have been configured](#connecting-to-the-pi).

Note that there will be a login message recommending to run `sudo raspi-config`. This can
be done later for tasks more specific to the installation, such as expanding the
partition to fill the SD card.

Most of the commands below need root privileges on the Pi, as they alter the system
configuration. To run a single command with root privileges, prepend the `sudo` command
to it. To run many commands this way without typing `sudo` each time, first start a root
shell with `sudo -s` (*logout with 'exit' or ctrl+d when finished*).

<!--
# retain user's $HOME for non-login shell using 'sudo'
# sed -i '/env_keep.*\+.*HOME/b; $ a\\nDefaults env_keep += \"HOME\"' /etc/sudoers
-->

```sh
# Login to the pi over serial, user:pi pw:raspberry
screen /dev/tty.usbserial 115200

# Disable the login message
touch ~/.hushlogin

# Disable setting of the terminal title
sed -i '/PS1=.*\][0-2];/s/\w/## &/' ~/.bashrc

# Use a root shell for the following commands
sudo -s

# Change password for root and user
passwd && passwd $(logname)

# Temporarily fit serial terminal to local window size
stty rows 17

# Configure time zone and locale
dpkg-reconfigure tzdata
dpkg-reconfigure locales

# Hide the Raspberry Pi logo during boot
sed -i "/logo\.nologo/b; /1/s/$/ logo.nologo/" /boot/cmdline.txt

# Don't prompt to run raspi-config on login
rm -f /etc/profile.d/raspi-config.sh

# Enable colored xterm support over serial
sed -i '/ttyAMA0/s/vt100$/xterm-256color/' /etc/inittab

# Update packages
apt-get update && apt-get upgrade

# Add tools for fat, exfat, and ntfs
apt-get install dosfstools exfat-fuse ntfs-3g

# Support for zeroconfig
apt-get install avahi-daemon

# Other utilities
apt-get install p7zip-full zip curl psmisc usbutils iw bc
```


#### Preferred Packages

Depending upon your uses, you may have a set of packages you install to every fresh
Raspbian image.  Below are the packages I like to install.

```sh
# SysV configuration tool (not for systemd)
apt-get install sysv-rc-conf

# Install scripting tools
apt-get install ruby ruby-dev ri libsqlite3-dev nodejs-legacy npm

# Set npm to use known registrars only
npm config set ca ""

# Install node packages
npm install -g node-gyp markdown-preview

# Install misc. packages
apt-get install mediainfo irssi

# Install rmate for TextMate editing over ssh
gem install rmate
```


## Change Hostname

If it's likely the Pi won't be the only one using the local network, then it should be
given a unique host name.

```sh
# replace the hostname in each file
NEWNAME=snhack
sed -i "s/${HOSTNAME}$/${NEWNAME}/" /etc/host{s,name}

# enable the changes
/etc/init.d/hostname.sh start
```


## Generate new RSA host keys

These keys confirm the identity of the Pi, to prevent a malicious host from intercepting the remote login process.  Not so important for your HTPi, but good standard security practice.  Also recommended if you have [more than one Pi](#change-hostname).

```sh
rm /etc/ssh/ssh_host_*
dpkg-reconfigure openssh-server

# remove old host key from clients using:
# ssh-keygen -R raspberrypi.local
```
<!--
ssh-keygen -t rsa -f /etc/ssh/ssh_host_rsa_key
ssh-keygen -t rsa1 -f /etc/ssh/ssh_host_key
-->

If you're using dropbear instead of openssh, then use:

```sh
rm -f /etc/dropbear/dropbear/dropbear_*
dropbearkey -t rsa -f /etc/dropbear/dropbear_rsa_host_key
dropbearkey -t rsa -f /etc/dropbear/dropbear_rsa_host_key
/etc/init.d/dropbear restart
```


## Configure WiFi adapter

Some distributions include WiFi setup as part of the terminal configuration script, but
this functionality is not yet included in `raspi-config`. If keeping the desktop
installed, then its network configuration tools can be used. Otherwise, use the
instructions for [manual configuration](#manually-configure-wifi-adapter).


<!-- My bluetooth adapter is not supported in the current build of Raspbian/XBian, but
[here is the procedure](#configure-bluetooth-adapter) I used when
trying to get it running (confirmed working on a laptop running Ubuntu). -->

<!-- If the keyboard layout is incorrect, you'll need to configure
[regional system settings](#regional-system-settings). -->


## SSH login by public key only

If the Pi is open to the internet or a shared WiFi hotspot, it's recommended to disable
SSH password logins (after first setting up [public key login](#copy-public-key-to-the-pi)).

```sh
# add settings to sshd config
echo '
# enforce public key login
ChallengeResponseAuthentication no
PasswordAuthentication no
UsePAM no' | sudo tee -a /etc/ssh/sshd_config

# restart sshd
sudo /etc/init.d/ssh reload
```


## Update dosfstools build

The current raspbian version of `dosfstools` is out of date and won't reset the 'dirty bit'
of a FAT volume, such as that used by the `/boot` partition.

```sh
git clone http://daniel-baumann.ch/git/software/dosfstools.git
cd dosfstools
make install
```


## Add a cron job for dynamic DNS

If you have a dynamic DNS entry to keep updated, or another simple command or script to
run every so often, then a cron job can be set up to run as a particular user.

```sh
# run crontab for the 'pi' user
sudo -u pi crontab -e

# append line to run command every 5 mins
*/5 * * * * curl -k -o /tmp/duckdns.log 'https://www.duckdns.org/update?domains=yourdomain&token=yourtoken&ip=' >/dev/null 2>&1
```


## Backup installation

At this point we have a Raspbian image with minor tweaks, added/updated packages, and
zeroconfig support which provides mDNS/Bonjour discovery of the Pi over a network without
needing it's IP address.

To minimise the time spent installing updates on future install, it's a good idea to
[backup the card](/blog/making-a-pi-disk-image/) to a disk image that can be reinstalled
later. This will also come in handy if you decide later to revert large changes made
after this point (such as removal of the desktop).


## Removing the desktop environment

If you only plan to run the terminal environment (and self-hosted programs, such as
XBMC), then a lot of space can be recovered by removing x11 and related packages.

```sh
# desktop extras
apt-get purge oracle-java.* wolfram-engine
apt-get purge squeak-vm sonic-pi pistore midori dillo
apt-get purge penguinspuzzle 'raspberrypi-(artwork|doc)'

# samba network drives
apt-get purge samba-common smbclient

# other libraries
apt-get purge python-numpy python-pygame python3-numpy
apt-get purge liblapack3 libatlas3-base
```

At this point, the base desktop system and utilities are still installed. However, over
800MB has already been recovered - so if you just wanted to save a little space, you may
want to keep the desktop after all.

<!-- *TODO: mono depends upon libx11-6, so it'll be (re)installed with mono as req.* -->

```sh
# desktop environment
apt-get purge x11-common libx11-.* xkb-data xdg-utils menu-xdg
apt-get purge desktop-file-utils lxde-icon-theme gnome-themes*
```
<!-- # apt-get purge fonts-freefont-ttf -->

Finally, purge packages no longer needed and clean up unused files.

```sh
# cleaning up
(cd ~pi; rm -rf Desktop python_games ocr_pi.png)
rm -rf /usr/share/{icons,images}
apt-get --purge autoremove
apt-get clean
```

*Note: see [cnx-software.com](http://www.cnx-software.com/2012/07/31/84-mb-minimal-raspbian-armhf-image-for-raspberry-pi/) for an even lighter install.*


## Configure automount for usb drives

Install udisks-glue to automount usb drives without starting the desktop.

```sh
# install udisks-glue
apt-get install udisks-glue policykit-1

# edit config file to enable automount
sed -i '/^match disks /a\    automount = true' /etc/udisks-glue.conf

# startup using rc.local
sed -i "/^exit 0/isudo -u $(logname) udisks-glue\n" /etc/rc.local
```

Note: for startup methods other than `rc.local`, see
[angryelectron.com](http://angryelectron.com/udisks-glue-initscript/) for an example of
how to add `udisks-glue` as a service using a traditional init.d script, or the section:
[Add services using Supervisor](#add-services-using-supervisor).


## Add services using Supervisor

[Supervisor] provides a simple way to add user services, without messing with `SysV` or
other `init` scripts.

[Supervisor]: http://supervisord.org

```sh
apt-get install supervisor
```

Create a startup script `/etc/supervisor/conf.d/udisks-glue.conf`:

```ini
[program:udisks-glue]
user = pi
command = udisks-glue -f
autostart = true
autorestart = true
stdout_logfile = /var/log/supervisor/udisks-glue-out.log
stderr_logfile = /var/log/supervisor/udisks-glue-err.log
```

Remove the previous startup of `udisks-glue` - if already
[configured above](#configure-automount-for-usb-drives) - and start the new service
by reloading the `supervisor` config.

```sh
# remove the startup line from rc.local
sed -i "/udisks-glue/d" /etc/rc.local

# stop already running process
killall udisks-glue

# reload supervisor config
supervisorctl reload
```

The `udisks-glue` program should start automatically on boot. Use `supervisorctl` with
`start`, `stop`, and `restart` to manually control Supervisor programs: `supervisorctl
restart udisks-glue`.


## Install Mono

Install Mono development tools, runtime, and interactive shell.

Note: If you've previously shrunk the system partition down, it may need to be expanded
(use `sudo raspi-config`) to be able to fit the mono installation (about 153MB).

```sh
apt-get install mono-devel mono-utils mono-csharp-shell
```

<!--
pkg-count
353 mono-complete
214 cli-common-dev
172 mono-devel
168 libmono-cil-dev

# for a minimal cli toolchain:
apt-get install mono-mcs mono-xbuild mono-csharp-shell
-->


## Link settings to root profile

Occasionally you'll want to use a root shell, and then be annoyed that your aliases etc. are not configured for the root user.  You can either copy the profile files into ~/root or, as shown here, link them symbolically so that any future modifications will be reflected.

```sh
# change to the home folder you want to link from
cd /home/$(logname)

# define a list of dot files to link
dotfiles=".profile .bashrc .bash_aliases .bash_logout .nanorc .toprc .inputrc"

# link each existing file into the root user's home folder
for i in $dotfiles; do
  [ -f $i ] && sudo ln -sfv ~+/$i ~root/
done

# hush login message
sudo touch ~root/.hushlogin
```

If you prefer to copy the files into ~root, then replace `ln -sfv` with `cp`.


## Build and install WiringPi

[WiringPi] is a library to access the Pi's GPIO, SPI, and I2C headers, modelled on the Arduino Wiring system.  It also includes the `gpio` utility for use of the libraries from the command prompt.

[WiringPi]: https://projects.drogon.net/raspberry-pi/wiringpi

```sh
apt-get install gcc make git-core libi2c-dev
git clone git://git.drogon.net/wiringPi
cd wiringPi; ./build

# test with gpio utility
gpio readall

# install ruby gem (optional)
gem install --user-install wiringpi
```


## Install Transmission

The transmission bittorrent client doesn't have many dependencies, can be installed as a
daemon, and accessed remotely using a Web UI.

```sh
# install packages
apt-get install transmission-daemon

# start daemon as logged in user (usually 'pi')
sed -i "/^USER=/s/.*/#&\nUSER=$(logname)/" /etc/init.d/transmission-daemon

# don't store config in '/var/lib/'
sed -i "/^OPTIONS=/s/^/# /" /etc/default/transmission-daemon

# test transmission-daemon uses ~/.config
service transmission-daemon restart
ls -l .config/transmission-daemon

# customise settings for transmission (see below)
nano .config/transmission-daemon/settings.json
service transmission-daemon reload
```

The following block includes settings needed to get the Web UI up and running, pasting it
before the last setting will replace any definitions duplicated earlier in the file.

```json
"watch-dir": "/home/pi/torrents",
"download-dir": "/home/pi/torrents",
"incomplete-dir": "/home/pi/torrents/incomplete",
"incomplete-dir-enabled": false,
"watch-dir-enabled": true,
"trash-original-torrent-files": true,

"rpc-port": 9091,
"rpc-whitelist": "127.0.0.1, 192.168.*.*, 172.16.0.*",

"rpc-password": "pi",
"rpc-username": "transmission",
"rpc-authentication-required": true,
```

Full details on transmission's config files may be found
[here](https://trac.transmissionbt.com/wiki/EditConfigFiles). Other settings such as port
forwarding, bandwidth throttling, and encryption, can be configured from the Web UI
(usually at `raspbian.local:9091`) .


## Dropbox sync using dbox

Follow the [dbox installation instructions][dbox] to set up the dropbox sdk developer
keys and authorisation tokens. To use dbox for automatic folder syncing, see my post:
[Dropbox on Pi](/blog/pi-box/).

[dbox]: https://github.com/kenpratt/dbox

```sh
apt-get install gcc make ruby ruby-dev libsqlite3-dev
gem install dbox
```


## Precompiled XBMC

If you're running raspbian and want to use the same image with XBMC, you can install the
base XBMC and required packages as follows (adapted from
[michael.gorven.za.net](http://michael.gorven.za.net/raspberrypi/xbmc)):

```sh
echo 'deb http://archive.mene.za.net/raspbian wheezy contrib
' | sudo tee /etc/apt/sources.list.d/mene.list

apt-key adv --keyserver keyserver.ubuntu.com --recv-key 5243CDED

apt-get update && apt-get install xbmc

nano /etc/default/xbmc  # edit to enable and set user
```


## Connecting to the Pi

With support for zeroconfig installed (via `avahi-daemon`), it's much easier to
find your Pi on the network[^hostname].

```sh
# login using zeroconfig
ssh pi@raspberrypi.local
```

*Note: If multiple Pis are active on the same network, they should be given
[unique hostnames](#change-hostname).*

[^hostname]: It should be possible to connect with the hostname even without zeroconfig,
    e.g. `pi@raspberrypi`, but I've had no luck with this (except for when the router sets
    this up via DNS).


### Copy public key to the Pi

Before setting up the Pi remotely, there are some things that can be done locally (on your PC, laptop, etc.) to ease logging in when using SSH.  This will obviate the need to enter a password, or specify the full host name each time we access the Pi.

If generating a new key pair, accept the default key location as suggested by `ssh-keygen` below.  While a passphrase is optional, anyone can use a copy of the unencrypted private key to authenticate with your identity.  Many operating systems are preconfigured to use `ssh-agent` or a similar utility, to avoid having to enter a passphrase multiple times (if at all).


```sh
# generate a key pair if none already exists
test -f ~/.ssh/id_rsa.pub || ssh-keygen

# remove any old, conflicting host entries
ssh-keygen -R raspberrypi.local

# password is required until the key is installed
ssh-copy-id pi@raspberrypi.local
```

The easiest way to append a key to the remote user's `~/.ssh/authorized_keys` file is to use `ssh-copy-id` as shown above.  Download [ssh-copy-id] from source (or [github][ssh-copy-id-for-OSX]) if your system doesn't already have it ([installation instructions]), this [alternative method] should also work in most situations.

[ssh-copy-id]: http://hg.mindrot.org/openssh/raw-file/tip/contrib/ssh-copy-id
[ssh-copy-id-for-OSX]: https://github.com/beautifulcode/ssh-copy-id-for-OSX
[installation instructions]: http://www.commandlinefu.com/commands/view/10228/...if-you-have-sudo-access-you-could-just-install-ssh-copy-id-mac-users-take-note.-this-is-how-you-install-ssh-copy-id-
[alternative method]: http://www.commandlinefu.com/commands/view/188/copy-your-ssh-public-key-to-a-server-from-a-machine-that-doesnt-have-ssh-copy-id


### Add an alias to .ssh/config

Locally define the alias `pi`, to be used in place of `pi@raspberrypi.local` with
ssh commands such as `ssh pi` and `sftp pi`. Enter the text below as a single command, or
manually paste the quoted text into `~/.ssh/config` using `nano` or similar (the IP
address can be used for `Hostname` if preferred).

```sh
echo '
Host pi
  User  pi
  Hostname  raspberrypi.local' >> ~/.ssh/config
```


### Transfer files

If you have previous files from your Pi stored locally, you can transfer them using `sftp`, `scp`, etc.  For easily transferring many arbitrary files , a GUI sftp client is recommended.

```sh
# connect to the pi, using the `xb` alias defined above
sftp xb
sftp> put .bash_aliases
sftp> exit

# you can also use ctlr+d to logout
```


### Manually configure WiFi adapter

[Instructions adapted from here](http://www.savagehomeautomation.com/raspi-airlink101).

Use `lsusb` to check that the adapter is recognised, and `lsmod` to check the kernel module (e.g. `8192cu`) is loaded.

```sh
sudo nano /etc/network/interfaces
```

Make sure the following lines exist in the interfaces file, adding them as needed:

```
allow-hotplug wlan0
iface wlan0 inet manual
wpa-roam /etc/wpa_supplicant/wpa_supplicant.conf
```

<!--
To manually set a static IP, add the following lines with the desired values:

    address 192.168.1.31
    netmask 255.255.255.0
    gateway 192.168.1.254

-->

Open the file that configures WiFi hotspots:

```sh
sudo nano /etc/wpa_supplicant/wpa_supplicant.conf
```

Add your network details, using the following template:

```
network={
  ssid="YOUR-NETWORK-SSID"
  psk="YOUR-WLAN-PASSWORD"
}
```

Or, more fully as required:

```
network={
  ssid="YOUR-NETWORK-SSID"
  psk="YOUR-WLAN-PASSWORD"
  proto=WPA2
  key_mgmt=WPA-PSK
  pairwise=CCMP TKIP
  group=CCMP TKIP
}
```

Reinitialise the adapter, and check it's connected.

```sh
sudo ifdown wlan0
sudo ifup wlan0
# you may get some errors here, even when successful
```

Use `iwconfig` to view wifi adapter info and `ifconfig` for general network info.



### Configure bluetooth adapter

[Adapted from ctheroux](http://www.ctheroux.com/2012/08/a-step-by-step-guide-to-setup-a-bluetooth-keyboard-and-mouse-on-the-raspberry-pi/).

```sh
# install bluetooth support and dependencies
agi bluez python-gobject  # minimal?
agi bluetooth bluez-utils  # full

# for management from the desktop
agi blueman

# check adapter is working*
hcitool dev

# scan for devices
hcitool scan

# pair with device, using the address listed from scan
bluez-simple-agent hci0 XX:XX:XX:XX:XX:XX

# trust the device
bluez-test-device trusted XX:XX:XX:XX:XX:XX yes

# connect to input device
bluez-test-input hci0 XX:XX:XX:XX:XX:XX

# adapter status
hciconfig
```

**Note: a bluetooth adapter may be listed in `lsusb` and `hciconfig`, without being recognised by `hcitool`. This is the case with the belkin dongle I have, so use `hcitool` to check that a device is working properly.*






### Install aircrack and related tools

Taken from [blog.petrilopia.net][petrilopia].

[petrilopia]: http://blog.petrilopia.net/linux/raspberry-pi-install-aircrackng-suite/

```sh
apt-get -y install iw reaver
apt-get -y install libssl-dev libnl-3-dev libnl-genl-3-dev
wget http://download.aircrack-ng.org/aircrack-ng-1.2-beta3.tar.gz
tar -zxvf aircrack-ng-1.2-beta3.tar.gz
cd aircrack-ng-1.2-beta3
make
make install
```

### Using Reaver

```sh
apt-get install reaver

# put wifi device into monitor mode
airmon-ng start wlan0

# list available networks
airodump-ng wlan0
# airodump-ng mon0

# set bssid to target network name
bssid="setthistobssid"

# start reaver
reaver -i mon0 -b "$bssid" -vv
```


### Configure automount for usb drives

If there's no service already installed to automount usb drives, then udisks-glue can be
setup as follows.

```sh
# install udisks-glue
apt-get install udisks-glue

# create config file, including spin down of drives after 10 mins
echo 'filter disks {
    optical = false
    partition_table = false
    usage = filesystem
}

match disks {
    automount = true
    automount_options = { sync, noatime }
    post_insertion_command = "udisks --set-spindown %device_file --spindown-timeout 600 --mount %device_file --mount-options sync,noatime"
}' | sudo tee /etc/udisks-glue.conf

# create 'upstart' service definition
echo '#
# udisks-glue
#

description "udisks-glue for udisks"

start on started
stop on (runlevel [06] or stopped dbus)

expect fork
respawn
setuid pi
exec /usr/bin/udisks-glue
' | sudo tee /etc/init/udisks-glue.conf

# alternative startup using rc.local
# sed -i "/^exit 0/isudo -u $(logname) udisks-glue\n" /etc/rc.local
```

Note: see [angryelectron.com](http://angryelectron.com/udisks-glue-initscript/) for an
example of how to add udisks-glue as a service when using a traditional init.d script.


### Install XBian onto a Raspbian image

From [Installing XBIAN directly on RASPBIAN IMG](http://forum.xbian.org/thread-1850.html)

```sh
# install xbian repo
wget http://xbian.brantje.com/pool/stable/main/x/xbian-package-repo/xbian-package-repo_1.0.0_armhf.deb
dpkg -i xbian-package-repo_1.0.0_armhf.deb

# enable staging and development repositories and update
sed -i '/ staging\|devel main/s/^#\+ //' /etc/apt/sources.list.d/xbian.list
apt-get update
# revert previous edit
sed -i '/ staging\|devel main/s/^/### /' /etc/apt/sources.list.d/xbian.list

# install xbmc **only**
# apt-get install xbian-package-xbmc # 97MB

# install xbian and xbmc
apt-get install xbian-package-xbianhome xbian-package-kernel

# reboot required to make file system changes, and conversion to btrfs
sed -i /rootfstype/s/btrfs/ext4/ /boot/cmdline.txt # do not use btrfs
reboot

# install remaining xbian packages
# Note: a warning for removal of 'sysvinit' will be displayed,
# as xbian replaces it with the 'upstart' service manager.
# Also, cmdline.txt should remain unchanged, with new
# parameters written to /boot/cmdline.new.
apt-get install xbian-update xbian-package-rasp-switching

# XBMC can now be started with "start XBMC"

# boot to desktop
# apt-get install lightdm
```

<!-- XBMC can be started with "start XBMC", Raspbian X desktop with "service lightdm start".

if you Raspbian setup used to boot into X with PI user automatically, you can keep this and indeed start / load XBMC quick and easy way - even jumping back and forth between XBMC and you X Desktop. We can click "Logout", put "xbian" as user to log in and XBMC starts. If you quit XBMC, X will come again. No reboots needed.
Both X and XBMC can be managed now from xbian-config tool (from SSH or even XBMC). There is how you RPI will boot if you select what service for autostart: -->



<!-- endofpost -->


### Not used with recent versions

[^Reaper]: http://f.cl.ly/items/0S1S1Y0B3Q241Z2F0z1j/Untitled.png

Previously useful functionality or workarounds


### Clear cached network adapter

(needed for switching cards between devices)

```sh
echo | sudo tee /etc/udev/rules.d/70-persistent-net.rules
```


### Regional system settings

If the console keyboard is not setup correctly for the en_GB layout, a solution I've used before is to install `keyboard-configuration`, which only seems to work once `console-setup` is also installed.

*The latter package will change the console font, but this can be reverted during the commands shown below. Just choose the font you prefer, or choose 'Do not change the boot/kernel font'.*

```sh
# Packages needed to change keyboard layout
apt-get install console-setup keyboard-configuration

# Configure keyboard settings
dpkg-reconfigure keyboard-configuration

# Configure console font
# accept the defaults until the font selection screen
dpkg-reconfigure console-setup

# Configure time zone and locale
dpkg-reconfigure tzdata
dpkg-reconfigure locales
```

