# Kali WSL2 - Custom Kernel and `rtl8812au` driver installation

:pushpin: This tutorial demonstrates how to build and use a custom kernel for WSL2 distros.

## Setup Kali WSL2

* Disable password prompt when using `sudo` command :
```
echo "kevin ALL=(ALL) NOPASSWD: ALL" | sudo tee -a /etc/sudoers.d/010_kevin-nopasswd > /dev/null
```

* Update the system :
```
sudo apt update && sudo apt upgrade
```

* Install useful packages :
```
sudo apt install bash-completion build-essential gcc g++ avr-libc avrdude default-jre default-jdk git clang make nano xz-utils usbip wget
source .bashrc
```

* Install Kali packages :
```
sudo apt install kali-linux-large
```

* Setup remote access :
```
sudo apt install kali-win-kex
kex --esm -s
```

## Build and install a custom WSL2 kernel

* Install required packages :
```
sudo apt install flex bison libssl-dev libelf-dev git dwarves bc
```

* Download official WSL2 kernel and prepare the installation :
```
wget https://github.com/microsoft/WSL2-Linux-Kernel/archive/refs/tags/linux-msft-wsl-$(uname -r | cut -d- -f 1).tar.gz
tar -xvf linux-msft-wsl-$(uname -r | cut -d- -f 1).tar.gz
cd WSL2-Linux-Kernel-linux-msft-wsl-$(uname -r | cut -d- -f 1)
cat /proc/config.gz | gunzip > .config
make prepare modules_prepare -j $(expr $(nproc) - 1)
```

* Open the kernel's configuration menu to add `cfg80211` wireless modules (802.11 protocols) :
```
make menuconfig -j $(expr $(nproc) - 1)
```

* Build and install modules :
```
make modules -j $(expr $(nproc) - 1)
sudo make modules_install
make -j $(expr $(nproc) - 1)
sudo make install
```
Note : Kernel headers are going to be installed in the `/lib/modules/` directory.

* Copy the built kernel image to `C:\Users\<User>\` :
```
cp vmlinux /mnt/c/Users/Kevin/
```

* Create a `.wslconfig` file to declare the new kernel :
```
nano /mnt/c/Users/Kevin/.wslconfig
```

* Paste the following content into this file :
```
[wsl2]
kernel=C:\\Users\\Kevin\\vmlinux
```

* Switch to Powershell and shutdown running WSL2 distros :
```
wsl --shutdown
```
:triangular_flag_on_post: When a WSL2 distro will be rebooted, the default WSL2 kernel located in `C:\Windows\System32\lxss\tools\kernel` will be replaced by the newly built kernel.

## Compile and load a kernel module

Note : This example illustrates how to build and load the `rtl8812au` module to the WSL2 kernel.

* Clone the `aircrack-ng/rtl8812au` Github repository :
```
git clone https://github.com/aircrack-ng/rtl8812au
cd rtl8812au
```

* Build the module with the new kernel headers :
```
sudo make
```
:white_check_mark: This commands generates a `.ko` file which correspond to the built module.  
:warning: The headers must be installed in the `/lib/modules/$(uname -r)/build` directory.  
You can check your WSL2 version by running `uname -r`.

Test : Enable `cfg80211` and `88XXau.ko` modules :
```
sudo modprobe cfg80211
sudo insmod 88XXau.ko
lsmod
```
:warning: `cfg80211` module must be loaded before `88XXau.ko`.

* Install the module :
```
sudo mkdir -p /lib/modules/$(uname -r)/kernel/drivers/net/wireless
sudo make install
```
Note : Now, `sudo modprobe 88XXau` command will work.

* Enable the module :
```
sudo modprobe 88XXau
```

## Load modules at boot time

* Add modules that need to be loaded at boot time to the `/etc/modules-load.d/` directory :
```
echo "cfg80211" | sudo tee -a /etc/modules-load.d/cfg80211.conf
echo "88XXau" | sudo tee -a /etc/modules-load.d/88XXau.conf
```
:warning: Don't add `.ko` extension to the module name.

## Wi-Fi adapter connection using `rtl8812au` driver :

* Attach a USB device using `usbip` :
```
sudo usbip list --remote=<IP>
sudo usbip attach --remote=<IP> --busid=<BUS-ID>
ip a
```

* Install `aircrack-ng` packages :
```
sudo apt install aircrack-ng pciutils
```

* Set the adapter in monitor mode :
```
sudo airmon-ng
sudo airmon-ng start wlan0
```

* Start to capture packets with `airodump-ng` :
```
sudo airodump-ng wlan0
```

* Check if monitor mode is successfully enabled on `wlan0` interface :
```
ip a
```

* Disable monitor mode on `wlan0` interface :
```
airmon-ng stop wlan0
```

* Detach USB device :
```
sudo usbip port
sudo usbip detach --port=
```

## Create `usbip` scripts

### Attach a USB device

* Create and edit a file called `musb` :
```
nano musb
```

* Add the following content to this file :
```
#!/bin/bash

if ! [ $(id -u) = 0 ]; then
   echo "The script need to be run as root." >&2
   exit 1
fi

echo "Enter the IP address of the foreign device :"
read ip

echo ""
usbip list -r $ip

echo "Enter the name of the shared USB device to import :"
read usb

echo ""
usbip attach -r $ip -b $usb

sleep 1
usbip port
```

* Add execution permissions to the file :
```
chmod +x musb
```

* Run the script :
```
sudo ./musb
```

### Detach a USB device


* Create and edit a file called `musb` :
```
nano uusb
```

* Add the following content to this file :
```
#!/bin/bash

if ! [ $(id -u) = 0 ]; then
   echo "The script need to be run as root." >&2
   exit 1
fi

usbip port

echo ""
echo "Enter the USB device to detach :"
read usb

echo ""
usbip detach -p $usb

sleep 1
usbip port
```

* Add execution permissions to the file :
```
chmod +x uusb
```

* Run the script :
```
sudo ./uusb
```

## References

* [Microsoft Github Repository - WSL2-Linux-Kernel](https://github.com/microsoft/WSL2-Linux-Kernel)
* [WSL2 `/lib/modules` Stack Exchange Discussion](https://unix.stackexchange.com/questions/594470/wsl-2-does-not-have-lib-modules)
* [Github Gist - Custom WSL2 Kernel Build](https://gist.github.com/charlie-x/96a92aaaa04346bdf1fb4c3621f3e392)
* [`RTL8812AU` Driver Installation On RPI](https://raspberrypi.stackexchange.com/questions/120134/install-drivers-for-rtl8812au-for-raspibian-kernel-5-4-79-v71-rpi-4)
* [Basic `RTL8812AU` Drive Configuration](https://adam-toscher.medium.com/configure-your-new-wireless-ac-1fb65c6ada57)
* [`aircrack-ng` Deauthentication Attack](https://hackernoon.com/forcing-a-device-to-disconnect-from-wifi-using-a-deauthentication-attack-f664b9940142)
* [Windows `aireplay-ng` Packet Injection](https://web.archive.org/web/20080921000952/http://airdump.net/aireplay-packet-injection-windows/)
* [Wireless Capture On Windows](https://blog.packet-foo.com/2019/04/wireless-capture-on-windows/comment-page-1/)
* [NPCAP Developer Guide](https://npcap.com/guide/npcap-devguide.html)
* [USB-IP For Windows](https://github.com/kevin-doolaeghe/usbip-win)
* [WSL2 Setup Guide](https://learn.microsoft.com/fr-fr/windows/wsl/install)
* [Kali Tools Guide](https://www.kali.org/tools/)
