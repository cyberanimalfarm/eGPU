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

If you receive depenency/version errors with the apt installation, check to make sure that you don't have additional CUDA repos in /etc/apt/sources.list.d

## Post-driver-install

NVIDIA open drivers currently (as of version 515 and 525) do not support commercial GTX/RTX video card models. Do enable support for these cards the 
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
This configuration requires the synaptics package to be installed. This can be installed with apt with:
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



---
[nvidia-conf]: https://github.com/cyberanimalfarm/eGPU/blob/master/etc/modprobe.d/nvidia.conf
[xorg-conf]: https://github.com/cyberanimalfarm/eGPU/tree/master/etc/X11/xorg.conf.d
[synaptics-conf]: https://github.com/cyberanimalfarm/eGPU/blob/master/etc/X11/xorg.conf.d/70-synaptics-options.conf
[input-conf]: https://github.com/cyberanimalfarm/eGPU/blob/master/etc/X11/xorg.conf.d/20-input.conf
