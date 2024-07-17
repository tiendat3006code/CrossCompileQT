This is a steps to customize Raspberry Pi’s boot up screen and run QT Application on Boot

### 1. Disable Rainbow image
Open “/boot/config.txt” as root.

```sh
  sudo nano /boot/config.txt
```

Then add below line at the end of the file.

```sh
disable_splash=1
```

### 2. Remove Boot Messages and other
Open “/boot/cmdline.txt” as root.

```sh
sudo nano /boot/cmdline.txt
```

Then, replace “console=tty1” with “console=tty3”. This redirects boot messages to tty3.

Still in “/boot/cmdline.txt”, add below at the end of the line
```sh
splash quiet logo.nologo vt.global_cursor_default=0 pymouth.enable=0
```

| Options | Explanations |
| ---------- | :--------: |
| splash | enables splash image |
| quiet | disable boot message texts |
| logo.nologo | emoves Raspberry Pi logo in top left corner |
| vt.global_cursor_default=0 | removes blinking cursor |
| pymouth.enable=0 | Disable pymouth (use fbi instead)  |


### 3. Replace Splash Image
#### 1. Copy your splash image to /home/pi/splash.png

#### 2. Install fbi

```sh
sudo apt install fbi
```

#### 3. Create service file

```sh
sudo nano /file/systemd/system/splashscreen.service
```

with contents like following:

```
[Unit]
Description=Splash screen
DefaultDependencies=no
After=local-fs.target

[Service]
ExecStart=/usr/bin/fbi -d /dev/fb0 --noverbose -a /home/pi/splash.png
StandardInput=tty
StandardOutput=tty

[Install]
WantedBy=sysinit.target
```

#### 4. Enable service
```sh
sudo systemctl enable splashscreen
```

### 4. Boot Rasperberry to CLI
Open Rasperbarry Configuration Tool by command
```sh
sudo raspi-config 
```
then go to: System Options -> Boot / Auto Login -> Console Autologin

### 5. Run QT Application on Boot
#### 1. Prepare EGFLS config
Edit file 
```sh
sudo nano /home/pi/eglfs.json
```
Add following lines
```
{
  "device": "/dev/dri/card1",
  "hwcursor": false,
  "pbuffers": true,
  "outputs": [
    {
      "name": "HDMI1",
      "mode": "800x480"
    }
  ]
}
```
#### 2. Edit rc.local
Run command
```sh
sudo nano /etc/rc.local
```
Add following lines
```
export QT_QPA_EGLFS_HIDECURSOR=1
export QT_QPA_EGLFS_KMS_CONFIG=/home/pi/eglfs.json
su - pi -c /home/pi/your_qt_applicationn &
```
### 6. Check result
Reboot device by command
```sh
sudo reboot
```