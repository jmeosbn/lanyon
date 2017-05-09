title:        Home Theatre Pi (HTPi)
categories:   arm pi htpc
date:         2012-12-17 19:13

A collection of notes for setting up a clean image of [XBian](http://www.xbian.org) (1.0 Beta 1) on the Raspberry Pi.

<!-- more -->

Jump to [first login tasks](#first-login-tasks) if you have already set up terminal access to the Pi.

[^Todo]: Install fsck.ntfs (used to fix unclean unmount)

* TOC
{:toc}


## Basic Pi Tips


Guide for navigating XBMC with the [keyboard].


### Connecting to the Pi


XBian names the default user account ``xbian``, other distributions normally use ``pi``.  The default password is ``raspberry``.  XBian includes support for zeroconfig (via avahi-daemon), so it's easier to find your Pi on the network[^hostname].

```sh
# login using zeroconfig
ssh xbian@xbian.local
```

If zeroconfig can't be used, you can [find the Pi's network IP address](#find-the-ip-address) and use that to login.

```sh
# login using IP address
ssh xbian@192.168.1.10

# login as user pi on raspbian
ssh pi@192.168.1.10
```

*Note: If multiple Pis are active on the same network, they should be given [unique hostnames](#change-hostname).*


[^hostname]: It should be possible to connect with the hostname even without zeroconfig, e.g. ``xbian@xbian`` or ``pi@raspberrypi`` (on raspbian), but I've had no luck with this.

[keyboard]: http://wiki.xbmc.org/index.php?title=Keyboard


## Pre-setup; on PC, laptop, etc.

Before setting up the Pi remotely, there are some things to do locally to ease logging in when using SSH.  This will obviate the need to enter a password, or specify the full host name each time we access the Pi.


### Copy public key to Pi

If generating a new key pair, accept the default key location as suggested by ``ssh-keygen`` below.  While a passphrase is optional, anyone can use a copy of the unencrypted private key to authenticate with your identity.  Many operating systems are preconfigured to use ``ssh-agent`` or a similar utility, to avoid having to enter a passphrase multiple times (if at all).


```sh
# generate a key pair if none already exists
test -f ~/.ssh/id_rsa.pub || ssh-keygen

# remove any old, conflicting host entries
ssh-keygen -R xbian.local

# password is required until the key is installed
ssh-copy-id xbian@xbian.local
```

The easiest way to append a key to the remote user's `~/.ssh/authorized_keys` file is to use `ssh-copy-id` as shown above.  Download [ssh-copy-id] from source if your system doesn't already have it ([installation instructions]), this [alternative method] should also work in most situations.

[ssh-copy-id]: http://hg.mindrot.org/openssh/raw-file/tip/contrib/ssh-copy-id
[installation instructions]: http://www.commandlinefu.com/commands/view/10228/...if-you-have-sudo-access-you-could-just-install-ssh-copy-id-mac-users-take-note.-this-is-how-you-install-ssh-copy-id-
[alternative method]: http://www.commandlinefu.com/commands/view/188/copy-your-ssh-public-key-to-a-server-from-a-machine-that-doesnt-have-ssh-copy-id


### Add an alias to .ssh/config

Locally define the alias `xb`, to be used in place of `xbian@xbian.local` with commands such as `ssh xb` and `sftp xb`.  Enter the text below as a single command, or manually paste the quoted text into ``~/.ssh/config`` using ``nano`` or similar (the IP address can be used for Hostname if preferred).

```sh
echo '
Host xb
  User  xbian
  Hostname  xbian.local' >> ~/.ssh/config
```


## Transfer files

If you have previous files from your Pi stored locally, you can transfer them using `sftp`, `scp`, etc.  For easily transferring many arbitrary files , a GUI sftp client is recommended.

```sh
# connect to the pi, using the `xb` alias defined above
sftp xb
sftp> put .bash_aliases
sftp> exit

# you can also use ctlr+d to logout
```


## First login tasks

Most of the commands below need root privileges on the Pi, as they alter the system configuration.  To run a single command with root privileges, prepend the `sudo` command to it.  To run many commands this way without typing `sudo` each time, first start a root shell with `sudo -s`; *remember to logout with 'exit' or ctrl+d when finished*.

```sh
# Login to the pi, using the `xb` alias
ssh xb

# xbian-config may run here, set it up as you like then exit

# Disable autorun of xbian-config
# Use 'sudo xbian-config' to run manually
echo 0 > .xbian-config-start

# Disable the login message
touch ~/.hushlogin

# Disable setting of the terminal title
sed -i '/PS1=.*\][0-2];/s/^/##/' ~/.bashrc

# Use a root shell for the following commands
sudo -s

# Change password for root and xbian users
passwd && passwd xbian

# Allow full use of sudo without needing password
# note: XBian 1.0a4 made this much less essential
echo '%sudo  ALL=(ALL) NOPASSWD: ALL' >> /etc/sudoers

# Allow boot partition to be mounted
sed -i '/\s\/boot\s/s/,noauto//' /etc/fstab

# Update packages
apt-get update && apt-get upgrade

# Install some utilities and services
apt-get install p7zip zip curl mediainfo avahi-daemon iw

# Install gcc compiler, dev tools
apt-get install gcc make git-core

# Install scripting tools
apt-get install ruby ruby-dev ri libsqlite3-dev npm
# Fix node binary being named nodejs
cd /usr/bin; sudo ln -s nodejs node; cd

# Install rmate for TextMate editing over ssh
gem install rmate
```


## Change Hostname

If it's likely the Pi won't be the only one using the local network, then it should be given a unique host name.

```sh
nano /etc/hostname               # enter the desired name
nano /etc/hosts                  # replace the hostname
/etc/init.d/hostname.sh start    # to enable the changes
```


## Fake a hardware clock

The Pi doesn't have a real time clock, so it usually defaults to some point in the past until the time can be set correctly using the Internet.  To make the clock more consistent across power cycles, it can be initialised using the last recorded date and time.  *(note: previous distros required the [unabridged instructions](#fake-a-hardware-clock-unabridged).)*

```sh
apt-get install fake-hwclock
```


## Link settings to root profile

Occasionally you'll want to use a root shell, and then be annoyed that your aliases etc. are not configured for the root user.  You can either copy the profile files into ~/root or, as shown here, link them symbolically so that any future modifications will be reflected.

```sh
# change to the home folder you want to link from
cd ~xbian

# define a list of dot files to link
dotfiles=".profile .bashrc .bash_aliases .bash_logout .nanorc .toprc .gemrc"

# link each existing file into the root user's home folder
for i in $dotfiles; do [ -f $i ] && sudo ln -sfv ~+/$i ~root/; done

# hush login message
sudo touch ~root/.hushlogin
```


## Generate new RSA host keys

These keys confirm the identity of the Pi, to prevent a malicious host from intercepting the remote login process.  Not so important for your HTPi, but good standard security practice.  Also recommended if you have [more than one Pi](#change-hostname).

```sh
rm /etc/ssh/ssh_host_*
dpkg-reconfigure openssh-server

# remove old host key from clients using:
# ssh-keygen -R xbian.local
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


## Configure WiFi and Bluetooth

XBian now includes simple WiFi configuration as part of the xbian-config setup menu.  See the [manual configuration](#manually-configure-wifi-adapter) if you want finer control over the network settings (e.g. connecting to multiple networks).

My bluetooth adapter is not supported in the current build of Raspbian/XBian, but [here is the procedure](#configure-bluetooth-adapter) I used when trying to get it running (confirmed working on a laptop running Ubuntu).


## TTL serial console

The display of full screen terminal programs becomes corrupted when using a TTL to USB serial connection to the Pi from Mac OS X.  Changing the terminal type enables use of programs such as ``nano`` and ``xbian-config``.

```sh
sed -i '/ttyAMA0/s/vt100$/xterm/' /etc/inittab
```


## Build and install WiringPi

[WiringPi] is a library to access the Pi's GPIO, SPI, and I2C headers, modelled on the Arduino Wiring system.  It also includes the ``gpio`` utility for use of the libraries from the command prompt.

[WiringPi]: https://projects.drogon.net/raspberry-pi/wiringpi

```sh
apt-get install gcc make git-core libi2c-dev
git clone git://git.drogon.net/wiringPi
cd wiringPi; ./build

# test with gpio utility
gpio readall

# install ruby gem (optional)
gem install wiringpi
```


## Remove the desktop environment

If you only plan to run the terminal environment (including self-hosted programs such as XBMC), then a lot of space can be recovered by removing x11 and related packages.

```sh
# desktop
apt-get purge x11-common libx11-.* lxde-icon-theme xkb-data
apt-get purge fonts-freefont-ttf libraspberrypi0

# cleaning up
(cd ~pi; rm -rf Desktop python_games ocr_pi.png)
apt-get autoremove
apt-get clean
```

*Note: this task is not normally required for distros such as xbian.*


## Install Mono

Install Mono development tools, runtime, and interactive shell.

```sh
apt-get install mono-devel mono-csharp-shell
```


## Install aircrack and related tools

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


## Regional system settings

Both XBian and Raspbian have system setup menus, but neither seem to setup the console keyboard correctly for the en_GB layout.  The current solution I use is to install ``keyboard-configuration``, which only seems to work once ``console-setup`` is also installed.

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


## Install dbox (dropbox tool)

Follow the [dbox installation instructions][dbox] to set up the dropbox sdk developer keys and authorisation tokens.  To use dbox for automatic folder syncing, see my post: [Dropbox on Pi](/blog/pi-box/).

[dbox]: https://github.com/kenpratt/dbox


```sh
apt-get install gcc make ruby ruby-dev libsqlite3-dev
gem install dbox
```


## Setup XBMC


### XBMC Extensions

Use `wget` to download the latest versions from the links below, then open the zip files directly from xbmc's addons page.

- <http://code.google.com/p/xbmc-iplayerv2/downloads/list>
- <http://code.google.com/p/mossy-xbmc-repo/downloads/list>
- <http://code.google.com/p/xbmc-itv-player/downloads/list>


### XBMC Settings

If you are storing media in the root folder of an NTFS formatted hard drive, you may see some system folders while using the video/audio file views.  You can hide these system folders, change other advanced xbmc behaviour, or preset/lock standard settings in [advancedsettings.xml].

[advancedsettings.xml]: http://wiki.xbmc.org/?title=Advancedsettings.xml

```xml ~/.xbmc/userdata/advancedsettings.xml
<video>
    <!-- hide system folders from the video files view -->
    <!-- you could also add these to 'excludefromscan' -->
    <excludefromlisting>
        <regexp>(\$RECYCLE.BIN|System Volume Information)</regexp>
    </excludefromlisting>
</video>
<audio>
    <!-- do likewise for the audio files view -->
    <excludefromlisting>
        <regexp>(\$RECYCLE.BIN|System Volume Information)</regexp>
    </excludefromlisting>
</audio>
```


## Useful extras, not always used

Stuff used infrequently, or currently being tested


### Packages from Raspbian

Some standard packages that are usually excluded from the xbian distro, as they are not required for use of xbmc only.

```sh
apt-get install omxplayer

# included on xbian >= 1.0b
apt-get install psmisc usbutils
```


### Other useful packages

```sh
apt-get install fs2resize exfat-fuse
apt-get install clang geany
apt-get install sysv-rc-conf
```


### Find the IP address

You can get the IP address from your Pi, by running either of the following commands locally on the device.

```sh
ip r | grep -o 'src.*'

ifconfig 2>&1 | grep cast | grep -o 'inet [^ ]*'
```

Connect to the address that has the same subnet (starts similar) as the IP address you will be connecting from, ignoring the localhost address (127.0.0.1).

If it isn't possible to run a command locally on the Pi (e.g. there is no monitor or keyboard attached), you can either [scan the network][findPi script], run `nmap 192.168.0.1/24 -p 22`, or view 'Attached Devices' in your Router's setup.  Look for a matching hostname or MAC address (which will start with ``b8:27:eb`` for the on-board LAN).

[findPi script]:http://www.recantha.co.uk/blog/?p=2397


### Manually configure WiFi adapter

[Instructions adapted from here](http://www.savagehomeautomation.com/raspi-airlink101).

Use ``lsusb`` to check that the adapter is recognised, and ``lsmod`` to check the kernel module (e.g. ``8192cu``) is loaded.

```sh
sudo nano /etc/network/interfaces
```

Make sure the following lines exist in the interfaces file, adding them as needed:

```
allow-hotplug wlan0
iface wlan0 inet manual
wpa-roam /etc/wpa_supplicant/wpa_supplicant.conf
```

You may have the line ``wireless-power off`` in this file, which relates to power ***management*** only.  I've commented it out as it resulted in errors logged during ``ifup`` and power management remained off without it.

```sh
sudo nano /etc/wpa_supplicant/wpa_supplicant.conf
```

Add your network details to wpa_supplicant.conf, using the following template:

```
network={
  ssid="YOUR-NETWORK-SSID"
  proto=WPA2
  key_mgmt=WPA-PSK
  pairwise=CCMP TKIP
  group=CCMP TKIP
  psk="YOUR-WLAN-PASSWORD"
}
```

Reinitialise the adapter, and check it's connected.

```sh
sudo ifdown wlan0
sudo ifup wlan0
# you may get some errors here, even when successful
```

Use ``iwconfig`` to view wifi adapter info and ``ifconfig`` for general network info.


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

**Note: a bluetooth adapter may be listed in ``lsusb`` and ``hciconfig``, without being recognised by ``hcitool``. This is the case with the belkin dongle I have, so use ``hcitool`` to check that a device is working properly.*


### Testing PVR

```sh
apt-get install vdr-plugin-dvbsddevice
```


### Setup webcam

Use 'motion' or 'fswebcam', motion may need a default cfg copying

```sh
apt-get install motion
cp /etc/default/motion /etc/motion/motion.conf
```


## Troubleshooting and backup

Some useful commands and procedures


### Quick Tips

* You can detect hdmi audio modes: `/opt/vc/bin/tvservice -a`
* Setup CEC remote over hdmi from console: `cec-config`


### Backup settings

- Settings, addons etc. are in ~/.xbmc
- .xbmc/userdata - preferences etc
- .xbmc/addons - binaries, themes
- .xbmc/addons/packages - original downloads, can use with "install from zip"

```sh
# backup profile settings
zip -FS -ry xbmc .xbmc/
zip -FS -ry dotfiles .bash_aliases .nanorc .toprc .ssh

# backup system config files
sudo zip -FS -ry basecfg /etc/wpa_supplicant/wpa_supplicant.conf
```

Or using tar..

```sh
# backup profile settings
tar -czf xbmc-backup.tar.gz .xbmc

# restore profile settings
sudo initctl stop xbmc
tar -xzf xbmc-backup.tar.gz
sudo initctl start xbmc
```


<!-- endofpost -->


## Not used with recent versions

[^Reaper]: http://f.cl.ly/items/0S1S1Y0B3Q241Z2F0z1j/Untitled.png

Previously useful functionality or workarounds


### Clear cached network adapter

(needed for switching cards between devices)

```sh
echo | sudo tee /etc/udev/rules.d/70-persistent-net.rules
```


### Fake a hardware clock (unabridged)

More complicated instructions, as used on previous versions of XBian.

```sh
touch /etc/init.d/hwclock.sh
/etc/init.d/ntp restart
apt-get install ntpdate fake-hwclock
ntpdate-debian
dpkg-reconfigure tzdata
sed -i 's/^exit 0/ntpdate-debian\nexit 0/g' /etc/rc.local
```


### Fix ssh access using public key

```sh
# must be owned by root
chown root: ~ ~/.ssh
# no write for others
chmod a=rx,u+w  ~

# no access for others
chmod -R go-rwx ~/.ssh
# public key can be readable
chmod a+r ~/.ssh/id_rsa.pub
```


### Download OpenSSH sftp server

If sftp is not already on your system (such as when using dropbear), it can't be installed
manually without installing the entire openssh package.

```sh
apt-get download openssh-server
dpkg-deb -X openssh-server_*.deb sftp
cp sftp/usr/lib/openssh/sftp-server /usr/lib/
rm -r sftp openssh-server_*.deb
```


### Update firmware without kernel

```sh
wget http://goo.gl/1BOfJ -O /usr/bin/rpi-update && chmod +x /usr/bin/rpi-update
SKIP_KERNEL=1 rpi-update 128
```


### Allow XBMC to unmount USB drives

XBian used to include the usbmount package to mount USB devices as soon as they are connected.  This prevented XBMC from bring able to use the udisk service to mount and unmount USB drives itself, due to root privileges being required to unmount devices mounted by usbmount.

```sh
# disable the usbmount package
sed -i '/ENABLED=/s/=1/=0/' /etc/usbmount/usbmount.conf

# optionally remove unused usbmount directories
# umount /media/usb*; rmdir /media/usb*; rm /media/usb
```

Drives can be unmounted manually using ``udisks`` without needing to be root, and members of the ``plugdev`` group can also use ``pumount``.


### Install Shairport

Instructions found [here](http://tomsolari.id.au/post/27169019561/airplay-music-streaming-on-raspberry-pi) (alt site [here](http://cheeftun.appspot.com/trouch.com/2012/08/03/airpi-airplay-audio-with-raspberry/))

More recent instructions: http://lifehacker.com/5978594/turn-a-raspberry-pi-into-an-airplay-receiver-for-streaming-music-in-your-living-room

A change in IOS 6 [requires Perl Net-SDP](http://jordanburgess.com/post/38986434391/raspberry-pi-airplay) module to installed.

```sh
git clone https://github.com/njh/perl-net-sdp.git perl-net-sdp
cd perl-net-sdp
perl Build.PL
sudo ./Build
sudo ./Build test
sudo ./Build install
cd ..
```

```sh
# do as root
sudo -s
apt-get install alsa-utils
modprobe snd_bcm2835
# optionally set to headphone output
# amixer cset numid=3 1
# optionally restore to hdmi output
# amixer cset numid=3 2
apt-get install build-essential libssl-dev libcrypt-openssl-rsa-perl libao-dev libio-socket-inet6-perl libwww-perl avahi-utils pkg-config
wget https://github.com/albertz/shairport/zipball/master
unzip master
cd albertz-shairport-*
make install
cp shairport.init.sample /etc/init.d/shairport
# add to start of shairport: modprobe snd_bcm2835
nano /etc/init.d/shairport
# optionally edit name of service (remove port number):
nano /usr/local/bin/shairport.pl
insserv shairport
# manually start [services](http://pi-raspberry.blogspot.co.uk/2012/08/shairport-raspberry-pi.html)
service avahi-daemon start
/etc/init.d/shairport start
# exit root
exit
```
