# raspberry-pi-yeelight-flic2
Control a Yeelight RGB wifi lamp with a flic2 BLE button via a Raspberry Pi

## Requirements
* Raspberry Pi 3, 4 or zero w (older versions will also work if combined with wifi and BLE dongles)
* [Yeelight RGB led bulb](https://www.yeelight.com/en_US/product/lemon2-color)
* [Flic 2 smart button](https://flic.io/flic2)

This guide assumes some basic understanding of UNIX.

## Getting started
First setup your Pi by following the [official guide](https://projects.raspberrypi.org/en/pathways/getting-started-with-raspberry-pi). During this guide we will be communicating with your Pi via SSH, so next [setup](https://www.raspberrypi.org/documentation/remote-access/ssh/) SSH on your Pi. You should now be able to connect with the command-line of your Pi from your laptop (in my case a Macbook).

After you're succesfully connected through SSH as user 'pi', upgrade the Pi's firmware (this may take a while):
```
sudo apt update
sudo apt full-upgrade
```

Install git:
```
sudo apt-get install git
```
Change directory to /home/pi, and create three new folders (if they don't yet exist):
```
cd /home/pi
mkdir Downloads
mkdir lib
cd lib
mkdir flic
```
Change directory to Downloads, clone the flic sdk and copy only the usefull files:
```
cd ../Downloads
git clone https://github.com/50ButtonsEach/fliclib-linux-hci.git
cp fliclib-linux-hci/bin/armv6l/flicd /home/pi/lib/flic
cp -r fliclib-linux-hci/clientlib/python/. /home/pi/lib/flic/
```
Then, check if pip (a Python package manager) is already installed.
```
pip3 --version
```
If it cannot find the command, install pip:
```
sudo apt-get install python3-pip
```
And install a Yeelight Python library:
```
pip3 install yeelight
```
