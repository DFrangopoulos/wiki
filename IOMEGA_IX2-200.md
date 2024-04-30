Iomega IX2-200 - Tutorial

## 1 - File Gathering

- https://ftp.debian.org/debian/dists/stable/main/installer-armel/current/images/kirkwood/netboot/initrd.gz
- https://ftp.debian.org/debian/dists/stable/main/installer-armel/current/images/kirkwood/netboot/vmlinuz-5.10.0-9-marvell
- https://ftp.debian.org/debian/dists/stable/main/installer-armel/current/images/kirkwood/device-tree/kirkwood-iomega_ix2_200.dtb

## 2 - Setup Serial Connection

### Physical Connection
- https://brianhoskins.uk/hardware-hacking-the-iomega-storcenter-ix2-200/

### S/W Config
- Baud : 115200
- Parity/Bits : 8N1
- H/W Flow Ctrl : No
- S/W Flow Ctrl : No

## 3 - Prepare Files
- sudo apt-get install u-boot-tools

### uImage
```bash
cp vmlinuz-5.10.0-9-marvell vmlinuz-5.10.0-9-marvellwdtb
cat kirkwood-iomega_ix2_200.dtb >> vmlinuz-5.10.0-9-marvellwdtb
mkimage -A arm -O linux -T kernel -C none -n Linux -a 0x00800000 -e 0x00800040 -d vmlinuz-5.10.0-9-marvellwdtb uImage
```
### uInitrd
- `mkimage -A arm -O linux -T ramdisk -C none -n "Debian 11 ARM initrd" -a 0x01000000 -d initrd.gz uInitrd`

## 4 - Boot Command

7mb for kernel (uImage)
1mb for the fdt
to end of memeory for uInitrd

```
setenv mainlineLinux 'yes'
setenv arcNumber '1682'
setenv FanTempStart '30'
setenv FanHysteresis '3'

setenv ix_memoffset_kernel '0x00800000'
setenv ix_memoffset_initrd '0x01000000'

setenv ix_bootargs_console 'console=ttyS0,115200'
setenv ix_bootargs_root 'root=/dev/sda2 ro quiet'
setenv ix_bootargs_combine 'setenv bootargs ${ix_bootargs_console} ${ix_bootargs_root}'

setenv ix_bootlinux 'bootm ${ix_memoffset_kernel} ${ix_memoffset_initrd}'

setenv ix_usb_load_uImage 'ext2load usb 0:1 ${ix_memoffset_kernel} /uImage'
setenv ix_usb_load_uInitrd 'ext2load usb 0:1 ${ix_memoffset_initrd} /uInitrd'

setenv ix_usb_load 'run ix_usb_load_uImage; run ix_usb_load_uInitrd'

setenv ix_usb_boot 'run ix_bootargs_combine; usb reset; run ix_usb_load ; run ix_bootlinux'

setenv bootcmd 'run ix_usb_boot'
saveenv
```

### Install Options
- main interface eth1
- ext2 /boot , ext4 / , swap
- install kernel manually afterwards (with chroot + apt)
- no bootloader will be install (it's fine we have uboot)

- `chroot /target`
- `apt install linux-image-5.10.0-9-marvell initramfs-tools linux-headers-$(uname -r) build-essential lm-sensors mdadm smartmontools vsftpd`

run sensors-detect (to add lm63 to /etc/modules)

### Compile lm63
Makefile
```bash
obj-m += lm63.o

all:
    make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
    make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```
DL
- `wget https://raw.githubusercontent.com/torvalds/linux/v5.10/drivers/hwmon/lm63.c`
- `make`
- `mv lm63.ko /lib/modules/$(uname -r)/kernel/drivers/hwmon/lm63.ko`

Updates Kernel Modules + temp load
- `depmod`
- `modprobe lm63`

Update Initramfs + make new uInitrd
- `update-initramfs -u`
- `mkimage -A arm -O linux -T ramdisk -C none -n "Debian 11 ARM initrd" -a 0x01000000 -d /boot/initrd /boot/uInitrd`

`echo 1 > /sys/class/hwmon/hwmon0/device/hwmon/hwmon0/pwm1_enable`
`echo 0 > /sys/class/hwmon/hwmon0/device/hwmon/hwmon0/pwm1_enable`
`echo 29 > /sys/class/hwmon/hwmon0/device/hwmon/hwmon0/pwm1`


# Change kernel boot arg to UUID

Connect usb to another linux install and run ```blikd``` to record the UUID of /dev/sda2 (/)

In u-boot
```
setenv ix_bootargs_root 'root=UUID=36e2ee40-daec-41ee-9fb1-ad4b6f6248ef ro quiet'
saveenv
```
# Recover Passwords
```
setenv ix_bootargs_root 'root=UUID=36e2ee40-daec-41ee-9fb1-ad4b6f6248ef ro quiet init=/bin/bash'
run ix_usb_boot
# Do not save just boot
mount -o remount,rw /
passwd <user>
```


# When updating create the new uInitrd
```
# If not already triggered
update-initramfs -u
cd /boot/
mkimage -A arm -O linux -T ramdisk -C none -n "Debian 11 ARM initrd" -a 0x01000000 -d initrd.img-5.10.0-9-marvell uInitrd
```

# Install RAID SW
1 . Create the raid device 
```mdadm --create --verbose /dev/md0 --level=1 --raid-devices=2 /dev/sda /dev/sdb```

2. Wait for the syncing process to complete

3. Create a FS in the raid device

```mkfs.ext4 /dev/md0```

4. Create a mount point
```mkdir -p /mnt/md0```

5. Adjust fstab
```blkid```
```echo 'UUID=a39ac380-e676-43d2-bed0-f6a5a50ef30e /mnt/md0 ext4 defaults 0 2' | tee -a /etc/fstab```

5. Save mdadm config into mdadm.conf: 
```mdadm --detail --scan | tee -a /etc/mdadm/mdadm.conf```

6. Update initramfs


# Configure vsftpd

1. Install
2. Allow writes
3. add bind mount to fstab
```
mkdir -p /mnt/md0/MountainFTP
chown -R local:local /mnt/md0/MountainFTP
chmod -R 755 /mnt/md0/MountainFTP
echo '/mnt/md0/MountainFTP /home/local/FTP none defaults,bind 0 0' | tee -a /etc/fstab
```

# Sample UBoot UART output
[UBoot](./uboot_sample.txt)