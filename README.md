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

  Bus 001 Device 101: ID 0483:df11 STMicroelectronics STM Device in DFU Mode

3. Clone <https://github.com/pgreenland/candleLight_fw.git>

4. Check out the `multichannel` branch

5. Build (FIXME)

6. Flash: `cd build; make flash-CANable2_MKS_fw`.  This will print an
error but that's ok.

7. Unplug, remove BOOT jumper, re-connect.  lsusb should show this:

  Bus 001 Device 102: ID 1d50:606f OpenMoko, Inc. Geschwister Schneider CAN adapter


# About that flashing error...
