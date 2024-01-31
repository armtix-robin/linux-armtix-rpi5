1) Download an image from https://armtix.artixlinux.org/images/
choose one with the latest date and init system.

```
wget https://armtix.artixlinux.org/images/armtix-runit-20231230.tar.xz
```

also download an alarm packages for rpi5 kernel Used used latest ie

```
wget http://mirror.archlinuxarm.org/aarch64/core/linux-rpi-6.6.14-1-aarch64.pkg.tar.xz
wget http://mirror.archlinuxarm.org/aarch64/core/linux-rpi-headers-6.6.14-1-aarch64.pkg.tar.xz
wget http://mirror.archlinuxarm.org/aarch64/alarm/rpi5-eeprom-20240118-1-any.pkg.tar.xz
wget http://mirror.archlinuxarm.org/aarch64/alarm/raspberrypi-utils-20231221-2-aarch64.pkg.tar.xz
wget http://mirror.archlinuxarm.org/aarch64/alarm/firmware-raspberrypi-20231022-1-any.pkg.tar.xz
```

2) prepare an SD card to receive armtix, declare SD as the appropriate device 
for me it's mmcblk0. The P variable is used to separate the device from the
partition number. For devices like /dev/sdX it should be an empty string.
For my device it's p

```
SD=/dev/mmcblk0
P="p"

# first ensure it's not mounted, do it manually if you don't like this for loop
for x in $(df | cut -d' ' -f1 | grep "$SD"); do sudo umount "$x"; done

# Start fdisk to partition the SD card:
sudo fdisk "$SD"
```
    At the fdisk prompt, delete old partitions and create a new one:
    Type o This will clear out any partitions on the drive.
    Type p to list partitions. There should be no partitions left.
    Type n then p for primary, 1 for the first partition on the drive, press ENTER to accept
           the default first sector, then type +200M for the last sector.
    Type t then c to set the first partition to type W95 FAT32 (LBA).
    Type n then p for primary, 2 for the second partition on the drive, and then
           press ENTER twice to accept the default first and last sector.
    Type w to write the partition table and exit

3)  Create the FAT & ext4 filesystems
    ```
    sudo mkfs.vfat "$SD$P"1
    sudo mkfs.ext4 "$SD$P"2
    ```

4)  Create the file system to receive armtix and mount it

    ```
    sudo mkdir root
    sudo mount "$SD$P"2 root
    ```
6)  extract the previously downloaded image into the filesystem note that
    we are only using the second partition as it will overflow our 200M p1

    ```
    sudo tar xf armtix-runit-20231230.tar.xz -C root
    ```

8)  clean up the image remove all the old uboot stuff

    ```
    sudo rm -rf root/boot/*

    #also clean out the old kernel modules
    sudo rm -rf root/usr/lib/modules/*
    ```

6)  now extract the downloaded alarm kernel into the two partitions

    ```
    #first mount p1
    sudo mount "$SD$P"1 root/boot
    
    sudo tar xf linux-rpi-6.6.14-1-aarch64.pkg.tar.xz -C root

    #clean up some unwanteds 
    sudo rm root/{.BUILDINFO,.INSTALL,.MTREE,.PKGINFO}

    #copying various items makes life a bit easier later
    sudo cp linux-rpi-6.6.14-1-aarch64.pkg.tar.xz root/root
    sudo cp linux-rpi-headers-6.6.14-1-aarch64.pkg.tar.xz root/root
    sudo cp rpi5-eeprom-20240118-1-any.pkg.tar.xz root/root
    sudo cp raspberrypi-utils-20231221-2-aarch64.pkg.tar.xz root/root
    sudo cp firmware-raspberrypi-20231022-1-any.pkg.tar.xz root/root
    ```

8)  unmount both drives

    ```
    sudo umount root/boot root
    ```

    you should now have a bootable sd card.

9)  If the card boots for you you should at least clean up the install a bit
    As root I did the following after which the fan control started to work.

    ```
    cd /root
    pacman -U linux-rpi-6.6.14-1-aarch64.pkg.tar.xz firmware-raspberrypi-20231022-1-any.pkg.tar.xz --overwrite='/boot/*' --overwrite='/usr/lib/modules/*' --overwrite=/etc/mkinitcpio.d/linux-rpi.preset
    pacman -U linux-rpi-headers-6.6.14-1-aarch64.pkg.tar.xz rpi5-eeprom-20240118-1-any.pkg.tar.xz raspberrypi-utils-20231221-2-aarch64.pkg.tar.xz
    sudo reboot
    ```

10)  If you want to use an nvme card you can add this to /boot/config.txt
   
     ```
     [pi5]
     dtparam=nvme
     ```

     To make X11 work I had to add this file /etc/X11/xorg.conf.d/99-v3d.conf 

     ```
     Section "OutputClass"
         Identifier "vc4"
         MatchDriver "vc4"
         Driver "modesetting"
         Option "PrimaryGPU" "true"
     EndSection
     ```
