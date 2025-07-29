# Cross compile QT 5.15.2 for Raspberry Pi 4B - 2024

## Set-up 
**Hardware**
Host: Intel i10700H - GTX 1650
Target: Raspberry Pi 4B

**Software**
Host: Ubuntu 22.04 LTS run in VMWare Virtual Machine within Windows 10
Target: **Rasberry Pi OS(32bit, legacy, bulleye)**
Cross Compiler: gcc-linaro-7.4.1-2019.02-x86_64_arm-linux-gnueabihf 

**Other notes**
VMWare Player run on 4 cores CPU + 4Gb RAM
## Achnowledgements
This guide is heavily based on the one published for Qt 14.1 by Walter Prechtl from INTERELECTRONIX found here:
https://www.interelectronix.com/de/qt-auf-dem-raspberry-pi-4.html (with the help of Google Translate)

I also used the following guides for reference:  
- https://wapel.de/?p=641  
- https://mechatronicsblog.com/cross-compile-and-deploy-qt-5-12-for-raspberry-pi/  
- https://www.tal.org/tutorials/building-qt-512-raspberry-pi

And many thanks to [Oliver Wilkins](https://github.com/oliverwilkins) for his guidance and support!

---

## Rasberry Pi Setup
### Step 1: Install Raspberry Pi OS
**Note: Must be 32 bit, bulleye**

### Step 2: Configure WiFi setup

### Step 3: Enable SSH

### Step 4: Change GPU memery and disable splash (optinal)
Add `gpu_mem=256` and `disable_splash=1` in the end of /boot/config.txt.
Use `sudo nano /boot/config.txt` 

### Step 5: Boot in CLI mode
Use `sudo raspi-config` -> System Options -> Boot / Auto login -> Console Autologin -> reboot


### Step 6: Enable Development Sources
To do this, enter the following command into terminal
`sudo nano /etc/apt/sources.list`

In the nano text editor, uncomment the following line by removing the `#` character (the line should exist already, if not then add it):

	deb-src http://raspbian.raspberrypi.org/raspbian/ bulleye main contrib non-free rpi
	
Now press `Ctrl+X` to quit. You will be asked if you want to save the changes. Press `y` for yes, and then press `Enter` to keep the same filename.

### Step 7: Update the system
Run the following command into a terminal to update and reboot.
```
sudo apt-get update
sudo apt-get upgrade
sudo apt-get dist-upgrade
sudo reboot
```

### Step 8: Enable rsync with root rights
+ Find the rsync path with the following command
`which rsync`
On my Rpi, it was here: `/usr/bin/rsync`
+ Edit the sudoer file
`sudo visudo`
+ Add the following line to the end of file with the structure: `<username> ALL=NOPASSWD:<path to rsync>`
+ Mine was: `pi ALL=NOPASSWD:/usr/bin/rsync`

### Step 9: Install the required development packages
Use the following command to install the required packages
```
sudo apt-get build-dep qt5-qmake
sudo apt-get build-dep libqt5gui5
sudo apt-get build-dep libqt5webengine-data
sudo apt-get build-dep libqt5webkit5
sudo apt-get install libudev-dev libinput-dev libts-dev libxcb-xinerama0-dev libxcb-xinerama0 gdbserver
sudo apt-get install libegl1-mesa libegl1-mesa-dev mesa-common-dev libgles2-mesa libgles2-mesa-dev libgbm-dev libdrm-dev
sudo apt install build-essential cmake unzip pkg-config gfortran
sudo apt install libxcb-randr0-dev libxcb-xtest0-dev libxcb-shape0-dev libxcb-xkb-dev
```

### Step 10: Install SSymlinker
```
wget https://raw.githubusercontent.com/abhiTronix/raspberry-pi-cross-compilers/master/utils/SSymlinker
sudo chmod +x SSymlinker
./SSymlinker -s /usr/include/arm-linux-gnueabihf/asm -d /usr/include
./SSymlinker -s /usr/include/arm-linux-gnueabihf/gnu -d /usr/include
./SSymlinker -s /usr/include/arm-linux-gnueabihf/bits -d /usr/include
./SSymlinker -s /usr/include/arm-linux-gnueabihf/sys -d /usr/include
./SSymlinker -s /usr/include/arm-linux-gnueabihf/openssl -d /usr/include
./SSymlinker -s /usr/lib/arm-linux-gnueabihf/crtn.o -d /usr/lib/crtn.o
./SSymlinker -s /usr/lib/arm-linux-gnueabihf/crt1.o -d /usr/lib/crt1.o
./SSymlinker -s /usr/lib/arm-linux-gnueabihf/crti.o -d /usr/lib/crti.o
```

### Step 11: Create a directory for the Qt install
This is where the built Qt sources will be deployed to on the Rasberry Pi. Run the following to create the directory:
```
sudo mkdir /usr/local/qt5.15
sudo chown -R pi:pi /usr/local/qt5.15
```

### Step 12: Get the RPI IP address (optional)
Run the following command `hostname -I` or `ifconfig`

---

## Host setup
### Step 1: Update host
Use
```
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install gcc g++ git bison python gperf pkg-config gdb-multiarch wget
sudo apt install build-essential
sudo apt install cmake unzip gfortran
sudo apt-get -y install gcc g++ gperf flex texinfo gawk bison openssl pigz libncurses-dev autoconf automake tar figlet
```
**Notes** 
By default: Ubuntu 22.04 LTS will install gcc 11 and g++ 11
Check the gcc and g++ version:
```
gcc --version
g++ --version
```
If you already have gcc 9 and g++ 9, skip these steps
If you don't, you must install gcc 9 and g++ 9
#### Step 1.1: Install
Use the following command
```
sudo apt-get install gcc-9
sudo apt-get install g++-9
```

#### Step 1.2: Use `update-alternatives` to control many gcc/g++ version
```
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-11 20
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-9 10

sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-11 20
sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-9 10
```
#### Step 1.3: Change the working gcc/g++ to gcc/g++ 9
Use these following commands 
```
sudo update-alternatives --config gcc
sudo update-alternatives --config g++
```

And then choose gcc/g++ 9

Check again the version using:
```
gcc --verison
g++ --version
```

If it is version 9, continue. If not, something went wrong and you need to check it again.


### Step 2: Connect to the RPI via SSH
Using ssh to connect to the RPI
`ssh <username>@<ip_address>`

### Step 3: Set up the directory structure
Setup directory for cross compile
>tools: cross compiler directory
>build: qt build directory
>sysroot: RPI libs directory

```
sudo mkdir ~/rpi-qt
sudo mkdir ~/rpi-qt/build
sudo mkdir ~/rpi-qt/tools
sudo mkdir ~/rpi-qt/sysroot
sudo mkdir ~/rpi-qt/sysroot/usr
sudo mkdir ~/rpi-qt/sysroot/opt
sudo chown -R 1000:1000 ~/rpi-qt
cd ~/rpi-qt
```

### Step 5: Download Qt source
Run the following line to download the source files:
```
sudo wget http://download.qt.io/archive/qt/5.15/5.15.2/single/qt-everywhere-src-5.15.2.tar.xz
```

Extract the downloaded tar file with the following command:
```
sudo tar xfv qt-everywhere-src-5.15.2.tar.xz
```

### Step 6: Configure Qt source
We need to slightly modify the a mkspec file within the source files to allow us to use our cross compiler. We will copy an existing directory within the source files, and modify the name of the directory and the 
contents of the qmake.conf file within that directory to follow the name of our compiler.  

To do this, run the following two command:
```
cp -R qt-everywhere-src-5.15.0/qtbase/mkspecs/linux-arm-gnueabi-g++ qt-everywhere-src-5.15.2/qtbase/mkspecs/linux-arm-gnueabihf-g++
	
sed -i -e 's/arm-linux-gnueabi-/arm-linux-gnueabihf-/g' qt-everywhere-src-5.15.2/qtbase/mkspecs/linux-arm-gnueabihf-g++/qmake.conf

```
### Step 7: Download the cross-compiler
Goto rpi/tools directory
`cd ~/rpi/tools`

Download cross compiler 
`sudo wget https://releases.linaro.org/components/toolchain/binaries/7.4-2019.02/arm-linux-gnueabihf/gcc-linaro-7.4.1-2019.02-x86_64_arm-linux-gnueabihf.tar.xz`

Extract compiler:
`tar xfv gcc-linaro-7.4.1-2019.02-x86_64_arm-linux-gnueabihf.tar.xz`

### Step 8: Sync our sysroot
Copy all RPI lig into sysroot directory
```
rsync -avz --rsync-path="sudo rsync" --delete pi@192.168.1.7:/lib sysroot
rsync -avz --rsync-path="sudo rsync" --delete pi@192.168.1.7:/usr/include sysroot/usr
rsync -avz --rsync-path="sudo rsync" --delete pi@192.168.1.7:/usr/lib sysroot/usr
rsync -avz --rsync-path="sudo rsync" --delete pi@192.168.1.7:/opt/vc sysroot/opt
```
>Bulleye OS doesn't not have /opt/vc so the final command should get an error. Just skip it.

### Step 9: Fix symbolic links
The files we copied in the previous step still have symbolic links pointing to the file system on the Raspberry Pi. We need to alter this so that they become relative links from the new sysroot directory on the host machine.
We can do this with a downloadable python script. To download it, enter the following:
`wget https://raw.githubusercontent.com/riscv/riscv-poky/master/scripts/sysroot-relativelinks.py`

	
Once it is downloaded, you just need to make it executable and run it, using the following commands:
```
sudo chmod +x sysroot-relativelinks.py
./sysroot-relativelinks.py sysroot
```
**Notes**
`/sysroot-relativelinks.py sysroot` may get an error because python3 is loacte in a different directory. 
You can use `python3 sysroot-relativelinks.py sysroot`
Or install python-is-python3: `sudo apt install python-is-python3` and use `./sysroot-relativelinks.py sysroot` again.

### Step 10: Configure Qt build (Most important)
Let's move into the build directory that we created earlier inside the rpi folder:
`cd ~/rpi/build`

Build Qt
```
../qt-everywhere-src-5.15.2/configure -release -opengl es2  -eglfs -device linux-rasp-pi4-v3d-g++ -device-option CROSS_COMPILE=~/rpi/tools/gcc-linaro-7.4.1-2019.02-x86_64_arm-linux-gnueabihf/bin/arm-linux-gnueabihf- -sysroot ~/rpi/sysroot -prefix /usr/local/qt5.15 -extprefix ~/rpi/qt5.15 -opensource -confirm-license -skip qtscript -skip qtwayland -skip qtwebengine -nomake tests -make libs -pkg-config -no-use-gold-linker -v -recheck -L$HOME/rpi/sysroot/usr/lib/arm-linux-gnueabihf -I$HOME/rpi/sysroot/usr/include/arm-linux-gnueabihf -qpa eglfs -no-feature-eglfs_brcm
```

This may take a few minutes to complete. Once it is completed you should get a summary of what has been configured. Make sure the following options appear:
<pre><code>
QPA backends:
  DirectFB ............................... no
  <b>EGLFS .................................. yes	[SHOULD BE YES]</b>
  EGLFS details:
    EGLFS OpenWFD ........................ no
    EGLFS i.Mx6 .......................... no
    EGLFS i.Mx6 Wayland .................. no
    EGLFS RCAR ........................... no
    <b>EGLFS EGLDevice ...................... yes	[SHOULD BE YES]</b>
    EGLFS GBM ............................ yes
    EGLFS VSP2 ........................... no
    EGLFS Mali ........................... no
    <b>EGLFS Raspberry Pi ................... no	[SHOULD BE NO]</b>
    EGLFS X11 ............................ yes
  LinuxFB ................................ yes
  VNC .................................... yes
</code></pre>  

If the your configuration summary doesn't have the EGLFS features set to what's shown above, something has probably gone wrong. You can look at the config.log file in the build directory to try and diagnose what the issue might be.
If you see an error at the bottom of your config summary along the lines of "EGLFS was enabled but...." that could also be an indication something went wrong. Look through the config.log files to try and diagnose the error.  
You may get a warning about QDoc not being compiled. This can be safely ignored unless you specifically need this.

If you have any issues, before running configure again, delete the current contents with the following command (save a copy of config.log first if you need to):
`rm -rf *`

### Step 11: Build Qt
Use 
`make -j<core>`
I am using 4 core on virtual machine so I use 
`make -j4`

Once it is completed, we can install the built package using the following command:
`make install`

### Step 12: Deploy Qt to our Raspberry Pi
We can now deploy Qt to our RPi. We will again make use of the rsync command. First move back into the rpi folder using the following command:
`cd ~/rpi`
	
You should now see a new folder named "qt5.15" here. Copy this to the raspberry pi using the following command [replace 192.168.1.7 with your RPi's IP address]:
`rsync -avz --rsync-path="sudo rsync" qt5.15 pi@192.168.1.7:/usr/local`

### Step 13: Update linker on Raspberry Pi (On Raspberry Pi)
Enter the following command to update the device letting the linker to find the new Qt library files:
```
echo /usr/local/qt5.15/lib | sudo tee /etc/ld.so.conf.d/qt5.15.conf
sudo ldconfig
```
### Step 14: Build an example application
And that is done. Now we ganna build some program to see if it works.
To make a copy of the example project, run the following command:
`cp -r ~/rpi/qt-everywhere-src-5.15.0/qtbase/examples/opengl/qopenglwidget ~/rpi/`

Move into that new folder and build the source files with these commands:
```
cd ~/rpi/qopenglwidget
../qt5.15/bin/qmake
make
```

Copy the built binary file to the Rasberry Pi with this command [change IP address to your RPi's]:
`scp qopenglwidget pi@192.168.1.7:/home/pi`

Now **switch to the Raspberry Pi** and navigate to the home directory:
`cd ~`
	
Now run the compiled executable that we copied over from the host machine:
`./qopenglwidget`
	
The demo should start running on the display connected to the Raspberry Pi.

## Step 15: Windowed vs Full Screen Mode
In previous builds of Qt I've used on a Raspberry Pi, the apps I developed ran on the frame buffer directly, bypassing the X Window Manager. This also meant that the apps always ran full screen. 
I have found that this is not the case with this build of Qt.  

If I boot into the desktop mode and launch the app, it will run in windowed mode. This does have the benefit of being able to VNC into the Raspberry Pi and view what the result looks like.

However, if you want it to bypass the X Window Manager, the only solution I have found so far is to boot the RPi directly into CLI. This way the X Window Manager is not running, and the app runs full screen when launched.

[Pablojr has pointed out](https://github.com/UvinduW/Cross-Compiling-Qt-for-Raspberry-Pi-4/issues/2) that you can access the app through VNC even when you're in command line mode by adding the `-platform vnc` flag when launching it, like this:
`./qopenglwidget -platform vnc`

For this to work, you first need to disable the Raspberry Pi's built in VNC server through `raspi-config`. I did run into issues with applications which use OpenGL components (such as the example above), where the OpenGL graphics fail to render.

## Step 17: Configuring Qt Creator
Currently I develop my Qt apps on a Windows Machine. Therefore it doesn't have access to the cross-compiler directly. I simply copy my project source files from my Windows environment to the Ubuntu environment inside the Virtual Machine, and then run `qmake` and `make` to compile the code.
I have not yet configured Qt creator within Ubuntu to automatically cross-compile and deploy to my Raspberry Pi. 


