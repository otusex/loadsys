# System without /boot part, only LVM


I have system without LVM

Do it, Install lvm2

```
yum install -y lvm2
```

```bash
NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sda      8:0    0  40G  0 disk
`-sda1   8:1    0  40G  0 part /
sdb      8:16   0   2G  0 disk
sdc      8:32   0   2G  0 disk
```

Create part on /dev/sdb
```bash
fdisk /dev/sdb

fdisk /dev/sdb -l

Disk /dev/sdb: 2147 MB, 2147483648 bytes, 4194304 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x39e5f0e2

   Device Boot      Start         End      Blocks   Id  System
/dev/sdb1   *        2048     4194303     2096128   8e  Linux LVM
```


Init PV, need set --bootloaderareasize 1M

```bash
pvcreate /dev/sdb1 --bootloaderareasize 1M
```


Init VG

```bash
vgcreate otus_root /dev/sdb1
```

Init LV

```bash
lvcreate -n root -l 100%FREE otus_root

/dev/otus_root/root
```

Install filesystem on /dev/otus_root/root

```bash
mkfs.ext4 /dev/otus_root/root
```


Create tmp root folder and mount /dev/mapper/otus_root-root

```bash
mkdir /mnt/tmproot
mount /dev/mapper/otus_root-root /mnt/tmproot/
```

Sync data

```bash
rsync -auxvHP / /mnt/tmproot/ 
rsync -auxvHP /boot /mnt/tmproot/
```

Mount filesys

```bash
mount --rbind /dev/ /mnt/tmproot/dev; mount --rbind /proc /mnt/tmproot/proc; mount --rbind /sys /mnt/tmproot/sys; mount --rbind /run /mnt/tmproot/run
```

Chroot

```bash
chroot /mnt/tmproot/
```


Grub2 PATCH

```bash
cat << EOF > /etc/yum.repos.d/grub2_patch.repo
[Grub_patch]
name=Grub_patch
baseurl=https://yum.rumyantsev.com/centos/7/x86_64/
enabled=1
EOF

cat << EOF > /etc/yum.conf
[main]
cachedir=/var/cache/yum/$basearch/$releasever
keepcache=0
debuglevel=2
logfile=/var/log/yum.log
exactarch=1
obsoletes=1
gpgcheck=1
plugins=1
installonly_limit=5
bugtracker_url=http://bugs.centos.org/set_project.php?project_id=23&ref=http://bugs.centos.org/bug_report_page.php?category=yum
distroverpkg=centos-release
sslverify=0
EOF

yum update
```


Install patch grub2

```bash
yum install grub2 -y --nogpgcheck
```

FSTAB

```bash
cat << EOF > /etc/fstab
/dev/mapper/otus_root-root  /  xfs     defaults        0 0
EOF
```

Change /etc/default/grub

```bash
cat << EOF > /etc/default/grub
GRUB_TIMEOUT=1
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="no_timer_check rd.lvm.lv=otus_root/root console=tty0 console=ttyS0,115200n8 net.ifnames=0 biosdevname=0 elevator=noop crashkernel=auto"
GRUB_DISABLE_RECOVERY="true"
EOF
```

Reconfigure grub conf and dracut

```bash
grub2-mkconfig -o /boot/grub2/grub.cfg
dracut -f -v /boot/initramfs-3.10.0-1127.el7.x86_64.img
```

Install Grub

```bash
grub2-install /dev/sdb
```

Disable Selinux

```bash

cat << EOF > /etc/selinux/config
SELINUX=disabled
EOF
```

reboot


I disable /dev/sda

```bash
[root@lvm vagrant]# lsblk
NAME               MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sda                  8:0    0   2G  0 disk
`-sda1               8:1    0   2G  0 part
  `-otus_root-root 253:0    0   2G  0 lvm  /
sdb                  8:16   0   2G  0 disk
```




