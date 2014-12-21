pi_wifi
=======

##Download Latest Raspbian Image

<http://www.raspberrypi.org/downloads/>

```shell
cd ~/Downloads
unzip 2014-09-09-wheezy-raspbian.zip
```

##Install Raspbian

See what devices are currently mounted.

```shell
df -h
```

If your computer has a slot for SD cards, insert the card. If not,
insert the card into an SD card reader, then connect the reader to
your computer.

```shell
df -h
```

Run df -h again. The new device that has appeared is your SD card. The
left column gives the device name of your SD card; it will be listed
as something like /dev/mmcblk0p1 or /dev/sdd1. The last part (p1 or 1
respectively) is the partition number but you want to write to the
whole SD card, not just one partition. Therefore you need to remove
that part from the name (getting, for example, /dev/mmcblk0 or
/dev/sdd) as the device for the whole SD card. Note that the SD card
can show up more than once in the output of df; it will do this if you
have previously written a Raspberry Pi image to this SD card, because
the Raspberry Pi SD images have more than one partition.

Now that you've noted what the device name is, you need to unmount it
so that files can't be read or written to the SD card while you are
copying over the SD image.

```shell
# Replace sdx1 with whatever your SD card's
# device name is (including the partition number).
umount /dev/sdx1
```

If your SD card shows up more than once in the output of df due to
having multiple partitions on the SD card, you should unmount all of
these partitions.

In the terminal, write the image to the card with the command below,
making sure you replace the input file if= argument with the path to
your .img file, and the /dev/sdd in the output file of= argument with
the right device name. This is very important, as you will lose all
data on the hard drive if you provide the wrong device name. Make sure
the device name is the name of the whole SD card as described above,
not just a partition of it; for example sdd, not sdds1 or sddp1; or
mmcblk0, not mmcblk0p1.

```shell
# Replace sdd with whatever your SD card's
# device name is (excluding the partition number).
sudo dcfldd bs=4M if=2014-09-09-wheezy-raspbian.img of=/dev/sdx
```

Please note that block size set to 4M will work most of the time; if
not, please try 1M, although this will take considerably longer.

You can check what's written to the SD card by dd-ing from the card
back to another image on your hard disk, and then running diff (or
md5sum) on those two images. There should be no difference.

Run sync. This will ensure the write cache is flushed and that it is
safe to unmount your SD card.

```shell
sync
```

Remove the SD card from the card reader and install it into the
RaspberryPi.

Connect RaspberryPi to local network with and ethernet cable.

Power up the RaspberryPi using the micro USB cable.

##Setup Raspbian

ssh into raspberrypi:

```shell ssh pi@raspberrypi ```

Configure:

```shell
sudo raspi-config
```

Options to change:

```shell
8 Advanced Options
A9 Update
A3 Memory Split (16)
7 Overclock (Medium)
<Finish> (reboot yes)
```

After reboot, ssh in again and run:

```shell
sudo apt-get update
sudo apt-get dist-upgrade -y
sudo apt-get install hostapd -y
mkdir ~/drivers
```

##Setup Raspbian WiFi

Install this WiFi module from Adafruit:

<http://www.adafruit.com/products/814>

###Download Drivers from Realtek

On host machine download latest drivers from (Choose the RTL8188CUS
Linux version)

<http://www.realtek.com.tw/downloads/downloadsView.aspx?Langid=1&PNid=21&PFid=48&Level=5&Conn=4&DownTypeID=3&GetDown=false>

###Or Download Drivers from Github

On host machine run:

```shell
cd ~/Downloads
wget https://github.com/peterpolidoro/pi_wifi/blob/master/drivers/RTL8188C_8192C_USB_linux_v4.0.2_9000.20130911.zip
```

###Copy Drivers to RaspberryPi

On host machine run:

```shell
scp ~/Downloads/RTL8188C_8192C_USB_linux_v4.0.2_9000.20130911.zip pi@raspberrypi:~/drivers
```


On raspberrypi run:

