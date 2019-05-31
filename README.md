# Hetzner (EX Series) with encrypted HDD incl. Software RAID1 and Debian 9 (Stretch)
Hetzner root server full disk encryption with cryptsetup LUKS and SWRAID1 on NVME storage


### Boot into rescue mode and OS installation

1. Set your root server into rescue mode in Hetzner Robot (https://robot.your-server.de).
<br>After login via ssh, you can install your favorite OS - in this tutorial we will use Debian 9 (Stretch).
```
installimage
```

2. Edit Hetzner OS configuration
```
# Partitioning
SWRAID 1                   # 0 = No RAID; 1 = RAID
SWRAIDLEVEL 1              # 0 = Striping; 1 = Mirrorin

# Hostname
HOSTNAME your.hostname

# Partition
PART /boot ext4 512M
PART / ext4 all
```

3. After that start the install routine with `F2 (save)` + `F10 (quit editor)` and confirm with `OK`.

### Encryption of main partition

1. Encrypting - main partition (/dev/md1)
```
cryptsetup --cipher=aes-xts-plain --verify-passphrase --key-size=512 luksFormat /dev/md1
```

2. Give crypted volume an alias
```
cryptsetup luksOpen /dev/md1 cryptroot
```

3. Format crypted volume to ext4
```
mkfs.ext4 -L root /dev/mapper/cryptroot
```

### Installing minimal Debian OS in encrypted HDD

1. Mount crypted volume and mount it into /mnt/boot
```
mount /dev/mapper/cryptroot /mnt
mkdir -p /mnt/boot
mount /dev/md0 /mnt/boot
cd /mnt
rm -rf lost+found
ls -ls /mnt
```

2. Install the latest Debian release (Stretch)
```
debootstrap stretch /mnt http://ftp.de.debian.org/debian/
```

3. Mount OS folders
```
mount -t proc proc /mnt/proc/
mount -t sysfs sys /mnt/sys/
mount -t devtmpfs dev /mnt/dev/
mount -t devpts devpts /mnt/dev/pts
mount -t tmpfs tmpfs /mnt/tmp
```

4. Changing boot order
```
genfstab -pL /mnt >> /mnt/etc/fstab
cat /mnt/etc/fstab
```

### Switching (chroot) into fresh installed OS + configuration

1. Chroot to installed OS
```
chroot /mnt /bin/bash
. /etc/profile
```

2. Check for updates
```
apt update -y && apt upgrade -y
```

3. Adding Google Nameserver to /etc/resolv.conf
```
echo "nameserver 8.8.8.8" > /etc/resolv.conf
echo "nameserver 8.8.4.4" > /etc/resolv.conf
```

4. Editing crypttab
```
nano /etc/crypttab
```
```
cryptroot /dev/md1 none luks,allow-discards
```

5. Installing needed Debian packages
```
apt install -y \
  vim \
  curl \
  linux-base \
  linux-image-amd64 linux-headers-amd64 \
  grub-pc \
  grub2-common \
  mdadm \
  cryptsetup \
  lvm2 \
  initramfs-tools \
  openssh-server \
  dropbear-initramfs \
  locales \
  console-data \
  htop \
  net-tools
```

6. Set initramfs parameters
```
echo "DEVICE=eth0" >> /etc/initramfs-tools/initramfs.conf
```

7. Setup locales
```
dpkg-reconfigure locales
sed -i '/de_DE.UTF-8/s/^#//' /etc/locale.gen
sed -i '/en_US.UTF-8/s/^#//' /etc/locale.gen
locale-gen
echo -e 'LANG="en_US.UTF-8"\nLANGUAGE="en_US.UTF-8"\nLC_ALL="en_US.UTF-8"\n' > /etc/default/locale
```

8. Set Root Password
```
echo -e "root\nroot" | passwd root
```

9. Adding your public_key to dropbear
```
cd /etc/dropbear-initramfs/
nano authorized_keys
```
```
no-port-forwarding,no-agent-forwarding,no-X11-forwarding,command="/bin/cryptroot-unlock" ssh-rsa AAAXXX_YOUR_PUBLIC_KEY

```
