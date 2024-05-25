Attempt to create a minimal EDK2 for Exynos 9830 devices

## Status
Boots to UEFI Shell.

### Building
Tested on Ubuntu 22.04.

First, clone EDK2.

```
cd ..
git clone https://github.com/tianocore/edk2 --recursive --depth=1
git clone https://github.com/tianocore/edk2-platforms.git
```

You should have all three directories side by side.

Next, install dependencies:

22.04:

```
sudo apt install build-essential uuid-dev iasl git nasm python3-distutils gcc-aarch64-linux-gnu
```

Also see [EDK2 website](https://github.com/tianocore/tianocore.github.io/wiki/Using-EDK-II-with-Native-GCC#Install_required_software_from_apt)

## Tutorials

First run ./firstrun.sh

Then, ./build.sh.

This should make a boot.tar image to be flashed in ODIN, you may need to adjust.

## Porting this project to other Exynos SOCs

> [!NOTE]
> This may get you to somewhat boot EDK2, however it may not reach shell at all and may crash/get stuck somewhere.

### What you will need

- A phone with an Exynos SOC
- ADB Access
- Access to TWRP
- Android Image Kitchen
- Device Tree Compiler
- Your phone's screen resolution 

### Grabbing files off your phone

First, we will grab your boot image as in some cases the boot.img generated by the build system won't boot.

To do this, in TWRP do the following command

```adb shell "dd if=/dev/block/by-name/boot of=/tmp/boot.img```

Now grab the file onto your PC

```adb pull /tmp/boot.img```

We also need the information about your phone's memory allocations so we can find where to write stuff, like enabling framebuffer.

To get it, in TWRP do the following command:
```adb pull /proc/iomem```

Finally we need your phones DTB (Device Tree Blob)

To get it, do the following command in TWRP:
```adb pull /sys/firmware/fdt```

###  Taking notes of what is needed

First we will need the memory base, to find it open up the iomem you dumped and look for the "System RAM" segment, specifically the first one that comes up, We will assume ```80000000-baafffff : System RAM``` as an example. You want to take note of the first part, which in this case is 80000000.

Next we need the decon area, this is needed for framebuffer, in the iomem dump look for the "decon_f" segment, We will assume ```19050000-1905ffff : decon_f@0x19050000``` to be decon as an example, just as before take note of the first part, which in this case is 19050000

Now we're gonna decompile your DTB into a DTS (Device Tree Source), to do this do the following command ```dtc -I dtb -O dts -o devicetree.dts fdt```

Now open devicetree.dts in a Text Editor, we're gonna look for the interrupt-controller node, again we will assume the interrupt-controller here to be

```
  interrupt-controller@10100000 {
    compatible = "arm,cortex-a15-gic\0arm,cortex-a9-gic";
    #interrupt-cells = <0x03>;
    #address-cells = <0x00>;
    interrupt-controller;
    reg = <0x00 0x10101000 0x1000 0x00 0x10102000 0x1000 0x00 0x10104000 0x2000 0x00 0x10106000 0x2000>;
    interrupts = <0x01 0x09 0xf04>;
    phandle = <0x01>;
  };
```

You want to take notes of 2 numbers, the first one being after the "interrupt-controller@", which in this case is 10100000, and the second one being after "reg = <reg 0x00 firstnum 0x1000 0x00" which in this case is 10102000.

The final thing to look for is your bootloader framebuffer address, to find this do the following command while your device is in TWRP ```cat /proc/cmdline``` and look for "s3cfb.bootloaderfb=" or anything along the lines of that, as an example we will assume ```s3cfb.bootloaderfb=0xf1000000```, take note of the address after the "=", which in this case is 0xf1000000.

Now you can proceed.

### Making the DSC File

> [!NOTE]
> You shouldn't make the device use all the ram yet as it may not boot, stick eith 1.5GB or smaller to be safe.

Go into EXYNOS9830Pkg/Devices and copy a10.dsc to (devicename).dsc, for example we'll copy a10.dsc into s20.dsc

Now theres only a couple things to edit here thankfully.

First thing to edit is ```gArmTokenSpaceGuid.PcdSystemMemoryBase```, replace "0x80000000" or whatever address it holds with 0x(Number of the memory base you noted down earlier)

Final things to edit is the framebuffer area, to avoid boring you with more text ill just put the things needing to be edited.

  ```gEXYNOS9830PkgTokenSpaceGuid.PcdMipiFrameBufferAddress|(Framebuffer offset from earlier)```
  
  ```gEXYNOS9830PkgTokenSpaceGuid.PcdMipiFrameBufferWidth|(Screen resolution width)```
  
  ```gEXYNOS9830PkgTokenSpaceGuid.PcdMipiFrameBufferHeight|(Screen resolution height)```

  ```gEXYNOS9830PkgTokenSpaceGuid.PcdMipiFrameBufferVisibleWidth|(Screen resolution width)```
  
  ```gEXYNOS9830PkgTokenSpaceGuid.PcdMipiFrameBufferVisibleHeight|(Screen resolution height)```

After you've edited that you can save the file and close it.

### Correcting the interrupt controller addresses

Open up EXYNOS9830Pkg/EXYNOS9830.dsc and replace ```gArmTokenSpaceGuid.PcdGicDistributorBase|0x12301000``` with ```gArmTokenSpaceGuid.PcdGicDistributorBase|0x(First interrupt address you noted)``` after which  replace ```gArmTokenSpaceGuid.PcdGicInterruptInterfaceBase|0x12302000``` with ```gArmTokenSpaceGuid.PcdGicInterruptInterfaceBase|0x(Second interrupt address you noted)```

You can now save and close the file.

### Correcting decon address

Go to https://godbolt.org/ and before doing anything chabge the compiler on the right to ARM64 GCC 5.4.

Then in the left paste

```
void enableDecon()
{
	*(int*) (0x(decon_f noted from before) + 0x70) = 0x1281;
}
```

The assembly you'll want is in the middle.


Go to EXYNOS9830Pkg/Library
/EXYNOS9830PkgLib and open up "EXYNOS9830PkgHelper.S"

Replace the ```enableDecon:``` function with the one you got from the online compiler (except for the last ret) then close and save the file.

### Editing the build script

Open build.sh

Replace any mention of a10.dsc with (Device name).dsc

Now you can build with the steps above.

### Running EDK2 in your device

After your device builds don't use the boot.img in workspace as the bootloader may not like it, instead take the boot.img you dumped earlier and place it into Android Image Kitchen, unpack it with ```./unpackimg.sh boot.img``` then take the "UEFI" file from the workspace folder and put it into split_image under the name "boot.img-kernel", obviously replacing the old one, repack the image with ```./repackimg.sh```.

Rename image-new.img to boot.img, then using 7zip (on windows) or tar on linux, archive it to a ``.tar``.

You can now flash boot.tar onto your phone using ODIN3, Heimdall, or TWRP if you prefer.

### Getting EDK2 shell fullscreen

If you made it this far, congratulations but you're here to get the shell fullscreen not to get some kind of trophy. To make the shell fullscreen open, up ```EXYNOS9830Pkg/Drivers/GraphicsConsoleDxe/GraphicsConsole.c```, go to line 293,

```NewModeBuffer[ValidCount].Columns``` should equal your devices resolution width divided by 8 and ```NewModeBuffer[ValidCount].Rows``` should equal your devices resolution height divided by 19, obviously if any of the divisions have remainders (e.g 23.4) go down to the lowest whole number to be safe (e.g 23)


# Credits

SimpleFbDxe screen driver is from imbushuo's [Lumia950XLPkg](https://github.com/WOA-Project/Lumia950XLPkg).

Zhuowei for making edk2-pixel3

All the people in ``EDKII pain and misery, struggles and disappointment`` on Discord.
