# Installation Instructions

This repo stores configuration files to use the Aorus RTX 3080 external GPU. The sample configuration files are located in the same directory structure 
as they are found in Ubuntu 22.04 LTS. This configuration has been tested on the XPS 15 9510 but it can be adapted for other hardware.

## Pre-Requisites

This configuration is intended for use with the NVIDIA open kernel driver. 
Currently version 515 is recommended for use. This can be installed with:

```shell
sudo apt install nvidia-driver-515-open
```
#### Gotchas

If you receive dependency/version errors with the apt installation, check to make sure that you don't have additional CUDA repos in /etc/apt/sources.list.d. I recommend you check apt and dpkg to fully remove any nvidia packages before installing the driver in a fresh state.

## Post-driver-install

NVIDIA open drivers currently (as of version 515 and 525) do not support commercial GTX/RTX video card models. To enable support for these cards the 
following option must be placed in /etc/modprobe.d/[nvidia.conf][nvidia-conf]:

```shell
options nvidia NVreg_OpenRmEnableUnsupportedGpus=1
```
Once the driver is loaded, nvidia-smi should produce a similar output:
```shell
nvidia-smi

+-----------------------------------------------------------------------------+
| NVIDIA-SMI 515.86.01    Driver Version: 515.86.01    CUDA Version: 11.7     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  NVIDIA GeForce ...  Off  | 00000000:3F:00.0  On |                  N/A |
| 30%   29C    P8    33W / 320W |    495MiB / 10240MiB |     12%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
```
This GPU should now be available for compute applications without further configuration. 

To use the GPU for display output and 3D acceleration with applications, additional configuration is necessary.

## X Configuartion

The Xorg server should be configured using configuration files placed in the [/etc/X11/xorg.conf.d][xorg-conf] directory

The [70-synaptics-options.conf][synaptics-conf] file provides configuration for the Synaptics touchpad:
```shell
Section "InputClass"
        Identifier "touchpad catchall"
        Driver "synaptics"
        MatchIsTouchpad "on"
        MatchDevicePath "/dev/input/event*"

        # Set natural scrolling
        Option "VertScrollDelta" "-100"

        # Set palm detection
        Option "PalmDetect" "1"
EndSection
```
This configuration requires the synaptics package to be installed. This can be installed with:
```shell
sudo apt install synaptic
```
The [20-input.conf][input-conf] file provides configuration for keyboard and mouse inputs to the Xorg server:
```shell
Section "InputDevice"
    Identifier     "Mouse0"
    Driver         "mouse"
    Option         "Protocol" "auto"
    Option         "Device" "/dev/psaux"
    Option         "Emulate3Buttons" "no"
    Option         "ZAxisMapping" "4 5"
EndSectionc

Section "InputDevice"
    Identifier     "Keyboard0"
    Driver         "kbd"
    Option         "Device" "/dev/input/event2"
    Option         "XkbLayout" "us"
    Option         "XkbModel"  "pc104"
EndSection

Section "ServerLayout"
    Identifier    "layout"
    InputDevice   "Keyboard0" "CoreKeyboard"
    InputDevice   "Mouse0" "CorePointer"
EndSection
```

If event2 is not the correct device file for your keyboard, you can find the right keyboard file with:
```shell
cat /proc/bus/input/devices | grep -i "keyboard" -A5

N: Name="AT Translated Set 2 keyboard"
P: Phys=isa0060/serio0/input0
S: Sysfs=/devices/platform/i8042/serio0/input/input2
U: Uniq=
H: Handlers=sysrq kbd event2 leds
B: PROP=0
```
The [10-nvidia-optimus.conf][nvidia-optimus-conf] file provides a configuration for the integrated intel graphics and NVIDIA 3050 Ti that are internal to the XPS 9510. This configuration allows X to render through the integrated intel graphics to the laptop screen, but also allows 3D acceleration on demand with the descrete 3050 GPU. This can be managed with the NVIDIA prime-select tool. If prime select is not installed it can be installed with:

```shell
sudo apt install nvidia-prime
prime-select query
on-demand
```

```shell
# Configuration for discrete RTX 3050 used with prime render offload
Section "Device"
    Identifier    "RTX3050"
    Driver        "nvidia"
    BusID         "PCI:1:0:0"
    Option        "AllowEmptyInitialConfiguration"
    #Option        "UseEDID" "false"
    Option        "PrimaryGPU" "yes"
EndSection

Section "Device"
    Identifier    "iGPU"
    Driver        "modesetting"
    BusID         "PCI:0:2:0"
    #Option        "AccelMethod" "sna"
    #Option        "DRI" "3"
EndSection

Section "Screen"
    Identifier    "iGPU"
    Device        "iGPU"
EndSection

Section "ServerLayout"
    Identifier "layout"
    Option     "AllowNVIDIAGPUScreens"
    #Option     "AutoAddGPU" "false"
    Screen 0   "iGPU"
EndSection
```
Note: the PCI BusIDs may be different on your system. To check the IDs on your system, run:
``` shell
lspci | grep -i 'vga\|3d'
00:02.0 VGA compatible controller: Intel Corporation TigerLake-H GT1 [UHD Graphics] (rev 01)
01:00.0 3D controller: NVIDIA Corporation GA107M [GeForce RTX 3050 Ti Mobile] (rev a1)
```
While lspci gives the ID output in hexadecimal, the Xorg config uses decimal. For example, if your lspci output was 01:1A.0, the Xorg config would use PCI :1:26:0.