```shell
cd ~/drivers
unzip RTL8188C_8192C_USB_linux_v4.0.2_9000.20130911.zip
cd RTL8188C_8192C_USB_linux_v4.0.2_9000.20130911
cd wpa_supplicant_hostapd
tar -xvf wpa_supplicant_hostapd-0.8_rtw_r7475.20130812.tar.gz
cd wpa_supplicant_hostapd-0.8_rtw_r7475.20130812
cd hostapd
sudo make
sudo make install
sudo cp hostapd /usr/sbin/hostapd
sudo chown root.root /usr/sbin/hostapd
sudo chmod 755 /usr/sbin/hostapd
sudo touch /etc/hostapd/hostapd.conf
```

Mount raspberrypi filesystem to make it more convenient to modify
files. This requires setting up a root account password.

On the raspberrypi run:

```shell
sudo -i
passwd root
sudo reboot
```

On host computer run:

```shell
sudo mkdir /mnt/raspberrypi
sudo chown $USER:$USER /mnt/raspberrypi
sshfs root@raspberrypi:/ /mnt/raspberrypi
```

Edit /mnt/raspberrypi/etc/hostapd/hostapd.conf

```shell
interface=wlan0
driver=rtl871xdrv
ssid=RaspberryPi
channel=1
wmm_enabled=0
wpa=1
wpa_passphrase=raspberry
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
auth_algs=1
macaddr_acl=0
```

Edit /mnt/raspberrypi/etc/default/hostapd

```shell
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```


Edit /mnt/raspberrypi/etc/network/interfaces

```shell
# The loopback network interface
auto lo
iface lo inet loopback
# Wired internet when connected
auto eth0
iface eth0 inet dhcp
# Wireless Setup
auto wlan0
iface wlan0 inet static
    address 10.0.0.1
    network 10.0.0.0
    netmask 255.255.255.0
    broadcast 10.0.0.255
    wireless-mode master
    wireless-essid RaspberryPi
```

Setup DNSMasq. On raspberrypi run:

```shell
sudo apt-get install dnsmasq -y
```

Edit /mnt/raspberrypi/etc/dnsmasq.conf

```shell
address=/#/10.0.0.1
interface=wlan0
dhcp-range=10.0.0.10,10.0.0.110,6h
```

Setup iptables to forward port 80 to port 5000. On raspberrypi run:

```shell
sudo iptables -A INPUT -i wlan0 -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -i wlan0 -p tcp --dport 5000 -j ACCEPT
sudo iptables -A PREROUTING -t nat -i wlan0 -p tcp --dport 80 -j REDIRECT --to-port 5000
sudo iptables-save | sudo tee /etc/iptables.sav
```

Edit /mnt/raspberrypi/etc/rc.local

```shell
iptables-restore < /etc/iptables.sav
exit 0
```

Unmount raspberrypi filesystem. On host machine run:

```shell
fusermount -u /mnt/raspberrypi
```

Disable root password and reboot. On raspberrypi run:

```shell
sudo passwd -dl root
sudo reboot
```

Use a computer/phone/tablet with WiFi connection to test raspberrypi
setup.

Connect to the RaspberryPi wireless network (password: raspberry) and
check to make sure you can connect and are assigned an IP address
from 10.0.0.10 to 10.0.0.110.

ssh into raspberrypi and shut it down

```shell
sudo shutdown now
```


##Save Modified Raspbian Image

See what devices are currently mounted on the host machine.

```shell
df -h
```

Remove the SD card from the RaspberryPi and insert it into the card
reader on the host machine.


```shell
df -h
```

Unmount all the SD card partitions:

```shell
# Replace sdx1 and sdx2 with whatever your SD card's
# device name is (including the partition number).
umount /dev/sdx1
umount /dev/sdx2
```

Save SD card to disk image:

```shell
mkdir ~/raspberrypi
cd ~/raspberrypi
# Replace sdd with whatever your SD card's
# device name is (excluding the partition number).
sudo dcfldd bs=4M if=/dev/sdx of=2014-09-09-wheezy-raspbian-wifi.img
```

##Finish Raspbian Setup

Remove the SD card from the card reader and install it into the
RaspberryPi.

Connect RaspberryPi to local network with and ethernet cable.

Power up the RaspberryPi using the micro USB cable.

ssh into raspberrypi:

```shell
ssh pi@raspberrypi
```

Configure:

```shell
sudo raspi-config
```

Options to change:

```shell
1 Expand Filesystem
<Finish> (reboot)
```
