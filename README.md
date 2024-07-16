# Information about the MKS CANable 2.0 PRO

This is a compact and inexpensive isolated USB-to-CAN
adapter that supports CAN-FD and runs well with the
[candleLight](https://github.com/candle-usb/candleLight_fw) firmware.
With candleLight, it's plug-and-play with the Linux socketcan
infrastructure.

<https://github.com/makerbase-mks/CANable-MKS>

<https://makerbase3d.com/product/makerbase-canable-v2/>

Note: The "CANable 2.0 PRO" has isolation between the CAN and the USB,
the "CANable 2.0" (non-PRO) has no isolation.  This repo focuses on the
isolated PRO version.


# Firmware update

The device ships with
[firmware](https://github.com/normaldotcom/canable2-fw) suitable for
use with [slcand](https://github.com/linux-can/can-utils/).


## One-time setup - compile the firmware

Do this once, then keep the firmware image around to flash as many
devices as you want.

1. Clone the firmware source repo: `git clone https://github.com/pgreenland/candleLight_fw.git`

2. `cd candleLight_fw`

3. Check out the `multichannel` branch (there is a
[PR](https://github.com/candle-usb/candleLight_fw/pull/176) to merge
this into mainline but it hasn't landed yet): `git checkout multichannel`

4. Install the build dependencies: `sudo apt install gcc-arm-none-eabi`

5. Build the firmware:
```
mkdir build
cd build
cmake .. -DCMAKE_TOOLCHAIN_FILE=../cmake/gcc-arm-none-eabi-8-2019-q3-update.cmake
make CANable2_MKS_fw
```

If that all works, you never have to do it again.


## Flash the device

1. Unplug from USB, install a jumper at BOOT, plug back in to USB.
Run `lsusb`, it should show up like this:
```
  Bus 001 Device 101: ID 0483:df11 STMicroelectronics STM Device in DFU Mode
```

2. Flash the firmware.  Make sure you're in the `candleLight_fw/build`
directory where you originally built the firmware image above, then run:
`make flash-CANable2_MKS_fw`.  This may print an error, `dfu-util:
can't detach`, but that's ok (see below).

3. Unplug from USB, remove BOOT jumper, re-connect USB.  `lsusb` should
show this:
```
  Bus 001 Device 102: ID 1d50:606f OpenMoko, Inc. Geschwister Schneider CAN adapter
```

At this point you can inspect and configure the CAN interface using the normal tools:

```
$ ip link show can0
20: can0: <NOARP,ECHO> mtu 16 qdisc noop state DOWN mode DEFAULT group default qlen 10 link/can
$ sudo ip link set can0 type can bitrate 500000
$ sudo ip link set can0 up
$ candump can0
  can0  401   [8]  03 00 89 3E 4C 26 38 47
  can0  440   [8]  00 18 70 39 01 26 38 47
  can0  441   [8]  E8 FF BB BD 13 26 38 47
  can0  400   [8]  00 50 F5 B8 38 3A 38 47
  can0  401   [8]  FD FF 88 3E 4A 3A 38 47
  can0  440   [8]  00 00 B7 36 8C 39 38 47
  can0  441   [8]  E8 FF BB BD 9D 39 38 47
  can0  1A4   [2]  00 01
```



# About that flashing error...

A hardware bug on the MKS CANable 2.0 PRO causes problems when rebooting
and resetting the board too quickly.  The problem is described here:
<https://github.com/makerbase-mks/CANable-MKS/issues/6>

If you don't mind leaving the board powered down for ~2 seconds whenever
you reset it, you can ignore this issue.

If you want the board to boot into the candleLight firmware every time it
resets, no matter how short the duration of the reset, you can reprogram
the Flash Option Bytes of the STM32 using these steps:

1. Install required programs (you only have to do this on each computer
you want to be able to flash): `sudo apt install openocd netcat-openbsd`

2. Connect a Single Wire Debug (SWD) probe to the CANable2 PRO board.
I use a Segger J-Link but any should work.
```
  J-Link    | CANable
  ----------+-------
  1 (Vtref) | 3.3V
  4 (GND)   | GND
  7 (SWDIO) | SWD
  9 (SWCLK) | SWC
```
![](/pics/j-link-connector.png)

3. Make sure the CANable2 is connected to USB.

4. Run OpenOCD (this will keep running until you stop it with
Ctrl-C): `openocd -f interface/jlink.cfg -c 'transport select swd'
-f target/stm32g4x.cfg`

5. In a different terminal, send the command list to the OpenOCD server:
`nc -N localhost 4444 < disable-boot-jumper.cfg`