The [10-nvidia-egpu.conf][nvidia-egpu-conf] file provides a configuration for the Aorus RTX 3080 external GPU. This configuration will render the Xorg server on the external displays attached to the eGPU unit. This configuration uses the eGPU as the sole GPU on the system, so nothing will be rendered on the laptop display and internal GPUs will not be utilized. Note: the BusID on this configuration is PCI:63:0:0 which is listed as 3f:0.0 in the hexadecimal lspci output. 

``` shell
Section "Monitor"
    Identifier     "Monitor0"
    VendorName     "Unknown"
    ModelName      "Samsung C34H89x"
    HorizSync       30.0 - 152.0
    VertRefresh     50.0 - 100.0
    Option         "DPMS"
EndSection

Section "Device"
    Identifier     "Device0"
    Driver         "nvidia"
    VendorName     "NVIDIA Corporation"
    BoardName      "NVIDIA GeForce RTX 3080"
    BusID          "PCI:63:0:0"
EndSection

Section "Screen"
    Identifier     "Screen0"
    Device         "Device0"
    Monitor        "Monitor0"
    DefaultDepth    24
    Option         "Stereo" "0"
    Option         "nvidiaXineramaInfoOrder" "DFP-0"
    Option         "metamodes" "DP-0: nvidia-auto-select +0+0, HDMI-0: nvidia-auto-select +3440+0"
    Option         "SLI" "Off"
    Option         "MultiGPU" "Off"
    Option         "BaseMosaic" "off"
    SubSection     "Display"
        Depth       24
    EndSubSection
EndSection

Section "ServerLayout"
    Identifier     "Layout"
    Screen      0  "Screen0" 0 0 
    InputDevice    "Keyboard0" "CoreKeyboard"
    InputDevice    "Mouse0" "CorePointer"
    Option         "Xinerama" "0"
EndSection
```
This configuration was created with two monitors attached to the eGPU on DP-0 and HDMI-0 respectively. The:
```shell
Option         "metamodes" "DP-0: nvidia-auto-select +0+0, HDMI-0: nvidia-auto-select +3440+0"
```
Sets the display port monitor as the left-most display, and places the screen attached to HDMI-0 3440 pixels to the right of it. The Xinerama option allows X to use both monitors as a single "extended desktop" type configuration. Xorg should be able to autodetect your displays, but you can delete or modify these options appropriately.

To prevent Linux from loading the NVIDIA driver for the integrated 3050 TI, I modified the GRUB bootloader options to pass the 3050 to the VFIO driver. This will prevent any application from attempting to use it for display output and force the eGPU to be used for acceleration. This can also be used to [pass through the GPU to a Windows VM][gpu-passthrough] which I have successfully implemented. 

These changes are made in the /etc/default/[grub][grub-default] configuration file:
```shell
# If you change this file, run 'update-grub' afterwards to update
# /boot/grub/grub.cfg.
# For full documentation of the options in this file, see:
#   info -f grub -n 'Simple configuration'

GRUB_DEFAULT="0"
GRUB_TIMEOUT_STYLE="hidden"
GRUB_TIMEOUT=10
GRUB_DISTRIBUTOR="`lsb_release -i -s 2> /dev/null || echo Debian`"
GRUB_CMDLINE_LINUX_DEFAULT="intel_iommu=on kvm.ignore_msrs=1 vfio-pci.ids=10de:25a0"
GRUB_CMDLINE_LINUX=""

# Uncomment to enable BadRAM filtering, modify to suit your needs
# This works with Linux (no patch required) and with any kernel that obtains
# the memory map information from GRUB (GNU Mach, kernel of FreeBSD ...)
#GRUB_BADRAM="0x01234567,0xfefefefe,0x89abcdef,0xefefefef"

# Uncomment to disable graphical terminal (grub-pc only)
#GRUB_TERMINAL="console"

# The resolution used on graphical terminal
# note that you can use only modes which your graphic card supports via VBE
# you can see them in real GRUB with the command `vbeinfo'
#GRUB_GFXMODE="640x480"

# Uncomment if you don't want GRUB to pass "root=UUID=xxx" parameter to Linux
#GRUB_DISABLE_LINUX_UUID="true"

