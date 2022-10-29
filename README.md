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
    sudo apt-get install libudev-dev libinput-dev libts-dev libxcb-xinerama0-dev libxcb-xinerama0 gdbserver pkg-config mesa-utils libgles2-mesa-dev libdrm.dev libgbm.dev

### Create a directory for the Qt install

    sudo mkdir /usr/local/qt
    sudo chown pi:pi /usr/local/qt


## Configure PC

### Update the PC and install the required development packages

    sudo apt-get update
    sudo apt-get upgrade
    sudo apt-get install gcc git bison python gperf pkg-config gdb-multiarch
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
The files we copied in the previous step still have symbolic links pointing to the file system on the Raspberry Pi. We need to alter this so that they become relative links from the new sysroot directory on the host machine. We can do this with a downloadable python script. To download it, enter the following:

    wget https://raw.githubusercontent.com/riscv/riscv-poky/master/scripts/sysroot-relativelinks.py

Once it is downloaded, you just need to make it executable and run it, using the following commands:

    sudo chmod +x sysroot-relativelinks.py
    ./sysroot-relativelinks.py sysroot

