title:        Stellaris Virtual Serial Device
categories:   stellaris arm
date:         2012-12-10 02:48

The Stellaris [Launchpad] is able to provide a virtual serial port over the debug USB interface.  Data can be sent in either direction using this serial connection, to interact with program execution, or log it's output.

[Launchpad]: http://www.ti.com/ww/en/launchpad/stellaris_head.html

<!-- more -->


## device permissions

Root privileges are required to access the device on some Linux and Unix based systems, so remember to use `sudo` with commands such as `lm4flash` and `ttylog` which directly access the device.

Alternatively, on Linux you can configure permissions for the device using udev rules:

``` sh
	$ sudo nano /etc/udev/rules.d/42-sellaris.rules
	
	# add the following lines:
	ATTRS{product}=="In-Circuit Debug Interface", OWNER="devuser" KERNEL=="ttyACM?"
```
		
The (fairly open) rules shown above, give the user named '*devuser*' permission to access the launchpad's usb interface, along with it's virtual serial device.

Note: these rules should be more specific if you have similar devices that you don't wish to allow access to inadvertently.

``` sh
	# Exact match for Stellaris LM4F120XL
	SUBSYSTEMS=="usb", ATTRS{idVendor}=="1cbe", ATTRS{idProduct}=="00fd", MODE="0660", OWNER="devuser"
```

The device in this rule is strictly defined, and read/write permissions have been explicitly set for it.  Also, because the whole device is matched directly (rather than it's individual interfaces), only one rule is required.  You could also try assigning the device to a group, using `GROUP="devgroup"`.

To define these rules, the exact name, vendor, and/or product id needs to be known.  You can check these values using `dmesg` after connecting your device; the lines of interest are shown below.

``` sh
	$ dmesg
	# <snip>
	[39911.201497] usb 1-1.2.3: New USB device found, idVendor=1cbe, idProduct=00fd
	[39911.201537] usb 1-1.2.3: Product: In-Circuit Debug Interface
	[39911.201551] usb 1-1.2.3: Manufacturer: Texas Instruments
	[39911.344575] cdc_acm 1-1.2.3:1.0: ttyACM0: USB ACM device
```

## serial device

If you want to use the serial connection to provide input or commands to the launchpad, you'll need a terminal emulator that can connect to a serial device.

``` sh
	$ cd stellaris/boards/ek-lm4f120xl/qs-rgb
	$ make
	$ lm4flash gcc/qs-rgb.bin
	
	# Use of screen on linux
	$ screen /dev/ttyACM? 115200
	
	# Use of screen on mac os x
	$ screen /dev/tty.usbmodem* 115200
	
	# Kill the connection by typing: ^A k
	#  to list other commands, type: ^A ?
```

Use of the wildcard in `ttyACM?` allows for the times the device may get assigned a different number, e.g. when power cycling or reconnecting.  If you have another ttyACM device connected then you should probably use it's full name.

On OS X the device should be named similar to `tty.usbmodem`, with the serial number of the device appended.