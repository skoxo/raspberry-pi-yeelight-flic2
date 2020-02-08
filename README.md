# raspberry-pi-yeelight-flic2
Control a Yeelight RGB wifi lamp with a Flic 2 smart button (BLE) via a Raspberry Pi

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
Change directory to `/home/pi`, and create three new folders (if they don't yet exist):
```
cd /home/pi
mkdir Downloads
mkdir lib
cd lib
mkdir flic
```
Change directory to Downloads, clone the flic sdk and copy only the usefull files to a permanent location:
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
sudo pip3 install yeelight
```

## Connecting with your Yeelight and Flic 2
### Yeelight
The Yeelight and Raspberry Pi should of course be connected to the same wifi network. To be able to connect with you light bulb you should determine it's IP adress. First open a Python prompt and then use the Yeelight Python library to obtain information about your bulb. Write down the ip-address that is returned in the prompt and exit the prompt with `ctrl D`.
```
python3

from yeelight import discover_bulbs
discover_bulbs()
```

### Flic 2
First, start the flic-daemon (this will remain running in the command-line):
```
cd /home/pi/lib/flic/
./flicd -f flic.sqlite3
```
Then open a new SSH connection with your Pi. Run the following code and press and hold your Flic button untill information about the button is printed to the command-line. You're Flic button is then succesfully coupled with the Raspberry Pi.
```
cd /home/pi/lib/flic/
python3 new_scan_wizard.py
```
In the Flic SDK, that you downloaded and copied to your `/home/pi/lib/flic` folder, there is a python script `test_client.py`. We will copy the contents of this file to a new file called `my_client.py`:
```
cp test_client.py my_client.py
```
Modify the contents of `my_client.py` with the nano command-line text editor:
```
nano my_client.py
```
In the nano editor, in between `import fliclib` and `client = fliclib.FlicClient("localhost")` insert the lines below. Where you replace `x.x.x.x` with the IP-address of your Yeelight bulb (the one that you wrote down). The three methods are called when a button is either single, double or long-pressed. You can of course change the light [settings](https://yeelight.readthedocs.io/en/stable/) in these methods to your liking.
```
from yeelight import Bulb
bulb = Bulb("x.x.x.x")

def singlePress():
	try:
		bulb.set_rgb(255,131,16)
		bulb.set_brightness(50)
	except:
		None

def doublePress():
	try:
		bulb.turn_on()
		bulb.set_color_temp(6464)
		bulb.set_brightness(100)
	except:
		None

def longPress():
	try:
		bulb.turn_on()
		bulb.set_rgb(255,0,0)
		bulb.set_brightness(50)
	except:
		None
```
And replace the following text:
```
cc = fliclib.ButtonConnectionChannel(bd_addr)
cc.on_button_up_or_down = \
		lambda channel, click_type, was_queued, time_diff: \
			print(channel.bd_addr + " " + str(click_type))
```
by:
```
cc = fliclib.ButtonConnectionChannel(bd_addr,0,10)
cc.on_button_single_or_double_click_or_hold = \
		lambda channel, click_type, was_queued, time_diff: ([bulb.toggle(), singlePress()] if click_type == fliclib.ClickType.ButtonSingleClick else (doublePress() if click_type == fliclib.ClickType.ButtonDoubleClick else longPress()))
```
Save the file by hitting `ctrl X`, typing `y` and hitting `enter`.

Now, we want to make sure that the flic-daemon and client script are started when the Raspberry Pi is powered on.
