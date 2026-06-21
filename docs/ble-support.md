# Bluetooth Low Energy (BLE) Support

This RISC-V Linux platform supports Bluetooth Low Energy (BLE) through the Linux BlueZ stack.
BLE connectivity can be achieved using a USB BLE dongle or a UART-connected BLE module.

## Prerequisites

The Linux kernel shipped with this project includes Bluetooth and BLE support compiled as loadable modules:
- `CONFIG_BT=m` — Bluetooth subsystem
- `CONFIG_BT_LE=y` — Bluetooth Low Energy
- `CONFIG_BT_HCIBTUSB=m` — HCI USB driver (for USB BLE dongles)
- `CONFIG_BT_HCIUART=m` — HCI UART driver (for UART BLE modules)
- `CONFIG_RFKILL=m` — RF kill switch management

## Option 1: USB BLE Dongle

Connect a USB BLE adapter (e.g., based on CSR8510, RTL8761, or similar chipsets) to the board's USB port.

### Install BlueZ

After booting Linux on the FPGA, install the BlueZ Bluetooth stack:

```
sudo apt update
sudo apt install bluetooth bluez bluez-tools
```

### Load kernel modules

```
sudo modprobe bluetooth
sudo modprobe btusb
```

### Start Bluetooth service

```
sudo systemctl enable bluetooth
sudo systemctl start bluetooth
```

### Verify Bluetooth adapter

```
hciconfig -a
```

You should see a `hci0` adapter listed.

## Option 2: UART BLE Module

Connect a UART BLE module (e.g., HC-08, HM-10, or similar) to the board's UART interface (`/dev/ttyUSBx` or `/dev/ttyS0`).

### Load HCI UART driver

```
sudo modprobe bluetooth
sudo modprobe hci_uart
```

### Attach the UART device

```
sudo hciattach /dev/ttyUSB0 any 115200 noflow
```

Adjust the baud rate to match your module's default (common values: 9600, 38400, 57600, 115200).

## BLE Operations with bluetoothctl

Use the interactive `bluetoothctl` utility to manage BLE connections:

```
sudo bluetoothctl
```

Inside `bluetoothctl`:

```
[bluetooth]# power on
[bluetooth]# agent on
[bluetooth]# default-agent
[bluetooth]# scan on
```

Wait for BLE devices to appear, then connect:

```
[bluetooth]# scan off
[bluetooth]# pair <MAC_ADDRESS>
[bluetooth]# connect <MAC_ADDRESS>
[bluetooth]# trust <MAC_ADDRESS>
```

## BLE Scanning with hcitool

Scan for nearby BLE devices:

```
sudo hcitool lescan
```

Example output:
```
LE Scan ...
AA:BB:CC:DD:EE:FF  MyBLEDevice
```

## GATT Attribute Browsing

Install `bluez-tools` and use `gatttool` to interact with GATT services:

```
sudo apt install bluez-tools
gatttool -b <MAC_ADDRESS> -I
```

Or use `bluetoothctl` GATT commands directly:

```
[bluetooth]# menu gatt
[bluetooth]# connect <MAC_ADDRESS>
[bluetooth]# list-attributes
[bluetooth]# select-attribute <UUID>
[bluetooth]# read
```

## Adding Bluetooth to the Device Tree

If you are connecting a UART BLE module to the FPGA UART controller, add the following to the `io-bus` section of `board/<board-name>/bootrom.dts`:

```
        bluetooth: bluetooth@60040000 {
            compatible = "uart-hci";
            reg = <0x60040000 0x10000>;
            interrupt-parent = <&L2>;
            interrupts = <5>;
            current-speed = <115200>;
        };
```

Adjust the `reg` address, interrupt number, and `current-speed` to match your module and design. Consult your BLE module's datasheet for the correct compatible string (e.g., `brcm,bcm43438-bt` for Broadcom BCM43438, `qcom,wcn3990-bt` for Qualcomm WCN3990). Then rebuild the FPGA bitstream:

```
make CONFIG=rocket64b2 BOARD=nexys-video bitstream
```

## Enabling Bluetooth at Boot

To automatically load Bluetooth modules on boot, add them to `/etc/modules`:

```
echo "bluetooth" | sudo tee -a /etc/modules
echo "btusb" | sudo tee -a /etc/modules
```

## Troubleshooting

- **No Bluetooth adapter found**: Make sure the `btusb` or `hci_uart` module is loaded and the hardware is connected.
- **rfkill blocked**: Run `sudo rfkill unblock bluetooth` to unblock the adapter.
- **Permission denied**: Add your user to the `bluetooth` group: `sudo usermod -aG bluetooth $USER`.
- **Module not found**: Rebuild the Linux kernel with `make linux` after updating `patches/linux.config`.

## References

- [BlueZ official documentation](http://www.bluez.org/)
- [Linux Bluetooth subsystem](https://www.kernel.org/doc/html/latest/networking/bluetooth.html)
- [Debian Bluetooth setup](https://wiki.debian.org/BluetoothUser)
