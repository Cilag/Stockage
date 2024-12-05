# Rendu de TP 1 Stockage Guillaume OZOUX

## Sommaire
1. [Fundamentals](#Fundamentals)
2. [Installation](#installation)
3. [Configuration](#configuration)
4. [Utilisation](#utilisation)
5. [Dépannage](#dépannage)
6. [FAQ](#faq)
7. [Conclusion](#conclusion)

---

## Fundamentals
### Lister tous les périphériques de stockage branchés à la VM

- uniquement les périphériques de stockage
```
[guigui@localhost ~]$ lsblk -d -o NAME,TYPE,SIZE,MODEL
NAME TYPE  SIZE MODEL
sda  disk   50G VBOX HARDDISK
sdb  disk   10G VBOX HARDDISK
sdc  disk   10G VBOX HARDDISK
sdd  disk   10G VBOX HARDDISK
sde  disk   10G VBOX HARDDISK
sdf  disk   10G VBOX HARDDISK
sdg  disk   10G VBOX HARDDISK
sr0  rom  1024M VBOX CD-ROM
```
###  Lister toutes les partitions des périphériques de stockage

- on doit voir les partitions physiques et logiques

```
[guigui@localhost ~]$ lsblk -o NAME,TYPE,SIZE,FSTYPE,MOUNTPOINT
NAME        TYPE  SIZE FSTYPE      MOUNTPOINT
sda         disk   50G
├─sda1      part    1G xfs         /boot
└─sda2      part   49G LVM2_member
  ├─rl-root lvm  46.9G xfs         /
  └─rl-swap lvm     2G swap        [SWAP]
sdb         disk   10G
sdc         disk   10G
sdd         disk   10G
sde         disk   10G
sdf         disk   10G
sdg         disk   10G
sr0         rom  1024M
```

###  Effectuer un test SMART sur le disque

- avec smartctl
```
[guigui@localhost ~]$ sudo smartctl -i /dev/sda
[sudo] password for guigui:
smartctl 7.2 2020-12-30 r5155 [x86_64-linux-5.14.0-503.14.1.el9_5.x86_64] (local build)
Copyright (C) 2002-20, Bruce Allen, Christian Franke, www.smartmontools.org

=== START OF INFORMATION SECTION ===
Device Model:     VBOX HARDDISK
Serial Number:    VB61f8bd6a-17f15312
Firmware Version: 1.0
User Capacity:    53,687,091,200 bytes [53.6 GB]
Sector Size:      512 bytes logical/physical
Device is:        Not in smartctl database [for details use: -P showall]
ATA Version is:   ATA/ATAPI-6 published, ANSI INCITS 361-2002
Local Time is:    Wed Dec  4 22:01:30 2024 CET
SMART support is: Unavailable - device lacks SMART capability.

[guigui@localhost ~]$ sudo smartctl -t short /dev/sda
smartctl 7.2 2020-12-30 r5155 [x86_64-linux-5.14.0-503.14.1.el9_5.x86_64] (local build)
Copyright (C) 2002-20, Bruce Allen, Christian Franke, www.smartmontools.org

SMART support is: Unavailable - device lacks SMART capability.
A mandatory SMART command failed: exiting. To continue, add one or more '-T permissive' options.

```
- souvent SMART est pas dispo avec les disques virtuels des VMs, mais sait-on jamais

### Espace disque...

- afficher l'espace disque restant sur la partition /

```
[guigui@localhost ~]$ df -h /
Filesystem           Size  Used Avail Use% Mounted on
/dev/mapper/rl-root   47G  1.7G   46G   4% /
```


### Inodes

- afficher la quantité d'inodes restants
- afficher la quantité d'inodes utilisés

```
[guigui@localhost ~]$ df -i /
Filesystem            Inodes IUsed    IFree IUse% Mounted on
/dev/mapper/rl-root 24614912 34295 24580617    1% /
```

### Latence disque

- utilisez ioping pour déterminer la latence disque
```
[guigui@localhost ~]$ sudo ioping -c 10 /
4 KiB <<< / (xfs /dev/dm-0 46.9 GiB): request=1 time=847.0 us (warmup)
4 KiB <<< / (xfs /dev/dm-0 46.9 GiB): request=2 time=2.81 ms
4 KiB <<< / (xfs /dev/dm-0 46.9 GiB): request=3 time=854.6 us
4 KiB <<< / (xfs /dev/dm-0 46.9 GiB): request=4 time=2.84 ms
4 KiB <<< / (xfs /dev/dm-0 46.9 GiB): request=5 time=2.82 ms
4 KiB <<< / (xfs /dev/dm-0 46.9 GiB): request=6 time=2.72 ms
4 KiB <<< / (xfs /dev/dm-0 46.9 GiB): request=7 time=3.72 ms
4 KiB <<< / (xfs /dev/dm-0 46.9 GiB): request=8 time=2.41 ms
4 KiB <<< / (xfs /dev/dm-0 46.9 GiB): request=9 time=4.03 ms
4 KiB <<< / (xfs /dev/dm-0 46.9 GiB): request=10 time=3.36 ms

--- / (xfs /dev/dm-0 46.9 GiB) ioping statistics ---
9 requests completed in 25.6 ms, 36 KiB read, 351 iops, 1.37 MiB/s
generated 10 requests in 9.01 s, 40 KiB, 1 iops, 4.44 KiB/s
min/avg/max/mdev = 854.6 us / 2.84 ms / 4.03 ms / 856.8 us
```

### Déterminer la taille du cache filesystem

- le filesystem, dès qu'il obtient une donnée du disque, il va cacher l'information
- le cache du filesystem est stocké en RAM
```
[guigui@localhost ~]$ grep -i 'cached' /proc/meminfo
Cached:           519252 kB
SwapCached:            0 kB
```

## Partitioning

### Ajouter sdb comme Physical Volume LVM

```
[guigui@localhost ~]$ sudo pvcreate /dev/sdb
[sudo] password for guigui:
  Physical volume "/dev/sdb" successfully created.
```
### Créer un Volume Group LVM nommé storage

```
[guigui@localhost ~]$ sudo vgcreate storage /dev/sdb
  Volume group "storage" successfully created
```
### Créer un Logical Volume LVM
- dans le VG storage
- nommé smol_data
- sa taille : 2G

```
[guigui@localhost ~]$ sudo lvcreate -L 2G -n smol_data storage
  Logical volume "smol_data" created.
```

### Créer un deuxième Logical Volume LVM

- dans le VG storage
- nommé big_data
- sa taille : tout le reste du VG

```
[guigui@localhost ~]$ sudo lvcreate -l 100%FREE -n big_data storage
  Logical volume "big_data" created.
```

```
[guigui@localhost ~]$ sudo pvs
  PV         VG      Fmt  Attr PSize   PFree
  /dev/sda2  rl      lvm2 a--  <49.00g    0
  /dev/sdb   storage lvm2 a--  <10.00g    0
[guigui@localhost ~]$ sudo vgs
  VG      #PV #LV #SN Attr   VSize   VFree
  rl        1   2   0 wz--n- <49.00g    0
  storage   1   2   0 wz--n- <10.00g    0
[guigui@localhost ~]$ sudo lvs
  LV        VG      Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  root      rl      -wi-ao---- <46.95g
  swap      rl      -wi-ao----  <2.05g
  big_data  storage -wi-a-----  <8.00g
  smol_data storage -wi-a-----   2.00g
```

### Créez un système de fichiers sur les deux LVs
- sur smol_data et big_data
- utilisez ext4
```
[guigui@localhost ~]$ sudo mkfs.ext4 /dev/storage/smol_data
mke2fs 1.46.5 (30-Dec-2021)
Creating filesystem with 524288 4k blocks and 131072 inodes
Filesystem UUID: b81f937a-ba37-4355-8976-15f17249d76c
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done

[guigui@localhost ~]$ sudo mkfs.ext4 /dev/storage/big_data
mke2fs 1.46.5 (30-Dec-2021)
Creating filesystem with 2096128 4k blocks and 524288 inodes
Filesystem UUID: 13da2a6a-88c8-47dd-a97d-85181ecaca10
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done
```
### Montez la partition

- sur le point de montage /mnt/lvm_storage
- il faudra le créer avant
```
[guigui@localhost ~]$ sudo mkdir -p /mnt/lvm_storage
```
### Configurer un automount

pour que la partition soit automatiquement montée quand elle est sollicitée
c'est systemd qui gère ça
```
[guigui@localhost ~]$ sudo nano /etc/systemd/system/mnt-lvm_storage.mount
```

```
[Unit]
Description=Mount LVM Storage

[Mount]
What=/dev/storage/smol_data
Where=/mnt/lvm_storage
Type=ext4
Options=defaults

[Install]
WantedBy=multi-user.target

```

```
[guigui@localhost ~]$ sudo mkdir -p /mnt/lvm_storage
```

```
[Unit]
Description=Automount LVM Storage

[Automount]
Where=/mnt/lvm_storage

[Install]
WantedBy=multi-user.target
TimeoutIdleSec=60
[Automount]
Where=/mnt/lvm_storage
TimeoutIdleSec=120


[guigui@localhost ~]$ sudo systemctl daemon-reload
[guigui@localhost ~]$ sudo systemctl enable mnt-lvm_storage.automount
[guigui@localhost ~]$ sudo systemctl enable mnt-lvm_storage.mount
[guigui@localhost ~]$ sudo systemctl start mnt-lvm_storage.automount

```
### Prouvez que l'automount est effectif

- affiche les partitions : elle ne doit pas être montée
```
[guigui@localhost ~]$ mount | grep lvm_storage
systemd-1 on /mnt/lvm_storage type autofs (rw,relatime,fd=41,pgrp=1,timeout=60,minproto=5,maxproto=5,direct,pipe_ino=52361)

```
- sollicite la partition
```
[guigui@localhost ~]$ ls /mnt/lvm_storage
lost+found
```
- affiche les partitions : elle doit être montée
```
[guigui@localhost ~]$ mount | grep /mnt/lvm_storage
systemd-1 on /mnt/lvm_storage type autofs (rw,relatime,fd=41,pgrp=1,timeout=60,minproto=5,maxproto=5,direct,pipe_ino=31514)
/dev/mapper/storage-smol_data on /mnt/lvm_storage type ext4 (rw,relatime,seclabel)
```
- attendre le temps du timeout
```
[guigui@localhost ~]$ mount | grep /mnt/lvm_storage
systemd-1 on /mnt/lvm_storage type autofs (rw,relatime,fd=52,pgrp=1,timeout=60,minproto=5,maxproto=5,direct,pipe_ino=33575)
```
- la partition doit être démontée
```
[guigui@storage ~]$ mount | grep /mnt/lvm_storage
systemd-1 on /mnt/lvm_storage type autofs (rw,relatime,fd=52,pgrp=1,timeout=60,minproto=5,maxproto=5,direct,pipe_ino=33575)
```

## RAID
### 1. Simple RAID

#### Prouvez que le RAID5 est en place, on doit voir :

- l'état du RAID (est-ce qu'il est en bonne santé ou pas)
- le nombre de disques dans le RAID
- la quantité d'espacce que propose le RAID

```
[guigui@localhost ~]$ sudo mdadm --detail /dev/md0
[sudo] password for guigui:
mdadm: Value "localhost.localdomain:0" cannot be set as name. Reason: Not POSIX compatible. Value ignored.
/dev/md0:
           Version : 1.2
     Creation Time : Thu Nov 14 15:14:48 2024
        Raid Level : raid5
        Array Size : 20953088 (19.98 GiB 21.46 GB)
     Used Dev Size : 10476544 (9.99 GiB 10.73 GB)
      Raid Devices : 3
     Total Devices : 3
       Persistence : Superblock is persistent

       Update Time : Thu Nov 14 16:42:30 2024
             State : clean
    Active Devices : 3
   Working Devices : 3
    Failed Devices : 0
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : localhost.localdomain:0  (local to host localhost.localdomain)
              UUID : 9b1c9bbd:6ec32518:edaea7c5:7b086dfd
            Events : 18

    Number   Major   Minor   RaidDevice State
       0       8       32        0      active sync   /dev/sdc
       1       8       48        1      active sync   /dev/sdd
       3       8       64        2      active sync   /dev/sde
```

#### Rendre la configuration automatique au boot de la machine

```
[guigui@storage ~]$ cat /etc/mdadm.conf
ARRAY /dev/md0 metadata=1.2 UUID=e78b538e:fdac9387:be0c6d79:c6766aa9
```
#### Créez un système de fichiers sur la partition proposé par le RAID

```
[guigui@storage ~]$ sudo mkfs.ext4 /dev/md0
mke2fs 1.46.5 (30-Dec-2021)
/dev/md0 contains a ext4 file system
        last mounted on Wed Dec  4 23:32:55 2024
Proceed anyway? (y,N) y
Creating filesystem with 5238272 4k blocks and 1310720 inodes
Filesystem UUID: 0887eaa8-48d6-4fbf-944e-c57efb681d6f
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
        4096000

Allocating group tables: done
Writing inode tables: done
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done
```

#### Monter la partition sur /mnt/raid_storage
```
[guigui@storage ~]$ sudo mkdir -p /mnt/raid_storage
[guigui@storage ~]$ sudo mount /dev/md0 /mnt/raid_storage
```

#### Prouvez que...

- la partition est bien montée
- il y a bien l'espace disponible attendue sur la partition
- vous pouvez lire et écrire sur la partition
```
[guigui@storage ~]$ df -h /mnt/raid_storage
Filesystem      Size  Used Avail Use% Mounted on
/dev/md0         20G   24K   19G   1% /mnt/raid_storage
[guigui@storage ~]$ echo "RAID 5 test successful!" | sudo tee /mnt/raid_storage/testfile.txt
RAID 5 test successful!
[guigui@storage ~]$ cat /mnt/raid_storage/testfile.txt
RAID 5 test successful!
```

#### Mini benchmark

- faites un test de vitesse d'écriture sur la partition mdadm montée : /mnt/raid_storage
```
[guigui@storage ~]$ fio --name=write-test --filename=/mnt/raid_storage/testfile --size=1G --bs=4k --rw=write --direct=1 --numjobs=1 --iodepth=1 --group_reporting
write-test: (g=0): rw=write, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=psync, iodepth=1
fio-3.35
Starting 1 process
write-test: Laying out IO file (1 file / 1024MiB)
Jobs: 1 (f=1): [W(1)][100.0%][w=4600KiB/s][w=1150 IOPS][eta 00m:00s]
write-test: (groupid=0, jobs=1): err= 0: pid=4782: Thu Dec  5 00:37:54 2024
  write: IOPS=1176, BW=4706KiB/s (4819kB/s)(1024MiB/222823msec); 0 zone resets
    clat (usec): min=372, max=65298, avg=848.72, stdev=651.91
     lat (usec): min=372, max=65298, avg=848.89, stdev=651.95
    clat percentiles (usec):
     |  1.00th=[  383],  5.00th=[  392], 10.00th=[  404], 20.00th=[  478],
     | 30.00th=[  515], 40.00th=[  627], 50.00th=[  742], 60.00th=[  832],
     | 70.00th=[  947], 80.00th=[ 1156], 90.00th=[ 1434], 95.00th=[ 1582],
     | 99.00th=[ 2180], 99.50th=[ 2737], 99.90th=[10814], 99.95th=[11994],
     | 99.99th=[14353]
   bw (  KiB/s): min= 3416, max= 5592, per=100.00%, avg=4709.77, stdev=354.16, samples=444
   iops        : min=  854, max= 1398, avg=1177.23, stdev=88.55, samples=444
  lat (usec)   : 500=27.37%, 750=23.69%, 1000=23.73%
  lat (msec)   : 2=23.75%, 4=1.22%, 10=0.11%, 20=0.14%, 50=0.01%
  lat (msec)   : 100=0.01%
  cpu          : usr=0.05%, sys=1.01%, ctx=262202, majf=0, minf=11
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,262144,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=4706KiB/s (4819kB/s), 4706KiB/s-4706KiB/s (4819kB/s-4819kB/s), io=1024MiB (1074MB), run=222823-222823msec

Disk stats (read/write):
    md0: ios=0/262146, merge=0/0, ticks=0/223118, in_queue=223118, util=100.00%, aggrios=43721/174869, aggrmerge=36/72, aggrticks=19950/58945, aggrin_queue=79242, aggrutil=27.21%
  sdd: ios=43650/174984, merge=0/109, ticks=19827/59346, in_queue=79555, util=24.11%
  sde: ios=43729/174777, merge=81/28, ticks=19867/70748, in_queue=90917, util=27.21%
  sdc: ios=43786/174848, merge=28/81, ticks=20158/46741, in_queue=67256, util=21.02%
```
- idem sur la partition créée avec LVM à la partie précédente : /mnt/lvm_storage
```
[guigui@storage ~]$ fio --name=write-test --filename=/mnt/lvm_storage/testfile --size=1G --bs=4k --rw=write --direct=1 --numjobs=1 --iodepth=1 --group_reporting
write-test: (g=0): rw=write, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=psync, iodepth=1
fio-3.35
Starting 1 process
write-test: Laying out IO file (1 file / 1024MiB)
Jobs: 1 (f=1): [W(1)][100.0%][w=11.3MiB/s][w=2884 IOPS][eta 00m:00s]
write-test: (groupid=0, jobs=1): err= 0: pid=4765: Thu Dec  5 00:34:06 2024
  write: IOPS=2778, BW=10.9MiB/s (11.4MB/s)(1024MiB/94355msec); 0 zone resets
    clat (usec): min=249, max=22318, avg=359.09, stdev=151.60
     lat (usec): min=249, max=22318, avg=359.21, stdev=151.66
    clat percentiles (usec):
     |  1.00th=[  258],  5.00th=[  277], 10.00th=[  285], 20.00th=[  310],
     | 30.00th=[  318], 40.00th=[  330], 50.00th=[  338], 60.00th=[  347],
     | 70.00th=[  359], 80.00th=[  375], 90.00th=[  424], 95.00th=[  498],
     | 99.00th=[  922], 99.50th=[ 1074], 99.90th=[ 1549], 99.95th=[ 1745],
     | 99.99th=[ 3064]
   bw (  KiB/s): min= 8720, max=11880, per=100.00%, avg=11123.06, stdev=737.24, samples=188
   iops        : min= 2180, max= 2970, avg=2780.65, stdev=184.34, samples=188
  lat (usec)   : 250=0.01%, 500=95.12%, 750=3.05%, 1000=1.17%
  lat (msec)   : 2=0.63%, 4=0.02%, 10=0.01%, 20=0.01%, 50=0.01%
  cpu          : usr=0.00%, sys=6.17%, ctx=230919, majf=0, minf=12
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,262144,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=10.9MiB/s (11.4MB/s), 10.9MiB/s-10.9MiB/s (11.4MB/s-11.4MB/s), io=1024MiB (1074MB), run=94355-94355msec

Disk stats (read/write):
    dm-2: ios=0/261643, merge=0/0, ticks=0/38551, in_queue=38551, util=40.75%, aggrios=0/262202, aggrmerge=0/58, aggrticks=0/60602, aggrin_queue=60815, aggrutil=40.73%
  sdb: ios=0/262202, merge=0/58, ticks=0/60602, in_queue=60815, util=40.73%
```

- Bon on est en environnement virtuel VirtualBox, avec des PC portables et des OS pas toujours stables, alors à prendre avec des pincettes le résultat.


#### Simule une panne
- débranche un disque du RAID !
```
[guigui@storage ~]$ sudo mdadm --fail /dev/md0 /dev/sdd
[sudo] password for guigui:
mdadm: set /dev/sdd faulty in /dev/md0
```
#### Montre l'état du RAID dégradé

- avec une commande, on doit voir que le RAID est dégradé
```
[guigui@storage ~]$ cat /proc/mdstat
Personalities : [raid6] [raid5] [raid4]
md0 : active raid5 sde[3] sdd[1](F) sdc[0]
      20953088 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/2] [U_U]

unused devices: <none>
```
#### Remonte le disque dur

- il faut retrouver un RAID5 fonctionnel
```
[guigui@storage ~]$ sudo mdadm --add /dev/md0 /dev/sdd
mdadm: added /dev/sdd
```
- avec un cat /proc/mdstat on peut voir si une recovery est en cours
```
[guigui@storage ~]$ cat /proc/mdstat
Personalities : [raid6] [raid5] [raid4]
md0 : active raid5 sdd[4] sde[3] sdc[0]
      20953088 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/2] [U_U]
      [=====>...............]  recovery = 28.4% (2981564/10476544) finish=0.6min speed=186347K/sec

unused devices: <none>
```
- prouvez que le RAID est de nouveau fonctionnel
```
[guigui@storage ~]$ cat /proc/mdstat
Personalities : [raid6] [raid5] [raid4]
md0 : active raid5 sdd[4] sde[3] sdc[0]
      20953088 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/2] [U_U]
      [=====>...............]  recovery = 28.4% (2981564/10476544) finish=0.6min speed=186347K/sec

unused devices: <none>
[guigui@storage ~]$ cat /proc/mdstat
Personalities : [raid6] [raid5] [raid4]
md0 : active raid5 sdd[4] sde[3] sdc[0]
      20953088 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/3] [UUU]

unused devices: <none>
[guigui@storage ~]$ sudo mdadm --detail /dev/md0
/dev/md0:
           Version : 1.2
     Creation Time : Thu Dec  5 00:15:47 2024
        Raid Level : raid5
        Array Size : 20953088 (19.98 GiB 21.46 GB)
     Used Dev Size : 10476544 (9.99 GiB 10.73 GB)
      Raid Devices : 3
     Total Devices : 3
       Persistence : Superblock is persistent

       Update Time : Thu Dec  5 00:43:46 2024
             State : clean
    Active Devices : 3
   Working Devices : 3
    Failed Devices : 0
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : storage.b3:0  (local to host storage.b3)
              UUID : b5dd2d6c:607cbdd8:7fc4e509:63461865
            Events : 42

    Number   Major   Minor   RaidDevice State
       0       8       32        0      active sync   /dev/sdc
       4       8       48        1      active sync   /dev/sdd
       3       8       64        2      active sync   /dev/sde
```
#### Ajoutez un disque encore inutilisé au RAID5 comme disque de spare
- prouvez avec une commande que le RAID5 comporte désormais un disque en spare
- on en est à sdf si je tiens bien le compte !

```
[guigui@localhost ~]$ sudo mdadm --add /dev/md0 /dev/sdf
mdadm: added /dev/sdf
[guigui@localhost ~]$ sudo mdadm --detail /dev/md0
/dev/md0:
           Version : 1.2
     Creation Time : Wed Dec  4 13:26:37 2024
        Raid Level : raid5
        Array Size : 20953088 (19.98 GiB 21.46 GB)
     Used Dev Size : 10476544 (9.99 GiB 10.73 GB)
      Raid Devices : 3
     Total Devices : 4
       Persistence : Superblock is persistent

       Update Time : Wed Dec  4 13:59:17 2024
             State : clean
    Active Devices : 3
   Working Devices : 4
    Failed Devices : 0
     Spare Devices : 1

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : localhost.localdomain:0  (local to host localhost.localdomain)
              UUID : c77ea79a:d14cc579:42c529ca:15ccbeaa
            Events : 43

    Number   Major   Minor   RaidDevice State
       4       8       32        0      active sync   /dev/sdc
       1       8       48        1      active sync   /dev/sdd
       3       8       64        2      active sync   /dev/sde

```
#### Simuler une panne
- débrancher un disque du RAID
```
[guigui@storage ~]$ sudo mdadm --fail /dev/md0 /dev/sdd
mdadm: set /dev/sdd faulty in /dev/md0
```
- observer le comportement, la recovery
```
[guigui@storage ~]$ cat /proc/mdstat
Personalities : [raid6] [raid5] [raid4]
md0 : active raid5 sdf[5] sdd[4](F) sde[3] sdc[0]
      20953088 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/2] [U_U]
      [=>...................]  recovery =  9.5% (1005696/10476544) finish=0.7min speed=201139K/sec

unused devices: <none>
```
- déterminer le temps entre le début de la panne, et le RAID de nouveau complètement opérationnel
```
2 minutes
```
#### Remonter le disque débranché
- débrouillez-vous, on veut retrouver l'état initial : RAID5 avec un quatrième disque en spare
```
sudo mdadm --add /dev/md0 /dev/sdg
sudo mdadm --detail /dev/md0

```
### 4. Grow
#### Ajoutez un disque encore inutilisé au RAID5 comme disque de spare

- should be sdg
```
[guigui@storage ~]$ sudo mdadm --add /dev/md0 /dev/sdg
mdadm: added /dev/sdg
```
- prouvez que vous avez bien un nouveau disque en spare
```
[guigui@storage ~]$ sudo mdadm --detail /dev/md0
/dev/md0:
           Version : 1.2
     Creation Time : Thu Dec  5 00:15:47 2024
        Raid Level : raid5
        Array Size : 20953088 (19.98 GiB 21.46 GB)
     Used Dev Size : 10476544 (9.99 GiB 10.73 GB)
      Raid Devices : 3
     Total Devices : 5
       Persistence : Superblock is persistent

       Update Time : Thu Dec  5 01:04:25 2024
             State : clean
    Active Devices : 3
   Working Devices : 5
    Failed Devices : 0
     Spare Devices : 2

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : storage.b3:0  (local to host storage.b3)
              UUID : b5dd2d6c:607cbdd8:7fc4e509:63461865
            Events : 88

    Number   Major   Minor   RaidDevice State
       0       8       32        0      active sync   /dev/sdc
       4       8       48        1      active sync   /dev/sdd
       3       8       64        2      active sync   /dev/sde

       5       8       80        -      spare   /dev/sdf
       6       8       96        -      spare   /dev/sdg
```
- RAID5 avec 2 disques spare pour le moment !

#### Grow !

agrandissez le RAID5 pour utiliser désormais 4 disques actifs dans le RAID
on garde donc un seul spare
```
[guigui@storage ~]$ sudo mdadm --grow /dev/md0 --raid-devices=4
[sudo] password for guigui:
[guigui@storage ~]$ cat /proc/mdstat
Personalities : [raid6] [raid5] [raid4]
md0 : active raid5 sdf[5] sdd[4] sdg[6](S) sde[3] sdc[0]
      20953088 blocks super 1.2 level 5, 512k chunk, algorithm 2 [4/4] [UUUU]
      [>....................]  reshape =  1.4% (153612/10476544) finish=3.3min speed=51204K/sec

unused devices: <none>
```

#### Prouvez que le RAID5 propose désormais 4 disques actifs

l'espace proposé devrait aussi avoir grandi
alors combien d'espace sur un RAID5 de 4 disques ?
```
[guigui@storage ~]$ df -h /mnt/raid_storage
Filesystem      Size  Used Avail Use% Mounted on
/dev/md0         20G  1.1G   18G   6% /mnt/raid_storage
[guigui@storage ~]$ sudo resize2fs /dev/md0
resize2fs 1.46.5 (30-Dec-2021)
Filesystem at /dev/md0 is mounted on /mnt/raid_storage; on-line resizing required
old_desc_blocks = 3, new_desc_blocks = 4
The filesystem on /dev/md0 is now 7857408 (4k) blocks long.

[guigui@storage ~]$ df -h /mnt/raid_storage
Filesystem      Size  Used Avail Use% Mounted on
/dev/md0         30G  1.1G   27G   4% /mnt/raid_storage
```
#### Euuuh wait a sec... /mnt/raid_storage ???

- la partition montée sur /mnt/raid_storage fait toujours l'ancienne taille
```
[guigui@storage ~]$ df -h /mnt/raid_storage
Filesystem      Size  Used Avail Use% Mounted on
/dev/md0        20G   5G   15G   25% /mnt/raid_storage
```
- prouvez que la partition /mnt/raid_storage n'a pas changé de taille
```
[guigui@storage ~]$ sudo mdadm --detail /dev/md0
/dev/md0:
           Version : 1.2
     Creation Time : Thu Dec  5 00:15:47 2024
        Raid Level : raid5
        Array Size : 31429632 (29.97 GiB 32.18 GB)
     Used Dev Size : 10476544 (9.99 GiB 10.73 GB)
      Raid Devices : 4
     Total Devices : 5
       Persistence : Superblock is persistent

       Update Time : Thu Dec  5 01:17:48 2024
             State : clean
    Active Devices : 4
   Working Devices : 5
    Failed Devices : 0
     Spare Devices : 1

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : storage.b3:0  (local to host storage.b3)
              UUID : b5dd2d6c:607cbdd8:7fc4e509:63461865
            Events : 134

    Number   Major   Minor   RaidDevice State
       0       8       32        0      active sync   /dev/sdc
       4       8       48        1      active sync   /dev/sdd
       3       8       64        2      active sync   /dev/sde
       5       8       80        3      active sync   /dev/sdf

       6       8       96        -      spare   /dev/sdg
```
- il faut indiquer au filesystem ext4 qu'il doit rescanner la partition pour s'adapter à la nouvelle taille
```
[guigui@storage ~]$ mount | grep /mnt/raid_storage
[guigui@storage ~]$ sudo resize2fs /dev/md0
```
- prouvez que la partition /mnt/raid_storage a bien la nouvelle taille après votre opération
```
[guigui@storage ~]$ df -h /mnt/raid_storage
Filesystem      Size  Used Avail Use% Mounted on
/dev/md0         30G  1.1G   27G   4% /mnt/raid_storage
```

## NFS
#### Installer un serveur NFS
- il devra exposer les deux points de montage créés des parties précédentes : /mnt/raid_storage et /mnt/lvm_storage
- seul le réseau local du serveur NFS doit pouvoir y accéder
- il sera nécessaire de configurer le firewall de la machine
```
[guigui@storage ~]$ sudo systemctl enable --now nfs-server
Created symlink /etc/systemd/system/multi-user.target.wants/nfs-server.service → /usr/lib/systemd/system/nfs-server.service.
[guigui@storage ~]$ sudo nano /etc/exports
[guigui@storage ~]$ sudo exportfs -ra
[guigui@storage ~]$ sudo exportfs -v
/mnt/raid_storage
                192.168.56.0/24(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)
/mnt/lvm_storage
                192.168.56.0/24(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)
[guigui@storage ~]$ sudo firewall-cmd --permanent --add-service=nfs
sudo firewall-cmd --permanent --add-service=mountd
sudo firewall-cmd --permanent --add-service=rpc-bind
sudo firewall-cmd --reload
```
#### Pop une deuxième VM en vif

- installer un client NFS pour se connecter au serveur
- pour /mnt/raid_storage
  monter le dossier partagé /mnt/raid_storage du serveur
  sur le point de montage que vous aurez créé sur le client /mnt/raid_storage
  (même chemin aussi sur le client)
- et idem : partage /mnt/lvm_storage du serveur monté sur /mnt/lvm_storage côté client
```
[guigui2@localhost ~]$ sudo dnf install -y nfs-utils
[guigui2@localhost ~]$ sudo mkdir -p /mnt/raid_storage
sudo mkdir -p /mnt/lvm_storage
[guigui2@localhost ~]$ sudo mount -t nfs 192.168.56.106:/mnt/raid_storage /mnt/raid_storage
[guigui2@localhost ~]$ sudo mount -t nfs 192.168.56.106:/mnt/lvm_storage /mnt/lvm_storage
[guigui2@localhost ~]$ df -h
Filesystem                        Size  Used Avail Use% Mounted on
devtmpfs                          4.0M     0  4.0M   0% /dev
tmpfs                             888M     0  888M   0% /dev/shm
tmpfs                             356M  5.0M  351M   2% /run
/dev/mapper/rl-root                17G  1.3G   16G   8% /
/dev/sda1                         960M  225M  736M  24% /boot
tmpfs                             178M     0  178M   0% /run/user/1000
192.168.56.106:/mnt/raid_storage   30G  1.0G   27G   4% /mnt/raid_storage
192.168.56.106:/mnt/lvm_storage   2.0G  1.0G  804M  57% /mnt/lvm_storage
```
#### Benchmarkz

- faites un test de vitesse d'écriture sur la partition mdadm montée en NFS : /mnt/raid_storage

- faites un test de vitesse d'écriture sur la partition LVM montée en NFS : /mnt/lvm_storage
```
[guigui2@localhost ~]$ sync && dd if=/dev/zero of=/mnt/raid_storage/testfile bs=1G count=1 oflag=direct
1+0 records in
1+0 records out
1073741824 bytes (1.1 GB, 1.0 GiB) copied, 14.1014 s, 76.1 MB/s
[guigui2@localhost ~]$ sync && dd if=/dev/zero of=/mnt/lvm_storage/testfile bs=1G count=1 oflag=direct
1+0 records in
1+0 records out
1073741824 bytes (1.1 GB, 1.0 GiB) copied, 8.96163 s, 120 MB/s
[guigui2@localhost ~]$ ls /mnt/raid_storage
ls /mnt/lvm_storage
lost+found  testfile  testfile.txt
lost+found  testfile
```