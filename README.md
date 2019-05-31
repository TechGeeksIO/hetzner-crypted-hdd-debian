# Hetzner (EX Series) with encrypted HDD incl. Software RAID1 and Debian 9 (Stretch)
Hetzner root server full disk encryption with cryptsetup LUKS and SWRAID1 on NVME storage

1. Set your root server into rescue mode in Hetzner Robot (https://robot.your-server.de).
<br>After login via ssh, you can install your favorite OS - in this tutorial we will use Debian 9 (Stretch).
<br><br>```installimage```

2. Edit Hetzner OS configuration
<br><br>`HOSTNAME goer.li

Partition
PART /boot ext4 512M
PART / ext4 all`
