# Cross-Compiling Qt for Raspberry Pi 4
This guide documents the steps to corss-compile Qt 5.15.2 for the Raspberry Pi 4.

## Acknowledgments

Used the following guides for reference:

* https://mechatronicsblog.com/cross-compile-and-deploy-qt-5-12-for-raspberry-pi/
* https://www.tal.org/tutorials/building-qt-515-lts-raspberry-pi-raspberry-pi-os
* https://stackoverflow.com/questions/68081481/how-can-i-build-qt-5-13-2-with-gcc-11-1-on-windows
* https://github.com/UvinduW/Cross-Compiling-Qt-for-Raspberry-Pi-4

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
    sudo apt-get install libudev-dev libinput-dev libts-dev libxcb-xinerama0-dev libxcb-xinerama0 gdbserver pkg-config mesa-utils libgles2-mesa-dev libdrm.dev libgbm.dev

### Create a directory for the Qt install

    sudo mkdir /usr/local/qt
    sudo chown pi:pi /usr/local/qt


## Configure PC

### Update the PC and install the required development packages

    sudo apt-get update
    sudo apt-get upgrade
    sudo apt-get install gcc git bison python3 gperf pkg-config gdb-multiarch
    sudo apt install build-essential

### Set up SSH keys to speed up connecting with the Raspberry Pi

Normally, everytime you connect from your PC to the RPi, you will need to provide the login credentials. We can use SSH keys to avoid this and speed up the process.

Check if there are any existing keys:

    ls ~/.ssh

If any of the files are found as a result of the command execution:
* `id_rsa.pub`
* `id_dsa.pub`

This means that the keys are already present in the system. In this case, the generation step can be skipped

#### SSH key generation
    
    ssh-keygen

#### Copy key to Raspberry Pi
[change *rpi.lan* with the IP address or hostname for your RPi]

    ssh-copy-id pi@rpi.lan

### Create a folder on the host and download the toolchain for cross-compilation

    mkdir ~/raspberrypi
    cd ~/raspberrypi
    wget https://releases.linaro.org/components/toolchain/binaries/latest-7/arm-linux-gnueabihf/gcc-linaro-7.5.0-2019.12-x86_64_arm-linux-gnueabihf.tar.xz
    tar xf gcc-linaro-7.5.0-2019.12-x86_64_arm-linux-gnueabihf.tar.xz

### Creating a sysroot for cross-compiling under Raspberry Pi

    mkdir sysroot sysroot/usr sysroot/opt

### Sync sysroot on PC with Raspberry Pi
[change *rpi.lan* with the IP address or hostname for your RPi]

    rsync -avz pi@rpi.lan:/lib sysroot
    rsync -avz pi@rpi.lan:/usr/include sysroot/usr
    rsync -avz pi@rpi.lan:/usr/lib sysroot/usr
    rsync -avz pi@rpi.lan:/opt sysroot/opt

### Fix symbolic links
The files we copied in the previous step still have symbolic links pointing to the file system on the Raspberry Pi. We need to alter this so that they become relative links from the new sysroot directory on the host machine. We can do this with a downloadable python script.

To download it:

    wget https://raw.githubusercontent.com/AlexTimmi/Cross-Compiling-Qt-RPi-4/main/scripts/sysroot-relativelinks.py

Make it executable and run:

    sudo chmod +x sysroot-relativelinks.py
    ./sysroot-relativelinks.py sysroot

### Download the Qt sources

Use LTS version 5.15.2

    git clone git://code.qt.io/qt/qtbase.git -b 5.15.2
    cd qtbase

### Fix includ header

Since with GCC 11 some header dependencies have changed and Qt 5.15.2 does not always include the right headers.

Add the line `#include <limits>` within the `#ifdef __cplusplus` block to the file *src/corelib/global/qglobal.h*

Edit:

    nano src/corelib/global/qglobal.h

<img src="img/qt-fix-include.png" />

### Perform configuration
For Raspberry Pi 4, the -device argument will be linux-rasp-pi4-v3d-g++

    ./configure -release -opengl es2 -eglfs -device linux-rasp-pi4-v3d-g++ -device-option CROSS_COMPILE=~/raspberrypi/gcc-linaro-7.5.0-2019.12-x86_64_arm-linux-gnueabihf/bin/arm-linux-gnueabihf- -sysroot ~/raspberrypi/sysroot -prefix /usr/local/qt -extprefix ~/raspberrypi/qt -hostprefix ~/raspberrypi/host-qt -opensource -confirm-license -skip qtscript -skip qtwayland -skip qtwebengine -nomake tests -make libs -pkg-config -no-use-gold-linker -v -recheck

Upon successful completion of this step, the following items should be activated in the console:
    
<img src="img/config-res.png" />

In case of a failed configuration or build, do not forget to perform a full cleanup before trying again:

    git clean -dfx


### Make & install

    make -j8
    make install

### Deploying built Qt on Raspberry Pi
[change *rpi.lan* with the IP address or hostname for your RPi]

    cd ~/raspberrypi
    rsync -avz qt pi@rpi.lan:/usr/local


## Qt Creator for Raspberry Pi

### Install

    sudo apt install qtcreator

### Adding a device to Qt Creator

Go to the *Tools - Options* menu, open the *Devices* section and add the *Generic Linux Device*:

<img src="img/qt-device.png" />

Here we set the device name, ip address or hostname and username. In addition, specify a private SSH key for accessing the Raspberry Pi via SSH.

### Add compilers to Qt Creator

Go to the *Kits* section on the *Compilers* tab and add compilers for C and C ++

Compiler paths:

C:

    ~/raspberrypi/gcc-linaro-7.5.0-2019.12-x86_64_arm-linux-gnueabihf/bin/arm-linux-gnueabihf-gcc

C++:

    ~/raspberrypi/gcc-linaro-7.5.0-2019.12-x86_64_arm-linux-gnueabihf/bin/arm-linux-gnueabihf-g++

<img src="img/qt-compiler-c.png" />
<img src="img/qt-compiler-c++.png" />

### Add a debugger to Qt Creator

Pre-install:

    sudo apt-get install gdb-multiarch

In the same section, on the *Debuggers* tab, add a debugger to work with Raspberry

Path:

    /usr/bin/gdb-multiarch

<img src="img/qt-debugger.png" />

### Add version to Qt Creator

Qt Versions tab

qmake path:

    ~/raspberrypi/host-qt/bin/qmake

<img src="img/qt-version.png" />

### Add kit to Qt Creator

Kits tab

Sysroot:

    ~/raspberrypi/sysroot

<img src="img/qt-kit.png" />

When creating a new project, the created kit, which name *Raspberry Pi*, will be available.