# CANBUS

This document describes Klipper's CAN bus support.

## Device Hardware

Klipper currently supports CAN on stm32, same5x, and rp2040 chips. In
addition, the micro-controller chip must be on a board that has a CAN
transceiver.

To compile for CAN, run `make menuconfig` and select "CAN bus" as the
communication interface. Finally, compile the micro-controller code
and flash it to the target board.

## Host Hardware

In order to use a CAN bus, it is necessary to have a host adapter.
There are currently two common options:

1. Use a
   [Waveshare Raspberry Pi CAN hat](https://www.waveshare.com/rs485-can-hat.htm)
   or one of its many clones.

2. Use a USB CAN adapter (for example
   [https://hacker-gadgets.com/product/cantact-usb-can-adapter/](https://hacker-gadgets.com/product/cantact-usb-can-adapter/)). There
   are many different USB to CAN adapters available - when choosing
   one, we recommend verifying it can run the
   [candlelight firmware](https://github.com/candle-usb/candleLight_fw).
   (Unfortunately, we've found some USB adapters run defective
   firmware and are locked down, so verify before purchasing.)

It is also necessary to configure the host operating system to use the
adapter. This is typically done by creating a new file named
`/etc/network/interfaces.d/can0` with the following contents:
```
auto can0
iface can0 can static
    bitrate 500000
    up ifconfig $IFACE txqueuelen 128
```

Note that the "Raspberry Pi CAN hat" also requires
[changes to config.txt](https://www.waveshare.com/wiki/RS485_CAN_HAT).

## Terminating Resistors

A CAN bus should have two 120 ohm resistors between the CANH and CANL
wires. Ideally, one resistor located at each the end of the bus.

Note that some devices have a builtin 120 ohm resistor (for example,
the "Waveshare Raspberry Pi CAN hat" has a soldered on resistor that
can not be easily removed). Some devices do not include a resistor at
all. Other devices have a mechanism to select the resistor (typically
by connecting a "pin jumper"). Be sure to check the schematics of all
devices on the CAN bus to verify that there are two and only two 120
Ohm resistors on the bus.

To test that the resistors are correct, one can remove power to the
printer and use a multi-meter to check the resistance between the CANH
and CANL wires - it should report ~60 ohms on a correctly wired CAN
bus.

## Finding the canbus_uuid for new micro-controllers

Each micro-controller on the CAN bus is assigned a unique id based on
the factory chip identifier encoded into each micro-controller. To
find each micro-controller device id, make sure the hardware is
powered and wired correctly, and then run:
```
~/klippy-env/bin/python ~/klipper/scripts/canbus_query.py can0
```

If uninitialized CAN devices are detected the above command will
report lines like the following:
```
Found canbus_uuid=11aa22bb33cc, Application: Klipper
```

Each device will have a unique identifier. In the above example,
`11aa22bb33cc` is the micro-controller's "canbus_uuid".

Note that the `canbus_query.py` tool will only report uninitialized
devices - if Klipper (or a similar tool) configures the device then it
will no longer appear in the list.

## Configuring Klipper

Update the Klipper [mcu configuration](Config_Reference.md#mcu) to use
the CAN bus to communicate with the device - for example:
```
[mcu my_can_mcu]
canbus_uuid: 11aa22bb33cc
```

## USB to CAN bus bridge mode

Some micro-controllers support selecting "USB to CAN bus bridge" mode
during "make menuconfig". This mode may allow one to use a
micro-controller as both a "USB to CAN bus adapter" and as a Klipper
node.

When Klipper uses this mode the micro-controller appears as a "USB CAN
bus adapter" under Linux. The "Klipper bridge mcu" itself will appear
as if was on this CAN bus - it can be identified via `canbus_query.py`
and configured like other CAN bus Klipper nodes. It will appear
alongside other devices that are actually on the CAN bus.

Some important notes when using this mode:

* The "bridge mcu" is not actually on the CAN bus. Messages to and
  from it do not consume bandwidth on the CAN bus. The mcu can not be
  seen by other adapters that may be on the CAN bus.

* It is necessary to configure the `can0` (or similar) interface in
  Linux in order to communicate with the bus. However, Linux CAN bus
  speed and CAN bus bit-timing options are ignored by Klipper.
  Currently, the CAN bus frequency is specified during "make
  menuconfig" and the bus speed specified in Linux is ignored.

* Whenever the "bridge mcu" is reset, Linux will disable the
  corresponding `can0` interface. To ensure proper handling of
  FIRMWARE_RESTART and RESTART commands, it is recommended to replace
  `auto` with `allow-hotplug` in the `/etc/network/interfaces.d/can0`
  file. For example:
```
allow-hotplug can0
iface can0 can static
    bitrate 500000
    up ifconfig $IFACE txqueuelen 128
```
