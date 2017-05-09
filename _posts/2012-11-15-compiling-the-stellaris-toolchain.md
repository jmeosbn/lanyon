title:        Compiling the Stellaris Toolchain
categories:   stellaris arm
date:         2012-11-15 20:02

If you have a Stellaris [Launchpad] - and don't want to use the [official] tools - you can mostly follow the instructions at [y3xz] to build your own toolchain on any Unix/Linux based system using the [ARM EABI Toolchain Builder][EABI].  This includes Mac OS X, but I ran into a couple of minor issues as listed below.

<!-- more -->

*(If you don't fancy building the toolchain, [yagarto] offers recent binaries for Windows and Mac OS X that target ARM devices.)*

[Stellaris]: http://www.ti.com/product/lm4f120h5qr
[Launchpad]: http://www.ti.com/ww/en/launchpad/stellaris_head.html
[y3xz]: http://blog.y3xz.com/blog/2012/10/29/an-open-toolchain-for-the-ti-stellaris/
[official]: http://www.ti.com/tool/SW-EK-LM4F120XL
[EABI]: https://github.com/jsnyder/arm-eabi-toolchain
[outdated]: https://github.com/unhandledexception/armx/wiki
[yagarto]: http://www.yagarto.de/

Note, the libraries included with the Codesourcery Lite [toolchain][mentor] used here [do not support][libraries] the hardware FPU of the ARM [Cortex-M4F], using software floating point code instead.  There is a [hardfloat-toolchain] builder (which I've not used yet), and ARM is maintaining a [GCC toolchain][launchpad.net] targeting embedded ARM processors, which I'll probably try building next.

For more details on the launchpad, and the various libraries etc. that it uses, T.I. has a selection of [technical documentation][Stellaris] on their site.

[^softlib]: The compiler supports soft-float, VFP hard-float using the soft-float ABI, and VFP hard-float using the VFP ABI. The included libraries are only compiled with soft floating point support, and are link-compatible with VFP hard-float code only when using the soft-float ABI; linking with code that uses the VFP ABI will fail.

[libraries]: https://sourcery.mentor.com/GNUToolchain/release2322?@template=datasheet
[Cortex-M4F]: http://en.wikipedia.org/wiki/ARM_Cortex-M#Cortex-M4
[launchpad.net]: https://launchpad.net/gcc-arm-embedded/+download
[hardfloat-toolchain]: https://github.com/prattmic/arm-cortex-m4-hardfloat-toolchain


## toolchain

The makefile failed to download the source archive, so [download] it manually into the root of the toolchain repo.  Make will continue so long as the filename and checksum matches.

[mentor]: http://www.mentor.com/embedded-software/sourcery-tools/sourcery-codebench/editions/lite-edition

[download]: https://sourcery.mentor.com/GNUToolchain/package10384/public/arm-none-eabi/arm-2012.03-56-arm-none-eabi.src.tar.bz2

[mylink]: https://sourcery.mentor.com/GNUToolchain/subscription3053?lite=arm&lite=ARM&signature=4-1352914385-0-81777d693584c1d30acc48c7abaf41235a1766c3


## lm4tools

The `lm4flash` tool included in [recent] versions of [lm4tools] is unable to read the serial number of the device on OS X, so compiled code cannot be flashed to the launchpad device.

[recent]: https://github.com/utzig/lm4tools/commit/cc466b1

__Update: The dev has [committed] a workaround that fixes `lm4flash` on OS X.__

[committed]: https://github.com/utzig/lm4tools/commit/99d501b

``` sh
	$ ./lm4flash project0.bin
	Unable to get device serial number: LIBUSB_ERROR_OTHER
	Unable to find any ICDI devices
```

Newer versions of lm4tools require a kernel extension to be installed on OS X (this will prevent access to the virtual serial device), see the [issue] on github for more info.  The simplest workaround is to checkout and build commit [ea3c905], which doesn't check for the serial number.

Btw, you will get a similar error if your system requires root privileges to access the device directly over usb; try using `sudo` on linux/unix systems if you have issues.

[lm4tools]: https://github.com/utzig/lm4tools
[issue]: https://github.com/utzig/lm4tools/issues/8
[ea3c905]: https://github.com/utzig/lm4tools/commit/ea3c905
[cc49426]: https://github.com/utzig/lm4tools/commit/cc49426081


## more links

[Setting up the GCC ARM Toolchain](http://hertaville.com/2012/05/28/gcc-arm-toolchain-stm32f0discovery/) - focuses on using ARM's [GCC toolchain][launchpad.net] on Windows

[Testing the Stellaris Toolchain](/testing-the-stellaris-toolchain) - my overview for compiling and testing code on the device.

[^stlink]: An older [programming tool](https://github.com/texane/stlink) with some debugging info in the README that applies to lmicdi.

