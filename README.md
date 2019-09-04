# Hetzner (EX Series) with encrypted HDD incl. Software RAID1 and Debian 10 (Buster)
Hetzner root server full disk encryption with cryptsetup LUKS and SWRAID1 on NVME storage


### Boot into rescue mode and OS installation

1. Set your root server into rescue mode in Hetzner Robot (https://robot.your-server.de).
<br>After login via ssh, you can install your favorite OS - in this tutorial we will use Debian 10 (Buster).
```
installimage
```

2. Edit Hetzner OS configuration
```
# RAID setup
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
debootstrap buster /mnt http://ftp.de.debian.org/debian/
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
echo "nameserver 46.182.19.48" > /etc/resolv.conf
echo "nameserver 204.152.184.76" > /etc/resolv.conf
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
  sudo \
  unzip \
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
passwd root
```

9. Adding your public_key to dropbear
```
cd /etc/dropbear-initramfs/
nano authorized_keys
```
```
no-port-forwarding,no-agent-forwarding,no-X11-forwarding,command="/bin/cryptroot-unlock" ssh-rsa AAAXXX_YOUR_PUBLIC_KEY
```

10. Changing dropbear listening port
```
nano config
```
```
DROPBEAR_OPTIONS="-p 5799 -s -j -k -I 60"
```

11. Update initramfs
```
update-initramfs -u -k all
```

### Network configuration (choose between 1 and 2 - BE CAREFUL: Don't use both methods)
**Notice: Your network device name may be different**

1. Using systemd-networkd
```
cd /etc/systemd/network#
nano *.network
```
```
[Match]
Name=enp0s31f6

[Network]
## IPv6
Address=YOUR.IPv6.ADDRESS/MASK
Gateway=YOUR.IPv6.GATEWAY

## IPv4
Address=YOUR.IPv4.ADDRESS/MASK
Gateway=YOUR.IPv4.GATEWAY
```
```
systemctl enable systemd-networkd
```

2. Using network interfaces
```
source /etc/network/interfaces.d/*
```
```
cat > /etc/network/interfaces.d/lo << EOF
auto lo
iface lo inet loopback
EOF
```
```
cat > /etc/network/interfaces.d/enp0s31f6 << EOF
auto enp0s31f6
iface enp0s31f6 inet static
  address YOUR.IPv4.ADDRESS
  netmask YOUR.IPv4.NETMASK
  gateway YOUR.IPv4.GATEWAY

iface enp0s31f6 inet6 static
  address YOUR.IPv6.ADDRESS
  netmask YOUR.IPv6.NETMASK
  gateway YOUR.IPv6.GATEWAY
EOF
```

### Finishing

1. Hardening openssh - you should also disable root login later if everything works!
```
cd /etc/ssh/
nano sshd_config
```
```
KexAlgorithms diffie-hellman-group-exchange-sha256,diffie-hellman-group16-sha512,curve25519-sha256@libssh.org
Ciphers aes128-ctr,aes192-ctr,aes256-ctr
Macs hmac-sha2-256,hmac-sha2-512
HostKeyAlgorithms ssh-ed25519,rsa-sha2-256,rsa-sha2-512
```
```
systemctl enable ssh
```

2. Update initramfs + grub
```
update-initramfs -u -k all
update-grub
```

3. Finalizing and reboot
```
exit
```
```
# Cleanup
sync
umount -R /mnt
cd ..
```
```
reboot
```


### Troubleshooting

If something goes wrong and you need to boot again into your crypted os, please set your server back in rescue mode.

```
cryptsetup luksOpen /dev/md1 cryptroot
```
```
mount /dev/mapper/cryptroot /mnt
mount /dev/md0 /mnt/boot
mount -t proc proc /mnt/proc/
mount -t sysfs sys /mnt/sys/
mount -t devtmpfs dev /mnt/dev/
mount -t devpts devpts /mnt/dev/pts
mount -t tmpfs tmpfs /mnt/tmp
chroot /mnt /bin/bash
. /etc/profile
```

Your are now back in your encrypted OS and can change everything what you need.
