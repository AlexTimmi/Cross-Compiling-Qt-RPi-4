# Cross-Compiling Qt for Raspberry Pi 4
This guide documents the steps to corss-compile Qt 5.15.2 for the Raspberry Pi 4B.


## Configure the Raspberry Pi
For our build we need to have SSH and GL (FAKE KMS) enabled. These can both be done through the `raspi-config` utility.  

On the Raspberry Pi terminal:

	sudo raspi-config
	
A menu should pop up on your terminal
<img src="img/raspi-config.png" />

### Enable SSH

	Interfacing Options -> SSH -> Yes
	
### Enable GL (FAKE KMS)

	Advanced Options -> GL Driver -> GL (Full KMS) OpenGL desktop driver with full KMS

That should enable KMS. If you are using a minimal build, you may be prompted to download some updates before this option becomes available. If asked, do so.

### Enable Development Sources

	sudo nano /etc/apt/sources.list
	
Uncomment the following line by removing the `#` character:

	deb-src http://raspbian.raspberrypi.org/raspbian/ bullseye main contrib non-free rpi

<img src="img/deb-src.png" />
	
### Update the system

	sudo apt-get update
	sudo apt-get upgrade
	sudo reboot

### Install the required development packages

    sudo apt-get build-dep qt5-qmake
    sudo apt-get build-dep libqt5gui5
    sudo apt-get build-dep libqt5webengine-data
    sudo apt-get build-dep libqt5webkit5
    sudo apt-get install libudev-dev libinput-dev libts-dev libxcb-xinerama0-dev libxcb-xinerama0 dbserver pkg-config mesa-utils libgles2-mesa-dev libdrm.dev libgbm.dev

### Create a directory for the Qt install

    sudo mkdir /usr/local/qt
    sudo chown pi:pi /usr/local/qt

