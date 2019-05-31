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

### Beginn encryption

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

4. Mount crypted volume and mount it into /mnt/boot
```
mount /dev/mapper/cryptroot /mnt
mkdir -p /mnt/boot
mount /dev/md0 /mnt/boot
cd /mnt
rm -rf lost+found
ls -ls /mnt
```
