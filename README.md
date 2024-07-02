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

To switch to candleLight:

1. Unplug from USB, install a jumper at BOOT, plug back in.  Run `lsusb`,
it should show up like this:
```
  Bus 001 Device 101: ID 0483:df11 STMicroelectronics STM Device in DFU Mode
```

3. Clone <https://github.com/pgreenland/candleLight_fw.git>

4. Check out the `multichannel` branch (there is a
[PR](https://github.com/candle-usb/candleLight_fw/pull/176) to merge
this into mainline but it hasn't landed yet).

5. Build per the instructions in the candleLight README

6. Flash: `make flash-CANable2_MKS_fw`.  This will print an error,
`dfu-util: can't detach`, but that's ok (see below).

7. Unplug, remove BOOT jumper, re-connect.  lsusb should show this:
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

1. Connect a Single Wire Debug (SWD) probe to the board.  I use a Segger
J-Link but any should work.
```
  J-Link    | CANable
  ----------+-------
  1 (Vtref) | 3.3V
  4 (GND)   | GND
  7 (SWDIO) | SWD
  9 (SWCLK) | SWC
```

2. Run OpenOCD: `openocd -f interface/jlink.cfg -c 'transport select swd'
-f target/stm32g4x.cfg`

3. Telnet to the OpenOCD server and issue these commands:
```
reset halt

# Read current option bytes from "Flash option register (FLASH_OPTR) 0x20".
mdw 0x40022020
# 0x40022020: ffeffcaa

# read "Flash status register (FLASH_SR) 0x10", make sure bit 16 ("BUSY")
# is cleared.
mdw 0x40022010
#0x40022010: 00000000

# Read "Flash control register FLASH_CR 0x14", see that bit 31 "LOCK"
# and bit 30 "OPT LOCK" are both set.
mdw 0x40022014
# 0x40022014: c0000000

# Write flash unlock keys to the "Flash key register (FLASH_KEYR) 0x08)"
# to unlock "Flash control register (FLASH_CR) 0x14".
mww 0x40022008 0x45670123
mww 0x40022008 0xCDEF89AB

# verify FLASH_CR 0x14 bit 31 is cleared
mdw 0x40022014
# 0x40022014: 40000000

# Write flash option unlock keys to the "Flash option key register
# (FLASH_OPTKEYR) 0x0c" to unlock option bytes.
mww 0x4002200c 0x08192A3B
mww 0x4002200c 0x4C5D6E7F

# verify FLASH_CR 0x14 bit 30 OPT LOCK is cleared
mdw 0x40022014
# 0x40022014: 00000000

# Write the new option bytes to the "Flash option register (FLASH_OPTR)
# 0x20".  This should be the original value (0xffeffcaa), but with
# these changes:
#
# nSWBOOT0 (bit 26):
#     0: FLASH_OPTR.nBoot0 selects boot target
#     1: PB8-BOOT0 selects boot target
#
# nBOOT0 (bit 27), nBOOT1 (bit 23):
#     1, X: flash ("fw app")
#     0, 1: System Memory (ST ROM bootloader)
#     0, 0: SRAM1 (dont use this)

# original default
# PB8-BOOT0 pin selects boot target, flash app or ST ROM bootloader
# nSWBOOT0=1, nBOOT0=1, bBOOT1=1
#mww 0x40022020 0xffeffcaa

# boot to firmware application in flash
# nSWBOOT0=0, nBOOT0=1, bBOOT1=1
mww 0x40022020 0xfbeffcaa

# boot to System Memory (ST ROM) bootloader
# nSWBOOT0=0, nBOOT0=0, bBOOT1=1
#mww 0x40022020 0xf3effcaa


# Set the Options Start bit OPTSTRT in the Flash control register (FLASH_CR)
mww 0x40022014 0x00020000

# read "Flash status register (FLASH_SR) 0x10", make sure bit 16 ("BUSY")
# is cleared.
mdw 0x40022010
#0x40022010: 00000000
```
