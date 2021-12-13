## Manual LVM Partitioning for new Linux (Kubuntu/KDE NEON/Others)

This guide will describe how to setup a new disk using LVM encrypted disk. It was used recently to install linux on a dual boot Lenovo Yoga 17 Slim 7 Pro 16. Use at your own risk.

- Disable Bitlocker
- On a fresh disk, create an EFI System Partition. On a dual boot machine, they'll be one there
- Create a 1024MB /boot partion
- Create everything else on an encrypted LVM volume
- Re-enable bitlocker

[partition-to-encrypt] - e.g nvme0n1p7 (Shruken unallocated volume)
[partition-encrypted] - e.g. nvme0n1p7
[encrypted-name] - e.g. cryptubuntu
[vgname] - e.g. vgubuntu

After complete, we should have something like the following

```
[shrink windows partition] in window
[create boot partition] in partion editor on live cd#
[create unallocated partition] in partition editor from live cd
[create encrypted partition using BASH] 
sudo cryptsetup luksFormat /dev/[partition-to-encrypt]
sudo cryptsetup luksOpen /dev/[partition-encrypted] [encrypted-name]
ls -la /dev/mapper
sudo pvcreate /dev/mapper/[encrypted-name]
sudo vgcreate vgubuntu /dev/mapper/[encrypted-name]
sudo lvcreate -n lvswap -L 1g vgubuntu
sudo lvcreate -n lvroot -L 64g vgubuntu
sudo lvcreate -n lvhome -l 100%FREE vgubuntu
sudo mkswap /dev/mapper/vgubuntu-lvswap
sudo mkfs.ext4 /dev/mapper/vgubuntu-lvroot
sudo tune2fs -l /dev/mapper/vgubuntu-lvroot | grep "Reserved"
sudo tune2fs -m 1 /dev/mapper/vgubuntu-lvroot
sudo mkfs.ext4 /dev/mapper/vgubuntu-lvhome
# Reduce redundant blocks
sudo tune2fs -m 0.1 /dev/mapper/vgubuntu-lvhome

sudo lvdisplay
[install with gui]
[continue testing]
sudo blkid
sudo mount /dev/mapper/vgubuntu-lvroot /mnt
sudo mount /dev/mapper/vgubuntu-lvhome /mnt/home
sudo mount /dev/[boot-partition] /mnt/boot
sudo mount --bind /dev /mnt/dev
sudo chroot /mnt
mount -t proc proc /proc
mount -t sysfs sys /sys
mount -t devpts devpts /dev/pts
nano /etc/crypttabXXXXXX-XXXX-XXXXXX-XXXX-XXXXXXXXX
# [target name] [source device] [key file] [options]
update-initramfs -k all -c
# Finish the install
```
### Crypttab setup

Identify the blkid to add to the cryptsetup

```
sudo blkid 
/dev/mapper/kdecrypt: UUID="tM0gKf-nXhl-3oV5-YVmb-fsR4-H5SC-zmHxzU" TYPE="LVM2_member"

# Gives something similar to, you need the encrypted parition uuid for the encrypted partition
/dev/nvme0n1p7: UUID="XXXXXXX-XXXX-XXXX-XXXXXXX-XXXXXXXXXXXX" TYPE="crypto_LUKS" PARTUUID="6f9fa279-93a4-e14b-ad60-a1890fded013"

```

Add the UUID to /etc/cryptsetup

```
cat /etc/crypttab 
# <target name> <source device> <key file> <options>
# options used:
#     luks    - specifies that this is a LUKS encrypted device
#     tries=0 - allows to re-enter password unlimited number of times
#     discard - allows SSD TRIM command, WARNING: potential security risk (more: "man crypttab")
#     loud    - display all warnings
kdecrypt UUID="XXXXXXX-XXXX-XXXX-XXXXXXX-XXXXXXXXXXX" none luks,discard4cc" none luks,discard
```

## Resulting output when configured

```
lsblk

nvme0n1            259:0    0 953.9G  0 disk  
├─nvme0n1p1        259:1    0   260M  0 part  /boot/efi
├─nvme0n1p2        259:2    0    16M  0 part  
├─nvme0n1p3        259:3    0 475.9G  0 part  
├─nvme0n1p4        259:4    0  1000M  0 part  
├─nvme0n1p6        259:5    0     1G  0 part  /boot
└─nvme0n1p7        259:6    0 475.2G  0 part  
└─kdecrypt       253:0    0 475.2G  0 crypt 
    ├─vgkde-lvswap 253:1    0    32G  0 lvm   [SWAP]
    ├─vgkde-lvroot 253:2    0   256G  0 lvm   /
    └─vgkde-lvhome 253:3    0 187.2G  0 lvm   /home
```

# References
https://www.mikekasberg.com/blog/2020/04/08/dual-boot-ubuntu-and-windows-with-encryption.html
https://www.youtube.com/watch?v=YU97K8Xavcc