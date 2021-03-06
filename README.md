# rpi-raid1-root-partition

This guide is here to help future me, but also anyone else reading this.

The goal here is to have a RPi boot from MicroSD but only the boot partition, while the `/root` will be constituted by 2 x External Drives in a RAID1 array. In my case, that is 2 SSDs connected to USB3.0 via USB to Sata adapters.

The reasoning behing this is that writing to a MicroSD will eventually kill it, and for my use case, I didn't want to rebuild very often or have a separate device to back up on. So by using the MicroSD just at boot time, and shifting everything else into the array, anything that matters will always live on 2 drives, and if one fails, in principle it can be simply swapped and the array rebuilt.

## How to

1. Flash your RaspiOS to your MicroSD. You can keep this as small as possible, but capable to hold the whole of the OS. In my case I installed the RaspiOS Buster arm64 onto a 32GB SanDisk Ultra (overkill, the smallest one I had)
2. Activate ssh by creating a filed called `ssh` on `/boot` partition
3. If planning on using WiFi, setup your `wpa_supplicant.conf` file on `/boot` partition
4. Put the SD in your RPi and boot
5. Do your basic setup (you can also do all this at the end) - hostname, change password
6. Update the OS and reboot
```
sudo apt-get update
sudo apt-get upgrade
sudo reboot
```
7. Prepare your drives. If the drives you have come with a partition, it will be gone. Also, before you blindly copy and paste commands check where your drives are. Use `df -h` and `lsblk` to double check
```
sudo fdisk /dev/sda
d - delete partition
n - new partition
w - save

sudo fdisk /dev/sdb
d - delete partition
n - new partition
w - save

sudo mkfs.ext4 /dev/sda1
sudo mkfs.ext4 /dev/sdb1
```
9. Install mdadm
```
sudo apt install mdadm
```
10. Create the RAID1 array.
```
sudo mdadm --create --verbose /dev/md0 --level=1 --raid-devices=2 /dev/sda1 /dev/sdb1
```
11. Monitor the process - this will take some time, depending on the drives size. Wait until it completes and then proceed.
```
watch -n 3 cat /proc/mdstat
```
12. Format the array. This is technically redundant as the 2 drives are already `ext4`. But better to do it to be sure. Won't take long.
```
sudo mkfs.ext4 /dev/md0
```
13. Mount the array to a temporary location. Give user `pi` ownership of the location.
```
sudo mkdir /media/raid
sudo chown pi:pi /media/raid
sudo mount /dev/md0 /media/raid
```
14. Get RAID info
```
sudo mdadm --detail --scan
```
In my case:
```
ARRAY /dev/md0 metadata=1.2 name=hostingpi:0 UUID=152ceb27:2de4e2cf:de8f3fe2:7ae0e609
```
15. Copy the exact output at the bottom of the config file
```
sudo nano /etc/mdadm/mdadm.conf
```
16. Add modules to bootloader 
```
sudo nano /etc/initramfs-tools/modules
```
```
raid1
md_mod
ext4
```
17. Mount `/root` and rsync it.
```
sudo mkdir /media/sdrootfs
sudo mount /dev/mmcblk0p2 /media/sdrootfs
sudo rsync -axv /media/sdrootfs/* /media/raid
```
18. Update initramfs
```
sudo mkinitramfs -o /boot/initramfs-raid.gz
```
19. Add information about the kernel and initramfs to the boot config
```
sudo nano /boot/config.txt
```
```
kernel=kernel8.img
initramfs initramfs-raid.gz
```
Check your version of the kernel in `/boot`. In my case (rpi 4 x64) it was called `kernel8.img`

20. `sudo reboot`
21. Edit boot cmdline
```
sudo nano /boot/cmdline.txt
```
Change `root` to `/dev/md0` and add `rootdelay=5` at the end of the line. If you notice that the array is not assembled at boot time, increase the number as needed e.g. `rootdelay=10`

22. Mount the array to the temporary location again
```
sudo mount /dev/md0 /media/raid
```
23. Edit the `fstab` file located on the array.
```
sudo nano /media/raid/etc/fstab
```
24. Delete/comment out the old `/rootfs` partition line and replace with the array information
```
/dev/md0 / ext4 defaults,noatime,errors=remount-ro 0 1
```
25. `sudo reboot`
26. `sudo rm -rf /media/raid`
27. OPTIONAL but recommended step. Power off your RPi, plug your MicroSD into your laptop and delete the old `/root` partition.

## Important
### Kernel Updates
Whenever a kernel update happens, an intervention to update the initramfs is needed. You can check `ls -la /lib/modules` for the latest installed version. This will be different from `uname -r` if indeed there was an update applied. Failing to update this will render your system unusable. Do not reboot without fixing this!!!
```
sudo mkinitramfs -o /boot/initramfs-raid.gz new-version
```

An alternative is to create a custom script that hooks into the kernel update process. Create a new file `/etc/kernel/postinst.d/raid-update` with the following content. Make sure to use the correct kernel name based on your device and architecture (kernel8 in my case). The script will be passed the installed kernel version as the first parameter thus creating the correct image.
```
#!/bin/bash
if [ "x$2" != "x/boot/kernel8.img" ]; then
	exit 0
fi

echo ============ UPDATE INITRAMFS ==============
mkinitramfs -o /boot/initramfs-raid.gz $1
echo ============ UPDATE COMPLETED ==============
```
```
sudo chmod 755 /etc/kernel/postinst.d/raid-update
```

## Enjoy!

## Source
Original inspiration guide

https://www.raspberrypi.org/forums/viewtopic.php?f=28&t=306729&p=1834954&hilit=raid+1+root

Useful comments

https://www.raspberrypi.org/forums/viewtopic.php?f=29&t=314453&p=1881821&hilit=raid+1+root

Generic raid array setup

https://jlamoure.net/blog/raspberry-pi-raid-1/

Similar setup - automatic script to regenerate initramfs

https://www.raspberrypi.org/forums/viewtopic.php?f=29&t=319427

Thread where the challenges of this process have been overcome

https://www.raspberrypi.org/forums/viewtopic.php?f=29&t=320003&p=1916845#p1916845
