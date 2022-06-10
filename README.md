# Razer Blade Stealth 13 (RZ09-03101E72)

![Razer Blade Stealth 13](https://i.imgur.com/mgSF7Gb.png)

## Overview

The purpose of this guide is to provide a step-by-step guide to install Debian 11 on the Razer Blade Stealth 13 and solve the known Issues on the fly.

Before reading this guide:

* Everything is included in the guide, so if you have problem, read the guide carefully again or search in the guide.
* If you follow the guide and have problems, please post your questions on the [linux corner](https://insider.razer.com/index.php?forums/linux/).
* Every command with **"psykher"** word means to be replaced with your user.

**Note:** I recommend updating to the latest BIOS on windows before following the guide.


## Preparing UEFI USB

Download: [Debian 11 + nonfree](https://cdimage.debian.org/cdimage/unofficial/non-free/cd-including-firmware/)
Download: [Rufus](https://rufus.ie/en/)

Open Rufus and select the options as bellow:

> Partition scheme: MBR<br>
> Target System: UEFI<br>
> File System: FAT32<br>
> Cluster Size: 4096 bytes<br>
> Write in DD image mode<br>


## BIOS settings

Set BIOS to defaults and make sure:

> UEFI boot is enabled<br>
> Secure Boot is disabled<br>
> Fast boot is disabled (you can turn it back on after finishing installation)<br>


## Installing

Boot from the USB drive and select "Graphical Install":

* Choose your language, location and keyboard layout
* Set hostname (computer name)
* Leave the Domain name blank
* Choose your root password
* Set your user display name
* Set your username for your account
* Choose your password for your account
* Configure your clock
* Partition disks:
  * I recommend to use Guided with encrypted LVM
  * Select your disk for partitioning
  * Choose All files in one partition
  * Configure LVM: choose Yes
  * Don't change the default amount
  * Write changes to disks?: Yes
  * Don't scan for extra installation media
  * Choose your package manager: your country if available
  * Debian archive mirror: deb.debian.org
  * HTTP Proxy: leave it blank
  * Choose your desktop environment: debian standard desktop environment, GNOME, system utilities
  * Install GRUB boot loader to your primary drive
  * Choose your disk

**Note:** In case that you peripherals are not being detected during the install please attach an external one to be able to complete the installation, it will be fixed later.


## Post Installation

### **[-]** In order to use sudo you may need to add your user to the sudoers file, open a terminal and type:

```
$ su - root
# nano /etc/sudoers
```

Add your user right after root and save:

> root       ALL=(ALL:ALL) ALL<br>
> psykher    ALL=(ALL:ALL) ALL<br>

**Note:** You won't need root anymore on this guide so let's logout with the exit command.

```
# exit
```


### **[-]** Now let's fix the repositories sources:

```
$ sudo nano /etc/apt/sources.list
```

You can replace all with this and save:

> deb http://deb.debian.org/debian/ bullseye main non-free contrib<br>
> deb-src http://deb.debian.org/debian/ bullseye main non-free contrib<br>
> <br>
> deb http://security.debian.org/debian-security bullseye-security main contrib non-free<br>
> deb-src http://security.debian.org/debian-security bullseye-security main contrib non-free<br>
> <br>
> deb http://deb.debian.org/debian/ bullseye-updates main contrib non-free<br>
> deb-src http://deb.debian.org/debian/ bullseye-updates main contrib non-free<br>


### **[-]** Next thing to do is install the latest kernel, nvidia drivers and intel CPU firmware:

```
$ sudo apt-get update
$ sudo apt-get install -y build-essential dkms linux-headers-$(uname -r|sed 's,[^-]*-[^-]*-,,') nvidia-detect nvidia-driver firmware-misc-nonfree mesa-utils nvidia-cuda-toolkit intel-microcode
$ sudo dpkg --add-architecture i386 && sudo apt-get update
$ sudo apt-get install -y nvidia-driver-libs:i386
```

I recommend to block and remove the previous default driver to avoid future issues:

```
$ sudo nano /etc/modprobe.d/blacklist-nouveau.conf
```

Paste this and save:

> blacklist nouveau<br>
> blacklist lbm-nouveau<br>
> options nouveau modeset=0<br>
> alias nouveau off<br>
> alias lbm-nouveau off<br>

Then let's update the initramfs and remove the default driver:

```
$ echo options nouveau modeset=0 | sudo tee -a /etc/modprobe.d/nouveau-kms.conf
$ sudo update-initramfs -u
$ sudo apt-get remove --purge xserver-xorg-video-nouveau
```

Reboot your system to enable the nvidia driver:

```
$ systemctl reboot
```

Now verify that the nvidia driver is loading properly:

```
$ lspci -k | grep -EA3 'VGA|3D|Display'
```


### **[-]** Closing / opening lid is not being recognized, so let's fix it with a grub update:

```
$ sudo nano /etc/default/grub
```

First section should have these entries:

> GRUB_TIMEOUT_STYLE=hidden<br>
> GRUB_HIDDEN_TIMEOUT_QUIET=true<br>
> <br>
> GRUB_DEFAULT=0<br>
> GRUB_TIMEOUT=0<br>
> GRUB_GFXMODE="1920x1080x32"<br>
> GRUB_DISTRIBUTOR=\`lsb_release -i -s 2> /dev/null || echo Debian\`<br>
> GRUB_CMDLINE_LINUX_DEFAULT="quiet splash loglevel=0 button.lid_init_state=open mem_sleep_default=deep"<br>

Update the grub and remove the boot messages:

```
$ sudo update-grub
$ sudo nano /etc/init.d/rcS
```

Paste this and save:

> exec /etc/init.d/rc S >/dev/null 2>&1<br>


### **[-]** There are still some issues when suspending the laptop, let's fix them:

> Settings -> Power -> Automatic Suspend (When on battery power)<br>

**Note:** this setting must be set! Otherwise closing the lid won't suspend the laptop.

Now lets fix the timeout response when suspending / awaking:

```
$ sudo nano /etc/systemd/system.conf
```

Change the following lines with these values:

> #DefaultTimeoutStartSec=10s<br>
> #DefaultTimeoutStopSec=10s<br>


### **[-]** Now let's fix the webcam reloading the uvcvideo kernel module with the "quirks=512" argument:

```
$ sudo nano /etc/modprobe.d/razer.conf
```

Add this line to the file and save:

> options uvcvideo quirks=512<br>


### **[-]** If you have bluetooth disconnection / re-connection issues, you can execute the following to permanent fix them:

```
$ sudo nano /etc/bluetooth/main.conf
```

Comment out and change these values:

> FastConnectable = true<br>
> ReconnectAttempts=5<br>
> ReconnectIntervals=1,2,4,8,16,32,64<br>

Finally remove your bluetooth device (manual), whitelist it and connect again:

```
$ bluetoothctl
$ power on
$ scan on
$ pair [MAC Address]
$ trust [MAC Address]
$ connect [MAC Address]
```

**Note:** This last step should be done for all your bluetooth devices

When done you can exit the bluetooth interface with the exit command:

```
$ exit
```


### **[-]** Next thing is fix the laptop battery efficiency

```
$ sudo apt-get install -y tlp tlp-rdw
$ sudo nano /etc/tlp.conf
```

Look for this entry and change the value to 0

> USB_AUTOSUSPEND=0<br>

Then enable the service:

```
$ sudo tlp start
$ sudo tlp-stat -b
$ sudo systemctl enable tlp.service
```


### **[-]** Now let's enable the Network Manager

```
$ sudo nano /etc/NetworkManager/NetworkManager.conf
```

It should look like this:

> [ifupdown]<br>
> managed=true<br>

And restart the service:

```
$ sudo service NetworkManager restart
```


### **[-]** Since this is a 16GB fixed RAM laptop, let's fix the memory swap

```
$ sudo nano /etc/sysctl.conf
```

Change this value to 10 and save:

> vm.swappiness=10<br>


### **[-]** Finally you need to install some Razer tools and drivers:

```
$ sudo apt-get install -y curl
```

Add your user to the razer group:

```
$ sudo gpasswd -a psykher plugdev
```

Download and install Open Razer:

```
$ echo 'deb http://download.opensuse.org/repositories/hardware:/razer/Debian_11/ /' | sudo tee /etc/apt/sources.list.d/hardware:razer.list
$ curl -fsSL https://download.opensuse.org/repositories/hardware:razer/Debian_11/Release.key | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/hardware_razer.gpg > /dev/null
$ sudo apt-get update
$ sudo apt-get install -y openrazer-meta
```

Download and install Polychromatic:

```
$ echo "deb http://ppa.launchpad.net/polychromatic/stable/ubuntu focal main" | sudo tee /etc/apt/sources.list.d/polychromatic.list
$ sudo apt-key adv --recv-key --keyserver keyserver.ubuntu.com 96B9CD7C22E2C8C5
$ sudo apt-get update
$ sudo apt-get install -y polychromatic
```

Restart your system:

```
$ systemctl reboot
```

You are done, that's pretty much it! You can change your lights color with the polychromatic tool, find it on your apps icons.

Note: If you find more issues / fixes please raise a PR to keep this updated.
