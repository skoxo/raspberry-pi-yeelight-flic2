# Guide for controlling your Yeelight with a Flic 2 button via a Raspberry Pi
Use a wifi and bluetooth equiped Raspberry Pi to control your Yeelight RGB wifi bulb with a Flic 2 bluetooth button.

## Requirements
* Raspberry Pi 3, 4 or zero w (older versions will also work if combined with wifi and BLE dongles)
* [Yeelight RGB led bulb](https://www.yeelight.com/en_US/product/lemon2-color)
* [Flic 2 smart button](https://flic.io/flic2)

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
Check if pip (a Python package manager) is already installed.
```
pip3 --version
```
If it cannot find the command, install pip:
```
sudo apt-get install python3-pip
```
Install a Yeelight Python library:
```
sudo pip3 install yeelight
```

## Connecting with your Yeelight and Flic 2
### Yeelight
The Yeelight and Raspberry Pi should of course be connected to the same wifi network. To be able to connect with you light bulb you should determine it's IP adress. First open a Python command-line by typing:
```
python3
```
And use the Yeelight Python library, with the code below, to obtain information about your bulb. Write down the ip-address that is returned in the command-line and exit with `ctrl D`.
```
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
In the nano editor, in between `import fliclib` and `client = fliclib.FlicClient("localhost")` insert the lines below. Mind the indentation! Replace `x.x.x.x` with the IP-address of your Yeelight bulb (the one that you wrote down). The three methods are called when a button is either single, double or long-pressed. You can of course change the light [settings](https://yeelight.readthedocs.io/en/stable/) in these methods to your liking.
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
Replace the following text:
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


To test whether the Flic - Raspberry Pi - Yeelight connection is working, make sure that the flic-daemon script is still running in your first SSH command-line. Then start the client script with the following command. Information about the button and connection should be printed to the command-line. If you then press your button, your light bulb should react! 
```
python3 my_client.py
```
Stop the python script by hitting `ctrl C`. And stop the daemon by hitting `ctrl C` in the first command-line window.

## Wrapping up
Now, we want to make sure that the flic-daemon and client script are started when the Raspberry Pi is powered on. First, we create a new bash script that starts the daemon:
```
cd /home/pi/lib/flic
nano start_daemon.sh
```
Fill the script with the following code and save and close the file like you did before.
```
#!/bin/sh
/home/pi/lib/flic/flicd -f /home/pi/lib/flic/flic.sqlite3 -w
```
Second, we create a bash script that starts the client:
```
nano start_client.sh
```
Fill it with:
```
#!/bin/sh
/usr/bin/python3 /home/pi/lib/flic/my_client.py &
```
Set execution permission for these scripts:
```
sudo chmod +x start_daemon.sh start_client.sh
```
Next, we can check if the scripts work. First enter `./start_daemon.sh`. Then, open a new SSH command-line and enter `./start_client.sh`. Pressing the Flic button should turn on your Yeelight!

Now, we want to create a system service that is automatically started when your Raspberry boots and is restarted when it crashes. To do this we are going to create a system service that runs on boot and starts the Flic daemon (flicd):
```
sudo nano /etc/systemd/system/flicd.service
```
Fill the empty `flicd.service` with the following code:
```
[Unit]
Description=flicd Service
Before=rc-local.service

[Service]
TimeoutStartSec=0
ExecStart=/home/pi/lib/flic/start_daemon.sh
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```
Disable the standard bluetooth service and enable the Flic daemon to run on boot and start the service:
```
sudo systemctl stop bluetooth.service
sudo systemctl disable bluetooth.service
sudo systemctl enable flicd.service
sudo systemctl start flicd.service
```
The Flic daemon is all set, next is the Flic client. Open `rc.local` for editing:
```
sudo nano /etc/rc.local
```
Add the following code __above__ `exit 0`:
```
sleep 5
sudo -H -u pi /home/pi/lib/flic/start_client.sh
```
Reload the system services daemon:
```
sudo systemctl daemon-reload
```
Everything should be ready now!! 
To check, reboot your Raspberry Pi:
```
sudo reboot now
```