# Uncomment to disable generation of recovery mode menu entries
#GRUB_DISABLE_RECOVERY="true"

# Uncomment to get a beep at grub start
#GRUB_INIT_TUNE="480 440 1"
```
The key parameter here is: 
```shell
GRUB_CMDLINE_LINUX_DEFAULT="intel_iommu=on kvm.ignore_msrs=1 vfio-pci.ids=10de:25a0"
```
vfio-pci.ids=10de:25a0 passes the PCI ID of the 3050 TI to the vfio driver. To find the correct ID on your system run:
```shell
lspci -nn | grep -i 'vga\|3d'
00:02.0 VGA compatible controller [0300]: Intel Corporation TigerLake-H GT1 [UHD Graphics] [8086:9a60] (rev 01)
01:00.0 3D controller [0302]: NVIDIA Corporation GA107M [GeForce RTX 3050 Ti Mobile] [10de:25a0] (rev a1)
```
Finally, to automate switching back and forth between using the eGPU as a docking station that renders to external displays, and using my Laptop with integrated NVIDIA graphics, I created the [egpu-config script][egpu-script]:
```shell
#! /bin/bash
# This script swaps out X configurations between the built in GPU on the laptop and an eGPU configuration
# The internal GPU is passed to the VFIO driver before boot so that programs to not attempt to use it

function usage {
	echo "egpu-config --enable  or -e  enables the eGPU configuration"
	echo "egpu-config --disable or -d  disables the eGPU configuration"
	echo "-h or --help displays this message"
	echo "Note: egpu-config must be run as root"
}

if [[ $EUID -ne 0 ]]; then
	echo "This script must be run as root"
	exit 1
fi

if [[ $# -eq 0 ]]; then
	usage
	exit 1
fi 

if [[ $# -gt 1 ]]; then
	usage
	exit 1
fi

while [[ $# -gt 0 ]] 
do
args="$1"

case $args in
	-e|--enable)
	ENABLE="true"
        shift # past argument
	shift # past value
	;;
        -d|--disable)
	DISABLE="true"
        shift # past argument
        shift # past argument	
        ;;
        *) # Unknown option or help
	USAGE="true"
	shift # past argument
	;;
esac
done

set -- "${POSITIONAL[@]}" # restore positional parameters

if [[ $ENABLE == "true" ]]; then
    cp /etc/default/grub-3050-VFIO /etc/default/grub
    mv /etc/X11/xorg.conf.d/10-nvidia-egpu.disabled /etc/X11/xorg.conf.d/10-nvidia-egpu.conf
    mv /etc/X11/xorg.conf.d/10-nvidia-optimus.conf /etc/X11/xorg.conf.d/10-nvidia-optimus.disabled
    update-grub
fi

if [[ $DISABLE == "true" ]]; then
    cp /etc/default/grub.bak /etc/default/grub
    mv /etc/X11/xorg.conf.d/10-nvidia-egpu.conf /etc/X11/xorg.conf.d/10-nvidia-egpu.disabled
    mv /etc/X11/xorg.conf.d/10-nvidia-optimus.disabled /etc/X11/xorg.conf.d/10-nvidia-optimus.conf
    update-grub
fi

if [[ $USAGE == "true" ]]; then
	usage
fi

exit 0
```
Install the script with:
```shell
sudo cp egpu-config /usr/bin/
sudo chmod +x /usr/bin/egpu-config
```
Usage is simple:
```shell
sudo egpu-config --enable
```
Enables the egpu configuration (reboot is required)
```shell
sudo egpu-config --disable
```
Disables the egpu configuration (reboot is required)


---
[nvidia-conf]: https://github.com/cyberanimalfarm/eGPU/blob/master/etc/modprobe.d/nvidia.conf
[xorg-conf]: https://github.com/cyberanimalfarm/eGPU/tree/master/etc/X11/xorg.conf.d
[synaptics-conf]: https://github.com/cyberanimalfarm/eGPU/blob/master/etc/X11/xorg.conf.d/70-synaptics-options.conf
[input-conf]: https://github.com/cyberanimalfarm/eGPU/blob/master/etc/X11/xorg.conf.d/20-input.conf
[nvidia-optimus-conf]: https://github.com/cyberanimalfarm/eGPU/blob/master/etc/X11/xorg.conf.d/10-nvidia-optimus.conf
[nvidia-egpu-conf]: https://github.com/cyberanimalfarm/eGPU/blob/master/etc/X11/xorg.conf.d/10-nvidia-egpu.disabled
[gpu-passthrough]: https://mathiashueber.com/windows-virtual-machine-gpu-passthrough-ubuntu/
[grub-default]: https://github.com/cyberanimalfarm/eGPU/blob/master/etc/default/grub-3050-VFIO
[egpu-script]: https://github.com/cyberanimalfarm/eGPU/blob/master/usr/bin/egpu-config
