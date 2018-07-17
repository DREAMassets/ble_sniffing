# D.R.E.A.M.

## Raspberry Pi Setup

Here are some basic steps for setting up a Raspberry Pi from the CanaKit box.

## Initialize the RPi:
1. put the SD card in
1. hook up a monitor, keyboard and mouse
1. Follow the steps to install Raspberrian
  _This is going to take a while, so grab a :coffee: or whatnot_

## Get the RPi ready for remote usage
1. run `raspi-config` to hook the Pi up to the wireless network
1. run `raspi-config` to change the hostname to something unique that is not `raspberryPi`
1. update the password for the Pi account

At this point you should be able to disconnect the Pi from everything except power and it should be accessible.
Make sure you are on the **same wireless network** as it will only be availble within that subnet.

1. SSH into the RPi from your laptop's terminal so you no longer need to use a separate monitor:
`ssh pi@raspberryPi.local` where `raspberryPi` is replaced by your unique hostname.
*if this doesn't work, verify that both your laptop and RPi are on the same network.*
1. Disconnect your RPi from the monitor, keyboard and mouse and put it in whatever location is most convenient for you.

## Get the RPi ready for Bluetooth Low Energy (BLE)
We need a few extra packages for the Pi to be a BLE sniffer.  First get dependencies for [Bluez](http://www.bluez.org/):

```
sudo apt-get install libdbus-1-dev libdbus-glib-1-dev libglib2.0-dev libical-dev libreadline-dev libudev-dev libusb-dev make glib2.0 libdbus-1-dev libudev-dev  
```
Then get Bluez itself: 
```
sudo apt-get install bluez bluez-hcidump
```

Bluez provides the commands `hcitool` and `hcidump` which are the main tools we (probably) will be using to interact with Bluetooth.

To start sniffing, open 2 consoles on the Pi.  In the first console, start up a scanner

```
sudo hciconfig hc0 up
sudo hcitool lescan 
```
those commands return a list of hexadecimal BLE addresses and device names, which by default are `unknown`:
```
D5:BB:5C:B3:0C:1C (unknown)
7C:64:56:36:1F:D9 (unknown)
7C:64:56:36:1F:D9 [TV] Samsung 6 Series (55)
4C:0C:F7:63:33:CB (unknown)
8C:85:90:63:A9:29 (unknown)
18:B4:30:E3:A5:89 Nest Cam
DC:56:E7:3C:F6:5F (unknown) 
...
```

While the first console continues to output BLE addresses, go to the second console and start up the data dumper:
```
sudo hcidump --raw
```
this command returns a spew of data coming out of the `hcidump` terminal:
```
> 04 3E 23 02 01 00 01 90 2D A5 6C 18 55 17 02 01 06 13 FF 4C 00 0C 0E 00 5F 36 E2 C9 5C 2D 08 B1 72 A7 09 E3 AD CB
> 04 3E 0C 02 01 04 01 90 2D A5 6C 18 55 00 CC
> 04 3E 2B 02 01 00 01 89 A5 E3 30 B4 18 1F 11 07 6C 32 44 5D C8 91 B3 A2 9C 4D 99 9C EF F8 D3 D2 0C FF 18 B4 30 E3 A5 89
> 04 3E 19 02 01 04 01 89 A5 E3 30 B4 18 0D 02 01 06 09 08 4E 65 73 74 20 43 61 6D A5
> 04 3E 17 02 01 00 00 29 A9 63 90 85 8C 0B 02 01 06 07 FF 4C 00 10 02 0B 00 9F
> 04 3E 1A 02 01 00 00 5F F6 3C E7 56 DC 0E 02 01 1A 0A FF 4C 00 10 05 01 10 20 F6 71 9D
> 04 3E 23 02 01 00 01 01 3B F3 FD E1 4B 17 02 01 06 13 FF 4C 00 0C 0E 00 A0 1D 96 2C F6 98 2C 3B A2 97 3A EB F9 AA
> 04 3E 0C 02 01 04 01 01 3B F3 FD E1 4B 00 AA
< 01 0C 20 02 00 01
> 04 0E 04 01 0C 20 00
...
```
Congrats -- you're sniffing data from the BLE devices near your RPi. 

## Pulling the github repo to the Raspberry Pi

First you need to setup some ssh keys so that Github will be happy with your connection.  On your Pi

```
ssh-keygen
```

It will ask for a filename/directory to save this key.  Leave it as default.
It will also ask for a passphrase.  I've been using the same as the `pi` user's password.

Once that is setup you can clone the repository like so

```
git clone https://github.com/DREAMassets/ble_sniffer.git
```

At this point you'll have a directory called `ble_sniffer` with the contents of the github repo.  Follow
the instructions below on using the scanner and parser scripts.

## Scanner

A small bash script can be used to sniff/scan BLE packets that match our desired packets from the Fujitsu tags.

From the root project directory, you can run

```bash
sniffer/tag_scanner.sh
```

This will report a list of packets that it has collected.   Each packet is written out in the following format

```
[1531866884] 043E2102010301DFCE793166C41502010411FF5900010003000300640486FF6E0025F8C2
```
which include the timestamp (in seconds since the Epoch) in brackets followed by the packet data.

To get all this in a file you can read later

```bash
sniffer/tag_scanner.sh > sniffed_packets.txt
```

Logic to unpack that is in the parser.

## Parser

Using a short ruby script, you can parse the data from the sniffer.  Assuming you've saved the sniffer data in a file called `sniffed_packets.txt`, you'd run

```
cat sniffed_packets.txt | parser/packet_parser.rb
```

and you should get an output like
```
71BF99DC8CF7,77.27 degF,0.071,0.072,-1.019,219
1C0CB35CBBD5,77.13 degF,0.055,0.000,1.050,184
DFCE793166C4,76.98 degF,0.054,-0.009,1.018,210
71BF99DC8CF7,76.92 degF,0.054,0.081,-1.017,212
1C0CB35CBBD5,77.73 degF,0.056,-0.003,1.056,188
F2461FBDA1D4,77.28 degF,0.029,0.029,-1.026,194
```

which is a CSV format with columns "Device ID (UUID), Temperature (degF), x acceleration, y acceleration, z_acceleration, rssi".
Acceleration is measured in g's.  Rssi units are currently unknown but run from 0 to 255.
