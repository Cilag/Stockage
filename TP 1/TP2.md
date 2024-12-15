
# II. SAN network

L'objectif est le suivant :

- **les machines de stockage**
  - ont plein plein de disques
  - ont du RAID configuré
  - propose les volumes RAID à travers le réseau comme *target iSCSI*
- **les machines "chunks"**
  - accèdent aux disque des machines de stockage
  - grâce à iSCSI
  - mise en place d'un *multipathing iSCSI*

## 1. Storage machines

> **Toutes les manips de cettes section sont à réaliser sur `sto1` et `sto2`.**

### A. Disks and RAID

➜ **Ajouter des disques aux machines `sto1` et `sto2`**

🌞 **Configurer des RAID**

- à la fin, on veut 3 volumes RAID sur `sto1` et 3 volumes RAID sur `sto2`
- vous pouvez faire le setup simple indiqué avant ou vous faire un peu plais et partir sur du RAID5 par exemple
- **il faut 3 volumes RAID prêts à l'emploi en fin de setup**
- vous utiliserez MDADM pour mettre en place les RAID
```
[guigui@sto1 ~]$ df -h
Filesystem                Size  Used Avail Use% Mounted on
devtmpfs                  4.0M     0  4.0M   0% /dev
tmpfs                     888M     0  888M   0% /dev/shm
tmpfs                     355M  5.1M  350M   2% /run
/dev/mapper/rl_vbox-root   17G  1.6G   16G   9% /
/dev/sda1                 960M  314M  647M  33% /boot
tmpfs                     178M     0  178M   0% /run/user/1000
/dev/md0                  988M   24K  921M   1% /mnt/raid0
/dev/md1                  988M   24K  921M   1% /mnt/raid1
/dev/md2                  988M   24K  921M   1% /mnt/raid2
```

```
[guigui@sto2 ~]$  df -h
Filesystem                Size  Used Avail Use% Mounted on
devtmpfs                  4.0M     0  4.0M   0% /dev
tmpfs                     888M     0  888M   0% /dev/shm
tmpfs                     355M  5.1M  350M   2% /run
/dev/mapper/rl_vbox-root   17G  1.6G   16G   9% /
/dev/sda1                 960M  314M  647M  33% /boot
tmpfs                     178M     0  178M   0% /run/user/1000
/dev/md0                  988M   24K  921M   1% /mnt/raid0
/dev/md1                  988M   24K  921M   1% /mnt/raid1
/dev/md2                  988M   24K  921M   1% /mnt/raid2
```
🌞 **Prouvez que vous avez 3 volumes RAID prêts à l'emploi**

- avec une commande `lsblk`
```
[guigui@sto1 ~]$ lsblk
NAME             MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sda                8:0    0   20G  0 disk
├─sda1             8:1    0    1G  0 part  /boot
└─sda2             8:2    0   19G  0 part
  ├─rl_vbox-root 253:0    0   17G  0 lvm   /
  └─rl_vbox-swap 253:1    0    2G  0 lvm   [SWAP]
sdb                8:16   0    1G  0 disk
└─md0              9:0    0 1022M  0 raid1 /mnt/raid0
sdc                8:32   0    1G  0 disk
└─md0              9:0    0 1022M  0 raid1 /mnt/raid0
sdd                8:48   0    1G  0 disk
└─md1              9:1    0 1022M  0 raid1 /mnt/raid1
sde                8:64   0    1G  0 disk
└─md2              9:2    0 1022M  0 raid1 /mnt/raid2
sdf                8:80   0    1G  0 disk
└─md1              9:1    0 1022M  0 raid1 /mnt/raid1
sdg                8:96   0    1G  0 disk
└─md2              9:2    0 1022M  0 raid1 /mnt/rai
```

```
[guigui@sto2 ~]$ lsblk
NAME             MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sda                8:0    0   20G  0 disk
├─sda1             8:1    0    1G  0 part  /boot
└─sda2             8:2    0   19G  0 part
  ├─rl_vbox-root 253:0    0   17G  0 lvm   /
  └─rl_vbox-swap 253:1    0    2G  0 lvm   [SWAP]
sdb                8:16   0    1G  0 disk
└─md0              9:0    0 1022M  0 raid1 /mnt/raid0
sdc                8:32   0    1G  0 disk
└─md0              9:0    0 1022M  0 raid1 /mnt/raid0
sdd                8:48   0    1G  0 disk
└─md1              9:1    0 1022M  0 raid1 /mnt/raid1
sde                8:64   0    1G  0 disk
└─md1              9:1    0 1022M  0 raid1 /mnt/raid1
sdf                8:80   0    1G  0 disk
└─md2              9:2    0 1022M  0 raid1 /mnt/raid2
sdg                8:96   0    1G  0 disk
└─md2              9:2    0 1022M  0 raid1 /mnt/raid2
sr0               11:0    1 1024M  0 rom
```

### B. iSCSI target

On va configurer nos deux machines `sto1` et `sto2` pour qu'elles exposent leurs volumes RAID sur le réseau, avec iSCSI.

On appelle *target iSCSI* un device exposé sur le réseau grâce au protocole iSCSI.

On appelle *iSCSI initiator* une machine qui va initier une connexion iSCSI pour accéder à un *target iSCSI*.

Sur les machines `sto1` et `sto2`, on va donc s'occuper pour le moment de les configurer pour qu'elles exposent les volumes RAID comme *target iSCSI*.

Pour ça, on va utiliser l'outil `target` dispo sous les OS Linux.

🌞 **Installer `target`**

- c'est le paquet `targetcli`

🌞 **Démarrer le service `target`**

- activez le aussi au démarrage de la machine

🌞 **Configurer les *targets iSCSI***

- il faut configurer :
  - 3 target iSCSI sur chaque machine : un pour exposer chacun des volumes RAID
  - une ACL sur chaque target (vous pouvez try de mettre une authent, mais je vous conseille de rester simple)
- chaque target doit respecter la convention de nommage pour son IQN :
  - `iqn.2024-12.tp2.b3:data-chunk1` pour le premier target iSCSI (destiné à être utilisé par la machine `chunk1.tp2.b3`)
  - `iqn.2024-12.tp2.b3:data-chunk2` pour le deuxième
  - et `iqn.2024-12.tp2.b3:data-chunk3`
- en utilisant `target-cli`
  - exemple pour créer un target

### Sto1
```
/> /backstores/fileio create name=data-chunk1 file_or_dev=/dev/md0
Note: block backstore preferred for best results
Created fileio data-chunk1 with size 1071644672
/> /iscsi create iqn.2024-12.tp2.b3:data-chunk1
Created target iqn.2024-12.tp2.b3:data-chunk1.
Created TPG 1.
Global pref auto_add_default_portal=true
Created default portal listening on all IPs (0.0.0.0), port 3260.
/> /iscsi/iqn.2024-12.tp2.b3:data-chunk1/tpg1/acls create iqn.2024-12.tp2.b3:data-chunk1:chunk1-initiator
Created Node ACL for iqn.2024-12.tp2.b3:data-chunk1:chunk1-initiator
/> /iscsi/iqn.2024-12.tp2.b3:data-chunk1/tpg1/luns/ create /backstores/fileio/data-chunk1
Created LUN 0.
Created LUN 0->0 mapping in node ACL iqn.2024-12.tp2.b3:data-chunk1:chunk1-initiator
/> /backstores/fileio create name=data-chunk2 file_or_dev=/dev/md1
Note: block backstore preferred for best results
Created fileio data-chunk2 with size 1071644672
/> /iscsi create iqn.2024-12.tp2.b3:data-chunk2
Created target iqn.2024-12.tp2.b3:data-chunk2.
Created TPG 1.
Global pref auto_add_default_portal=true
Created default portal listening on all IPs (0.0.0.0), port 3260.
/> /iscsi/iqn.2024-12.tp2.b3:data-chunk2/tpg1/acls create iqn.2024-12.tp2.b3:data-c
hunk2:chunk2-initiator
Created Node ACL for iqn.2024-12.tp2.b3:data-chunk2:chunk2-initiator
/> /iscsi/iqn.2024-12.tp2.b3:data-chunk2/tpg1/luns/ create /backstores/fileio/data-
chunk2
Created LUN 0.
Created LUN 0->0 mapping in node ACL iqn.2024-12.tp2.b3:data-chunk2:chunk2-initiator
```
### Sto2
```
/> /backstores/fileio create name=data-chunk2 file_or_dev=/dev/md1
Note: block backstore preferred for best results
Created fileio data-chunk2 with size 1071644672
/> /iscsi create iqn.2024-12.tp2.b3:data-chunk2
Created target iqn.2024-12.tp2.b3:data-chunk2.
Created TPG 1.
Global pref auto_add_default_portal=true
Created default portal listening on all IPs (0.0.0.0), port 3260.
/> /iscsi/iqn.2024-12.tp2.b3:data-chunk2/tpg1/acls create iqn.2024-12.tp2.b3:data-chunk2:chunk2-initiator
Created Node ACL for iqn.2024-12.tp2.b3:data-chunk2:chunk2-initiator
/> /iscsi/iqn.2024-12.tp2.b3:data-chunk2/tpg1/luns/ create /backstores/fileio/data-chunk2
Created LUN 0.
Created LUN 0->0 mapping in node ACL iqn.2024-12.tp2.b3:data-chunk2:chunk2-initiator
/> /backstores/fileio create name=data-chunk3 file_or_dev=/dev/md2
Note: block backstore preferred for best results
Created fileio data-chunk3 with size 1071644672
/> /iscsi create iqn.2024-12.tp2.b3:data-chunk3
Created target iqn.2024-12.tp2.b3:data-chunk3.
Created TPG 1.
Global pref auto_add_default_portal=true
Created default portal listening on all IPs (0.0.0.0), port 3260.
/> /iscsi/iqn.2024-12.tp2.b3:data-chunk3/tpg1/acls create iqn.2024-12.tp2.b3:data-chunk3:chunk3-initiator
Created Node ACL for iqn.2024-12.tp2.b3:data-chunk3:chunk3-initiator
/> /iscsi/iqn.2024-12.tp2.b3:data-chunk3/tpg1/luns/ create /backstores/fileio/data-chunk3
Created LUN 0.
Created LUN 0->0 mapping in node ACL iqn.2024-12.tp2.b3:data-chunk3:chunk3-initiator
```
➜ **Vous devriez avoir un truc comme ça une fois en place, avec les trois volumes RAID exposés comme target :**
### Sto1
```
/> ls
o- / ........................................................................................... [...]
  o- backstores ................................................................................ [...]
  | o- block .................................................................... [Storage Objects: 3]
  | | o- data-chunk1 ..................................... [/dev/md0 (1022.0MiB) write-thru activated]
  | | | o- alua ..................................................................... [ALUA Groups: 1]
  | | |   o- default_tg_pt_gp ......................................... [ALUA state: Active/optimized]
  | | o- data-chunk2 ..................................... [/dev/md1 (1022.0MiB) write-thru activated]
  | | | o- alua ..................................................................... [ALUA Groups: 1]
  | | |   o- default_tg_pt_gp ......................................... [ALUA state: Active/optimized]
  | | o- data-chunk3 ..................................... [/dev/md2 (1022.0MiB) write-thru activated]
  | |   o- alua ..................................................................... [ALUA Groups: 1]
  | |     o- default_tg_pt_gp ......................................... [ALUA state: Active/optimized]
  | o- fileio ................................................................... [Storage Objects: 0]
  | o- pscsi .................................................................... [Storage Objects: 0]
  | o- ramdisk .................................................................. [Storage Objects: 0]
  o- iscsi .............................................................................. [Targets: 3]
  | o- iqn.2024-12.tp2.b3:data-chunk1 ...................................................... [TPGs: 1]
  | | o- tpg1 ................................................................. [no-gen-acls, no-auth]
  | |   o- acls ............................................................................ [ACLs: 1]
  | |   | o- iqn.2024-12.tp2.b3:data-chunk1:chunk1-initiator ........................ [Mapped LUNs: 1]
  | |   |   o- mapped_lun0 ............................................. [lun0 block/data-chunk1 (rw)]
  | |   o- luns ............................................................................ [LUNs: 1]
  | |   | o- lun0 .................................. [block/data-chunk1 (/dev/md0) (default_tg_pt_gp)]
  | |   o- portals ...................................................................... [Portals: 1]
  | |     o- 0.0.0.0:3260 ....................................................................... [OK]
  | o- iqn.2024-12.tp2.b3:data-chunk2 ...................................................... [TPGs: 1]
  | | o- tpg1 ................................................................. [no-gen-acls, no-auth]
  | |   o- acls ............................................................................ [ACLs: 1]
  | |   | o- iqn.2024-12.tp2.b3:data-chunk2:chunk2-initiator ........................ [Mapped LUNs: 1]
  | |   |   o- mapped_lun0 ............................................. [lun0 block/data-chunk2 (rw)]
  | |   o- luns ............................................................................ [LUNs: 1]
  | |   | o- lun0 .................................. [block/data-chunk2 (/dev/md1) (default_tg_pt_gp)]
  | |   o- portals ...................................................................... [Portals: 1]
  | |     o- 0.0.0.0:3260 ....................................................................... [OK]
  | o- iqn.2024-12.tp2.b3:data-chunk3 ...................................................... [TPGs: 1]
  |   o- tpg1 ................................................................. [no-gen-acls, no-auth]
  |     o- acls ............................................................................ [ACLs: 1]
  |     | o- iqn.2024-12.tp2.b3:data-chunk3:chunk3-initiator ........................ [Mapped LUNs: 1]
  |     |   o- mapped_lun0 ............................................. [lun0 block/data-chunk3 (rw)]
  |     o- luns ............................................................................ [LUNs: 1]
  |     | o- lun0 .................................. [block/data-chunk3 (/dev/md2) (default_tg_pt_gp)]
  |     o- portals ...................................................................... [Portals: 1]
  |       o- 0.0.0.0:3260 ....................................................................... [OK]
  o- loopback ........................................................................... [Targets: 0]
```

### Sto2
```
/> ls
o- / ........................................................................................... [...]
  o- backstores ................................................................................ [...]
  | o- block .................................................................... [Storage Objects: 3]
  | | o- data-chunk1 ..................................... [/dev/md0 (1022.0MiB) write-thru activated]
  | | | o- alua ..................................................................... [ALUA Groups: 1]
  | | |   o- default_tg_pt_gp ......................................... [ALUA state: Active/optimized]
  | | o- data-chunk2 ..................................... [/dev/md1 (1022.0MiB) write-thru activated]
  | | | o- alua ..................................................................... [ALUA Groups: 1]
  | | |   o- default_tg_pt_gp ......................................... [ALUA state: Active/optimized]
  | | o- data-chunk3 ..................................... [/dev/md2 (1022.0MiB) write-thru activated]
  | |   o- alua ..................................................................... [ALUA Groups: 1]
  | |     o- default_tg_pt_gp ......................................... [ALUA state: Active/optimized]
  | o- fileio ................................................................... [Storage Objects: 0]
  | o- pscsi .................................................................... [Storage Objects: 0]
  | o- ramdisk .................................................................. [Storage Objects: 0]
  o- iscsi .............................................................................. [Targets: 3]
  | o- iqn.2024-12.tp2.b3:data-chunk1 ...................................................... [TPGs: 1]
  | | o- tpg1 ................................................................. [no-gen-acls, no-auth]
  | |   o- acls ............................................................................ [ACLs: 1]
  | |   | o- iqn.2024-12.tp2.b3:data-chunk1:chunk1-initiator ........................ [Mapped LUNs: 1]
  | |   |   o- mapped_lun0 ............................................. [lun0 block/data-chunk1 (rw)]
  | |   o- luns ............................................................................ [LUNs: 1]
  | |   | o- lun0 .................................. [block/data-chunk1 (/dev/md0) (default_tg_pt_gp)]
  | |   o- portals ...................................................................... [Portals: 1]
  | |     o- 0.0.0.0:3260 ....................................................................... [OK]
  | o- iqn.2024-12.tp2.b3:data-chunk2 ...................................................... [TPGs: 1]
  | | o- tpg1 ................................................................. [no-gen-acls, no-auth]
  | |   o- acls ............................................................................ [ACLs: 1]
  | |   | o- iqn.2024-12.tp2.b3:data-chunk2:chunk2-initiator ........................ [Mapped LUNs: 1]
  | |   |   o- mapped_lun0 ............................................. [lun0 block/data-chunk2 (rw)]
  | |   o- luns ............................................................................ [LUNs: 1]
  | |   | o- lun0 .................................. [block/data-chunk2 (/dev/md1) (default_tg_pt_gp)]
  | |   o- portals ...................................................................... [Portals: 1]
  | |     o- 0.0.0.0:3260 ....................................................................... [OK]
  | o- iqn.2024-12.tp2.b3:data-chunk3 ...................................................... [TPGs: 1]
  |   o- tpg1 ................................................................. [no-gen-acls, no-auth]
  |     o- acls ............................................................................ [ACLs: 1]
  |     | o- iqn.2024-12.tp2.b3:data-chunk3:chunk3-initiator ........................ [Mapped LUNs: 1]
  |     |   o- mapped_lun0 ............................................. [lun0 block/data-chunk3 (rw)]
  |     o- luns ............................................................................ [LUNs: 1]
  |     | o- lun0 .................................. [block/data-chunk3 (/dev/md2) (default_tg_pt_gp)]
  |     o- portals ...................................................................... [Portals: 1]
  |       o- 0.0.0.0:3260 ....................................................................... [OK]
  o- loopback ........................................................................... [Targets: 0]
```
## 2. Chunks machine

### A. Simple iSCSI

🌞 **Installer les tools iSCSI sur `chunk1.tp2.b3`**

- c'est le paquet `iscsi-initiator-utils`

```
[guigui@chunk1 ~]$ sudo dnf install iscsi-initiator-utils
```

```
[guigui@chunk2 ~]$ sudo dnf install iscsi-initiator-utils
```

```
[guigui@chunk3 ~]$ sudo dnf install iscsi-initiator-utils
```

🌞 **Configurer un iSCSI initiator**

- utilisez `iscsiadm` pour découvrir les *targets iSCSI* proposés par `sto1` et `sto2`
- faites vos recherches pour la conf, c'est une conf plutôt standard
- les commandes `iscsiadm` génèrent des fichiers dans `/var/lib/iscsi`
- découvrez et utilisez les targets destinés à cette machine, y'en a 4 :
  - `iqn.2024-12.tp2.b3:chunk1` sur
    - `sto1` donc : `10.3.1.1` et `10.3.1.2`
    - `sto2` donc : `10.3.2.1` et `10.3.2.2`
- n'oubliez pas d'activer dès le démarrage de la machine les services iSCSI
  - service `iscsi`
  - service `iscsid`
- je vous donne la première commande :

```
[guigui@chunk1 ~]$ sudo iscsiadm -m discoverydb -t st -p 10.3.1.1:3260 --discover
10.3.1.1:3260,1 iqn.2024-12.tp2.b3:data-chunk3
10.3.1.1:3260,1 iqn.2024-12.tp2.b3:data-chunk2
10.3.1.1:3260,1 iqn.2024-12.tp2.b3:data-chunk1
[guigui@chunk1 ~]$ sudo iscsiadm -m node -T iqn.2024-12.tp2.b3:data-chunk1 -p 10.3.1.1:3260 --login

[guigui@chunk1 ~]$ sudo iscsiadm -m discoverydb -t st -p 10.3.1.2:3260 --discover
10.3.1.2:3260,1 iqn.2024-12.tp2.b3:data-chunk3
10.3.1.2:3260,1 iqn.2024-12.tp2.b3:data-chunk2
10.3.1.2:3260,1 iqn.2024-12.tp2.b3:data-chunk1
[guigui@chunk1 ~]$ sudo iscsiadm -m node -T iqn.2024-12.tp2.b3:data-chunk1 -p 10.3.1.2:3260 --login


[guigui@chunk1 ~]$ sudo iscsiadm -m discoverydb -t st -p 10.3.2.1:3260 --discover
10.3.1.2:3260,1 iqn.2024-12.tp2.b3:data-chunk3
10.3.1.2:3260,1 iqn.2024-12.tp2.b3:data-chunk2
10.3.1.2:3260,1 iqn.2024-12.tp2.b3:data-chunk1
[guigui@chunk1 ~]$ sudo iscsiadm -m node -T iqn.2024-12.tp2.b3:data-chunk1 -p 10.3.2.1:3260 --login


[guigui@chunk1 ~]$ sudo iscsiadm -m discoverydb -t st -p 10.3.2.2:3260 --discover
[guigui@chunk1 ~]$ sudo iscsiadm -m node -T iqn.2024-12.tp2.b3:data-chunk1 -p 10.3.2.2:3260 --login

```

> On peut par exemple aussi lister les hôtes découverts et utilisables avec `sudo iscsiadm -m node`.

🌞 **Modifier la configuration du démon iSCSI**

- le fichier `/etc/iscsi/iscsid.conf`
- modifier la ligne `node.session.timeo.replacement_timeout`
- mettez lui 0 pour valeur, ça donne : `node.session.timeo.replacement_timeout = 0`
- redémarrer les services `iscsi` et `iscsid`
```
# - If the value is 0, IO will be failed immediately.
# - If the value is less than 0, IO will remain queued until the session
# is logged back in, or until the user runs the logout command.
node.session.timeo.replacement_timeout = 0

# To specify the time to wait for login to complete, edit the line.
# The value is in seconds and the default is 15 seconds.
node.conn[0].timeo.login_timeout = 15

# To specify the time to wait for logout to complete, edit the line.
# The value is in seconds and the default is 15 seconds.
node.conn[0].timeo.logout_timeout = 15

# Time interval to wait for on connection before sending a ping.
# The value is in seconds and the default is 5 seconds.
node.conn[0].timeo.noop_out_interval = 5
```

🌞 **Prouvez que la configuration est prête**

- avec `lsblk` on doit voir les volumes
  - on doit donc voir 4 volumes en iSCSI
  - y'en a que deux en réalité (un proposé par `sto1` et un proposé par `sto2`) mais on les utilise depuis deux IPs différents à chaque fois, donc on les voit en double
- une commande `iscsiadm -m session -P 3` aussi dans le compte-rendu : on y voit en détails les connexions iSCSI en cours
```
[guigui@chunk1 ~]$ lsblk
NAME             MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                8:0    0   20G  0 disk
├─sda1             8:1    0    1G  0 part /boot
└─sda2             8:2    0   19G  0 part
  ├─rl_vbox-root 253:0    0   17G  0 lvm  /
  └─rl_vbox-swap 253:1    0    2G  0 lvm  [SWAP]
sdb                8:16   0 1022M  0 disk
sdc                8:32   0 1022M  0 disk
sdd                8:48   0 1022M  0 disk
sde                8:64   0 1022M  0 disk
sr0               11:0    1 1024M  0 rom
```
```
[guigui@chunk1 ~]$ sudo iscsiadm -m session -P 3
iSCSI Transport Class version 2.0-870
version 6.2.1.9
Target: iqn.2024-12.tp2.b3:data-chunk1 (non-flash)
        Current Portal: 10.3.1.1:3260,1
        Persistent Portal: 10.3.1.1:3260,1
                **********
                Interface:
                **********
                Iface Name: default
                Iface Transport: tcp
                Iface Initiatorname: iqn.2024-12.tp2.b3:data-chunk1:chunk1-initiator
                Iface IPaddress: 10.3.1.101
                Iface HWaddress: default
                Iface Netdev: default
                SID: 35
                iSCSI Connection State: LOGGED IN
                iSCSI Session State: LOGGED_IN
                Internal iscsid Session State: NO CHANGE
                *********
                Timeouts:
                *********
                Recovery Timeout: 120
                Target Reset Timeout: 30
                LUN Reset Timeout: 30
                Abort Timeout: 15
                *****
                CHAP:
                *****
                username: <empty>
                password: ********
                username_in: <empty>
                password_in: ********
                ************************
                Negotiated iSCSI params:
                ************************
                HeaderDigest: None
                DataDigest: None
                MaxRecvDataSegmentLength: 262144
                MaxXmitDataSegmentLength: 262144
                FirstBurstLength: 65536
                MaxBurstLength: 262144
                ImmediateData: Yes
                InitialR2T: Yes
                MaxOutstandingR2T: 1
                ************************
                Attached SCSI devices:
                ************************
                Host Number: 3  State: running
                scsi3 Channel 00 Id 0 Lun: 0
                        Attached scsi disk sdb          State: running
        Current Portal: 10.3.2.1:3260,1
        Persistent Portal: 10.3.2.1:3260,1
                **********
                Interface:
                **********
                Iface Name: default
                Iface Transport: tcp
                Iface Initiatorname: iqn.2024-12.tp2.b3:data-chunk1:chunk1-initiator
                Iface IPaddress: 10.3.2.101
                Iface HWaddress: default
                Iface Netdev: default
                SID: 36
                iSCSI Connection State: LOGGED IN
                iSCSI Session State: LOGGED_IN
                Internal iscsid Session State: NO CHANGE
                *********
                Timeouts:
                *********
                Recovery Timeout: 120
                Target Reset Timeout: 30
                LUN Reset Timeout: 30
                Abort Timeout: 15
                *****
                CHAP:
                *****
                username: <empty>
                password: ********
                username_in: <empty>
                password_in: ********
                ************************
                Negotiated iSCSI params:
                ************************
                HeaderDigest: None
                DataDigest: None
                MaxRecvDataSegmentLength: 262144
                MaxXmitDataSegmentLength: 262144
                FirstBurstLength: 65536
                MaxBurstLength: 262144
                ImmediateData: Yes
                InitialR2T: Yes
                MaxOutstandingR2T: 1
                ************************
                Attached SCSI devices:
                ************************
                Host Number: 4  State: running
                scsi4 Channel 00 Id 0 Lun: 0
                        Attached scsi disk sdc          State: running
        Current Portal: 10.3.2.2:3260,1
        Persistent Portal: 10.3.2.2:3260,1
                **********
                Interface:
                **********
                Iface Name: default
                Iface Transport: tcp
                Iface Initiatorname: iqn.2024-12.tp2.b3:data-chunk1:chunk1-initiator
                Iface IPaddress: 10.3.2.101
                Iface HWaddress: default
                Iface Netdev: default
                SID: 37
                iSCSI Connection State: LOGGED IN
                iSCSI Session State: LOGGED_IN
                Internal iscsid Session State: NO CHANGE
                *********
                Timeouts:
                *********
                Recovery Timeout: 120
                Target Reset Timeout: 30
                LUN Reset Timeout: 30
                Abort Timeout: 15
                *****
                CHAP:
                *****
                username: <empty>
                password: ********
                username_in: <empty>
                password_in: ********
                ************************
                Negotiated iSCSI params:
                ************************
                HeaderDigest: None
                DataDigest: None
                MaxRecvDataSegmentLength: 262144
                MaxXmitDataSegmentLength: 262144
                FirstBurstLength: 65536
                MaxBurstLength: 262144
                ImmediateData: Yes
                InitialR2T: Yes
                MaxOutstandingR2T: 1
                ************************
                Attached SCSI devices:
                ************************
                Host Number: 5  State: running
                scsi5 Channel 00 Id 0 Lun: 0
                        Attached scsi disk sdd          State: running
        Current Portal: 10.3.1.2:3260,1
        Persistent Portal: 10.3.1.2:3260,1
                **********
                Interface:
                **********
                Iface Name: default
                Iface Transport: tcp
                Iface Initiatorname: iqn.2024-12.tp2.b3:data-chunk1:chunk1-initiator
                Iface IPaddress: 10.3.1.101
                Iface HWaddress: default
                Iface Netdev: default
                SID: 38
                iSCSI Connection State: LOGGED IN
                iSCSI Session State: LOGGED_IN
                Internal iscsid Session State: NO CHANGE
                *********
                Timeouts:
                *********
                Recovery Timeout: 120
                Target Reset Timeout: 30
                LUN Reset Timeout: 30
                Abort Timeout: 15
                *****
                CHAP:
                *****
                username: <empty>
                password: ********
                username_in: <empty>
                password_in: ********
                ************************
                Negotiated iSCSI params:
                ************************
                HeaderDigest: None
                DataDigest: None
                MaxRecvDataSegmentLength: 262144
                MaxXmitDataSegmentLength: 262144
                FirstBurstLength: 65536
                MaxBurstLength: 262144
                ImmediateData: Yes
                InitialR2T: Yes
                MaxOutstandingR2T: 1
                ************************
                Attached SCSI devices:
                ************************
                Host Number: 6  State: running
                scsi6 Channel 00 Id 0 Lun: 0
                        Attached scsi disk sde          State: running
```

### B. Multipathing

On va configurer le *multipathing* pour que la machine `chunk1` ne voit que deux volumes, chacun ayant deux chemins d'accès possibles : **on ajoute un niveau de redondance pour tolérer les pannes réseau.**

🌞 **Installer les outils multipath sur `chunk1.tp2.b3`**

- c'est le paquet `device-mapper-multipath`

🌞 **Configurer le fichier `/etc/multipath.conf`**

- récupérez le fichier d'exemple `usr/share/doc/device-mapper-multipath/multipath.conf`
- modifier la section ``defaults` comme ceci :

```
[guigui@chunk1 ~]$ sudo grep -A 5 'defaults {' /etc/multipath.conf
defaults {
  user_friendly_names yes
  find_multipaths yes
  path_grouping_policy failover
  features "1 queue_if_no_path"
  no_path_retry 100
--
#defaults {
#       udev_dir                /dev
#       polling_interval        10
#       selector                "round-robin 0"
#       path_grouping_policy    multibus
#       prio                    alua
```

🌞 **Démarrer le service `multipathd`**
```
[guigui@chunk1 ~]$ sudo systemctl status multipathd
● multipathd.service - Device-Mapper Multipath Device Controller
     Loaded: loaded (/usr/lib/systemd/system/multipathd.service; enabled; preset: enabled)
     Active: active (running) since Fri 2024-12-06 16:02:19 CET; 17s ago
TriggeredBy: ○ multipathd.socket
    Process: 2124 ExecStartPre=/sbin/modprobe -a scsi_dh_alua scsi_dh_emc scsi_dh_rdac dm-multipath (code=exited, >
    Process: 2128 ExecStartPre=/sbin/multipath -A (code=exited, status=0/SUCCESS)
   Main PID: 2130 (multipathd)
     Status: "up"
      Tasks: 7
     Memory: 19.5M
        CPU: 356ms
     CGroup: /system.slice/multipathd.service
             └─2130 /sbin/multipathd -d -s

Dec 06 16:02:18 chunk1.tp2.b3 multipathd[2130]: --------start up--------
Dec 06 16:02:18 chunk1.tp2.b3 multipathd[2130]: read /etc/multipath.conf
Dec 06 16:02:18 chunk1.tp2.b3 multipathd[2130]: path checkers start up
Dec 06 16:02:18 chunk1.tp2.b3 multipathd[2130]: mpatha: option 'features "1 queue_if_no_path"' is deprecated, plea>
Dec 06 16:02:18 chunk1.tp2.b3 multipathd[2130]: mpatha: ignoring feature 'queue_if_no_path' because no_path_retry >
Dec 06 16:02:18 chunk1.tp2.b3 multipathd[2130]: mpatha: addmap [0 2093056 multipath 1 queue_if_no_path 1 alua 2 1 >
Dec 06 16:02:19 chunk1.tp2.b3 multipathd[2130]: mpathb: option 'features "1 queue_if_no_path"' is deprecated, plea>
Dec 06 16:02:19 chunk1.tp2.b3 multipathd[2130]: mpathb: ignoring feature 'queue_if_no_path' because no_path_retry >
Dec 06 16:02:19 chunk1.tp2.b3 multipathd[2130]: mpathb: addmap [0 2093056 multipath 1 queue_if_no_path 1 alua 2 1 >
Dec 06 16:02:19 chunk1.tp2.b3 systemd[1]: Started Device-Mapper Multipath Device Controller.
lines 1-24/24 (END)
```
🌞 **Et euh c'est tout, il est smart enough**

- ptit `lsblk`
  - vous devriez avoir des volumes `mpatha` et `mpathb`

```
[guigui@chunk1 ~]$ lsblk
NAME             MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sda                8:0    0   20G  0 disk
├─sda1             8:1    0    1G  0 part  /boot
└─sda2             8:2    0   19G  0 part
  ├─rl_vbox-root 253:0    0   17G  0 lvm   /
  └─rl_vbox-swap 253:1    0    2G  0 lvm   [SWAP]
sdb                8:16   0 1022M  0 disk
└─mpatha         253:2    0 1022M  0 mpath
sdc                8:32   0 1022M  0 disk
└─mpatha         253:2    0 1022M  0 mpath
sdd                8:48   0 1022M  0 disk
└─mpathb         253:3    0 1022M  0 mpath
sde                8:64   0 1022M  0 disk
└─mpathb         253:3    0 1022M  0 mpath
sr0               11:0    1 1024M  0 rom
```
- on peut aussi `multipath -ll` pour avoir + d'infos sur l'état du *multipathing*

```
[guigui@chunk1 ~]$ sudo multipath -ll
mpatha (36001405d363af4e9a69472cbd4711629) dm-2 LIO-ORG,data-chunk1
size=1022M features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=active
| `- 3:0:0:0 sdb 8:16 active ready running
`-+- policy='service-time 0' prio=50 status=enabled
  `- 4:0:0:0 sdc 8:32 active ready running
mpathb (36001405b52f5c056b524a9c8633cf2b0) dm-3 LIO-ORG,data-chunk1
size=1022M features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=active
| `- 5:0:0:0 sdd 8:48 active ready running
`-+- policy='service-time 0' prio=50 status=enabled
  `- 6:0:0:0 sde 8:64 active ready running
```

## 3. Formatage et montage

🌞 **Créez une partition sur les devices `mpatha` et `mpathb`**

- avec `fdisk`

🌞 **Formatez en `xfs` les partitions**

- avec `mkfs`

🌞 **Point de montage `/mnt/data_chunk1`**

- créez-le avec `mkdir`
- utilisez une commande `mount` pour y monter `mpatha`
- écrivez un unité systemd de type `mount` pour avoir un montage automatique
- ajoutez un `After=network-online.target` pour pas qu'il démarre avant le réseau et ne bloque pas le boot
```
[guigui@chunk1 ~]$ sudo cat /etc/systemd/system/mnt-data_chunk2.mount
[Unit]
Description=Mount mpathb1 on /mnt/data_chunk2
After=network-online.target
Wants=network-online.target

[Mount]
What=/dev/mapper/mpathb1
Where=/mnt/data_chunk2
Type=xfs

[Install]
WantedBy=multi-user.target
```

🌞 **Point de montage `/mnt/data_chunk2`**

- pareil, pour `mpathb`

Ca donne ça :

```
[guigui@chunk1 ~]$ lsblk
NAME             MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sda                8:0    0   20G  0 disk
├─sda1             8:1    0    1G  0 part  /boot
└─sda2             8:2    0   19G  0 part
  ├─rl_vbox-root 253:0    0   17G  0 lvm   /
  └─rl_vbox-swap 253:1    0    2G  0 lvm   [SWAP]
sdb                8:16   0 1022M  0 disk
└─mpatha         253:2    0 1022M  0 mpath
  └─mpatha1      253:4    0  990M  0 part  /mnt/data_chunk1
sdc                8:32   0 1022M  0 disk
└─mpatha         253:2    0 1022M  0 mpath
  └─mpatha1      253:4    0  990M  0 part  /mnt/data_chunk1
sdd                8:48   0 1022M  0 disk
└─mpathb         253:3    0 1022M  0 mpath
  └─mpathb1      253:5    0  990M  0 part  /mnt/data_chunk2
sde                8:64   0 1022M  0 disk
└─mpathb         253:3    0 1022M  0 mpath
  └─mpathb1      253:5    0  990M  0 part  /mnt/data_chunk2
sr0               11:0    1 1024M  0 rom
[guigui@chunk1 ~]$ df -h
Filesystem                Size  Used Avail Use% Mounted on
devtmpfs                  4.0M     0  4.0M   0% /dev
tmpfs                     888M     0  888M   0% /dev/shm
tmpfs                     355M  9.5M  346M   3% /run
/dev/mapper/rl_vbox-root   17G  1.6G   16G   9% /
/dev/sda1                 960M  314M  647M  33% /boot
tmpfs                     178M     0  178M   0% /run/user/1000
/dev/mapper/mpatha1       927M   39M  888M   5% /mnt/data_chunk1
/dev/mapper/mpathb1       927M   39M  888M   5% /mnt/data_chunk2
```

## 4. Tests

On va simuler une coupure réseau, en coupant l'un des liens sur `chunk1.tp2.b3`.

On va suivre en temps réel la bascule iSCSI.

L'idée :

- on lance une écriture continue sur le volume monté en multipath avec un ptit boucle shell
  - `while true; do date >> /mnt/iscsidisk/date; done`
  - cette ligne fait le taff, elle écrit continuellement l'heure dans un fichier
  - comme ça on verra quand ça stoppe ou non les écritures
- on coupe une interface réseau
  - utilisez une commande `date` juste avant de couper, pour savoir l'heure précise à laquelle vous coupez
  - rallumez au bout de ~30 secondes/1 minute (pareil, en tapant `date` avant)
- on observe la bascule iSCSI
  - on peut regarder les logs en temps réel, et observer aussi la sortie de `multipath -ll` : `watch -n 0.5 multipath -ll`

### A. Simulation de panne

🌞 **Simuler une coupure réseau**

```
[guigui@chunk1 ~]$ date
Fri Dec  6 05:37:02 PM CET 2024
[guigui@chunk1 ~]$ sudo ip link set enp0s9 down
[guigui@chunk1 ~]$ sudo multipath -ll
mpatha (36001405d363af4e9a69472cbd4711629) dm-2 LIO-ORG,data-chunk1
size=1022M features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=0 status=enabled
| `- 3:0:0:0 sdb 8:16 failed faulty running
`-+- policy='service-time 0' prio=50 status=active
  `- 4:0:0:0 sdc 8:32 active ready running
mpathb (36001405b52f5c056b524a9c8633cf2b0) dm-3 LIO-ORG,data-chunk1
size=1022M features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=active
| `- 5:0:0:0 sdd 8:48 active ready running
`-+- policy='service-time 0' prio=0 status=enabled
  `- 6:0:0:0 sde 8:64 failed faulty running
[guigui@chunk1 ~]$ sudo ip link set enp0s9 up
[guigui@chunk1 ~]$ sudo multipath -ll
mpatha (36001405d363af4e9a69472cbd4711629) dm-2 LIO-ORG,data-chunk1
size=1022M features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=enabled
| `- 3:0:0:0 sdb 8:16 active ready running
`-+- policy='service-time 0' prio=50 status=active
  `- 4:0:0:0 sdc 8:32 active ready running
mpathb (36001405b52f5c056b524a9c8633cf2b0) dm-3 LIO-ORG,data-chunk1
size=1022M features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=active
| `- 5:0:0:0 sdd 8:48 active ready running
`-+- policy='service-time 0' prio=50 status=enabled
  `- 6:0:0:0 sde 8:64 active ready running
```

### B. Jouer avec les paramètres

🌞 **Resimuler une panne**

- on devrait observer une bascule légèrement plus rapide

```
See system logs and 'systemctl status multipathd.service' for details.
[guigui@chunk1 iscsidisk]$ sudo systemctl restart multipathd
[guigui@chunk1 iscsidisk]$ date
Fri Dec  6 05:46:05 PM CET 2024
[guigui@chunk1 iscsidisk]$  sudo multipath -ll
mpatha (36001405d363af4e9a69472cbd4711629) dm-2 LIO-ORG,data-chunk1
size=1022M features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=enabled
| `- 3:0:0:0 sdb 8:16 active ready running
`-+- policy='service-time 0' prio=50 status=enabled
  `- 4:0:0:0 sdc 8:32 active ready running
mpathb (36001405b52f5c056b524a9c8633cf2b0) dm-3 LIO-ORG,data-chunk1
size=1022M features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=enabled
| `- 5:0:0:0 sdd 8:48 active ready running
`-+- policy='service-time 0' prio=50 status=enabled
  `- 6:0:0:0 sde 8:64 active ready running
[guigui@chunk1 iscsidisk]$ sudo ip link set enp0s9 down
[guigui@chunk1 iscsidisk]$  sudo multipath -ll
mpatha (36001405d363af4e9a69472cbd4711629) dm-2 LIO-ORG,data-chunk1
size=1022M features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=0 status=enabled
| `- 3:0:0:0 sdb 8:16 failed faulty running
`-+- policy='service-time 0' prio=50 status=enabled
  `- 4:0:0:0 sdc 8:32 active ready running
mpathb (36001405b52f5c056b524a9c8633cf2b0) dm-3 LIO-ORG,data-chunk1
size=1022M features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=enabled
| `- 5:0:0:0 sdd 8:48 active ready running
`-+- policy='service-time 0' prio=0 status=enabled
  `- 6:0:0:0 sde 8:64 failed faulty running
[guigui@chunk1 iscsidisk]$ sudo ip link set enp0s9 up
[guigui@chunk1 iscsidisk]$  sudo multipath -ll
mpatha (36001405d363af4e9a69472cbd4711629) dm-2 LIO-ORG,data-chunk1
size=1022M features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=enabled
| `- 3:0:0:0 sdb 8:16 active ready running
`-+- policy='service-time 0' prio=50 status=enabled
  `- 4:0:0:0 sdc 8:32 active ready running
mpathb (36001405b52f5c056b524a9c8633cf2b0) dm-3 LIO-ORG,data-chunk1
size=1022M features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=enabled
| `- 5:0:0:0 sdd 8:48 active ready running
`-+- policy='service-time 0' prio=50 status=enabled
  `- 6:0:0:0 sde 8:64 active ready running
```
## 5. Replicate

➜ **Répliquer le setup de `chunk1` sur les machines `chunk2` et `chunk3`**

- chaque machine ne doit accéder qu'aux targets iSCSI qui lui sont destinés
  - chaque machine "chunk" a un IQN qui lui est dédié
  - la machine `chunk1`, on lui a donné accès qu'aux IQN `iqn.2024-12.tp2.b3:chunk1` (sur chaque IP)
  - il faut donc faire pareil la machine `chunk2` et sur `chunk3`
- même setup :
  - `iscsiadm` pour les 4 targets iSCSI
  - puis multipath pour en avoir plus que 2
  - vous formatez en `xfs` les deux volumes `mpatha` et `mpathb` et montez sur `/mnt/data_chunk1` et `/mnt/data_chunk2`

🌞 **Preuve du setup**

- sur `chunk2.tp2.b3` et sur `chunk3.tp2.b3`
- les commandes suivantes :

### chunk2
```
[guigui@chunk2 ~]$ sudo iscsiadm -m session -P 3
sudo multipath -ll
lsblk
[sudo] password for guigui:
iSCSI Transport Class version 2.0-870
version 6.2.1.9
Target: iqn.2024-12.tp2.b3:data-chunk2 (non-flash)
        Current Portal: 10.3.1.1:3260,1
        Persistent Portal: 10.3.1.1:3260,1
                **********
                Interface:
                **********
                Iface Name: default
                Iface Transport: tcp
                Iface Initiatorname: iqn.2024-12.tp2.b3:data-chunk2:chunk2-initiator
                Iface IPaddress: 10.3.1.102
                Iface HWaddress: default
                Iface Netdev: default
                SID: 11
                iSCSI Connection State: LOGGED IN
                iSCSI Session State: LOGGED_IN
                Internal iscsid Session State: NO CHANGE
                *********
                Timeouts:
                *********
                Recovery Timeout: 5
                Target Reset Timeout: 30
                LUN Reset Timeout: 30
                Abort Timeout: 15
                *****
                CHAP:
                *****
                username: <empty>
                password: ********
                username_in: <empty>
                password_in: ********
                ************************
                Negotiated iSCSI params:
                ************************
                HeaderDigest: None
                DataDigest: None
                MaxRecvDataSegmentLength: 262144
                MaxXmitDataSegmentLength: 262144
                FirstBurstLength: 65536
                MaxBurstLength: 262144
                ImmediateData: Yes
                InitialR2T: Yes
                MaxOutstandingR2T: 1
                ************************
                Attached SCSI devices:
                ************************
                Host Number: 3  State: running
                scsi3 Channel 00 Id 0 Lun: 0
                        Attached scsi disk sdb          State: running
        Current Portal: 10.3.2.1:3260,1
        Persistent Portal: 10.3.2.1:3260,1
                **********
                Interface:
                **********
                Iface Name: default
                Iface Transport: tcp
                Iface Initiatorname: iqn.2024-12.tp2.b3:data-chunk2:chunk2-initiator
                Iface IPaddress: 10.3.2.102
                Iface HWaddress: default
                Iface Netdev: default
                SID: 12
                iSCSI Connection State: LOGGED IN
                iSCSI Session State: LOGGED_IN
                Internal iscsid Session State: NO CHANGE
                *********
                Timeouts:
                *********
                Recovery Timeout: 5
                Target Reset Timeout: 30
                LUN Reset Timeout: 30
                Abort Timeout: 15
                *****
                CHAP:
                *****
                username: <empty>
                password: ********
                username_in: <empty>
                password_in: ********
                ************************
                Negotiated iSCSI params:
                ************************
                HeaderDigest: None
                DataDigest: None
                MaxRecvDataSegmentLength: 262144
                MaxXmitDataSegmentLength: 262144
                FirstBurstLength: 65536
                MaxBurstLength: 262144
                ImmediateData: Yes
                InitialR2T: Yes
                MaxOutstandingR2T: 1
                ************************
                Attached SCSI devices:
                ************************
                Host Number: 4  State: running
                scsi4 Channel 00 Id 0 Lun: 0
                        Attached scsi disk sdc          State: running
        Current Portal: 10.3.1.2:3260,1
        Persistent Portal: 10.3.1.2:3260,1
                **********
                Interface:
                **********
                Iface Name: default
                Iface Transport: tcp
                Iface Initiatorname: iqn.2024-12.tp2.b3:data-chunk2:chunk2-initiator
                Iface IPaddress: 10.3.1.102
                Iface HWaddress: default
                Iface Netdev: default
                SID: 13
                iSCSI Connection State: LOGGED IN
                iSCSI Session State: LOGGED_IN
                Internal iscsid Session State: NO CHANGE
                *********
                Timeouts:
                *********
                Recovery Timeout: 5
                Target Reset Timeout: 30
                LUN Reset Timeout: 30
                Abort Timeout: 15
                *****
                CHAP:
                *****
                username: <empty>
                password: ********
                username_in: <empty>
                password_in: ********
                ************************
                Negotiated iSCSI params:
                ************************
                HeaderDigest: None
                DataDigest: None
                MaxRecvDataSegmentLength: 262144
                MaxXmitDataSegmentLength: 262144
                FirstBurstLength: 65536
                MaxBurstLength: 262144
                ImmediateData: Yes
                InitialR2T: Yes
                MaxOutstandingR2T: 1
                ************************
                Attached SCSI devices:
                ************************
                Host Number: 5  State: running
                scsi5 Channel 00 Id 0 Lun: 0
                        Attached scsi disk sdd          State: running
        Current Portal: 10.3.2.2:3260,1
        Persistent Portal: 10.3.2.2:3260,1
                **********
                Interface:
                **********
                Iface Name: default
                Iface Transport: tcp
                Iface Initiatorname: iqn.2024-12.tp2.b3:data-chunk2:chunk2-initiator
                Iface IPaddress: 10.3.2.102
                Iface HWaddress: default
                Iface Netdev: default
                SID: 14
                iSCSI Connection State: LOGGED IN
                iSCSI Session State: LOGGED_IN
                Internal iscsid Session State: NO CHANGE
                *********
                Timeouts:
                *********
                Recovery Timeout: 5
                Target Reset Timeout: 30
                LUN Reset Timeout: 30
                Abort Timeout: 15
                *****
                CHAP:
                *****
                username: <empty>
                password: ********
                username_in: <empty>
                password_in: ********
                ************************
                Negotiated iSCSI params:
                ************************
                HeaderDigest: None
                DataDigest: None
                MaxRecvDataSegmentLength: 262144
                MaxXmitDataSegmentLength: 262144
                FirstBurstLength: 65536
                MaxBurstLength: 262144
                ImmediateData: Yes
                InitialR2T: Yes
                MaxOutstandingR2T: 1
                ************************
                Attached SCSI devices:
                ************************
                Host Number: 6  State: running
                scsi6 Channel 00 Id 0 Lun: 0
                        Attached scsi disk sde          State: running
mpatha (36001405c560773d6f8c43629614f044b) dm-2 LIO-ORG,data-chunk2
size=1022M features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=active
| `- 3:0:0:0 sdb 8:16 active ready running
`-+- policy='service-time 0' prio=50 status=enabled
  `- 4:0:0:0 sdc 8:32 active ready running
mpathb (36001405de1c575803614c3295b239396) dm-3 LIO-ORG,data-chunk2
size=1022M features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=active
| `- 5:0:0:0 sdd 8:48 active ready running
`-+- policy='service-time 0' prio=50 status=enabled
  `- 6:0:0:0 sde 8:64 active ready running
NAME             MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sda                8:0    0   20G  0 disk
├─sda1             8:1    0    1G  0 part  /boot
└─sda2             8:2    0   19G  0 part
  ├─rl_vbox-root 253:0    0   17G  0 lvm   /
  └─rl_vbox-swap 253:1    0    2G  0 lvm   [SWAP]
sdb                8:16   0 1022M  0 disk
└─mpatha         253:2    0 1022M  0 mpath /mnt/data_chunk1
sdc                8:32   0 1022M  0 disk
└─mpatha         253:2    0 1022M  0 mpath /mnt/data_chunk1
sdd                8:48   0 1022M  0 disk
└─mpathb         253:3    0 1022M  0 mpath /mnt/data_chunk2
sde                8:64   0 1022M  0 disk
└─mpathb         253:3    0 1022M  0 mpath /mnt/data_chunk2
sr0               11:0    1 1024M  0 rom
[guigui@chunk2 ~]$
```
### chunk3
```
[guigui@chunk3 ~]$ sudo iscsiadm -m session -P 3
sudo multipath -ll
lsblk
iSCSI Transport Class version 2.0-870
version 6.2.1.9
Target: iqn.2024-12.tp2.b3:data-chunk3 (non-flash)
        Current Portal: 10.3.1.1:3260,1
        Persistent Portal: 10.3.1.1:3260,1
                **********
                Interface:
                **********
                Iface Name: default
                Iface Transport: tcp
                Iface Initiatorname: iqn.2024-12.tp2.b3:data-chunk3:chunk3-initiator
                Iface IPaddress: 10.3.1.103
                Iface HWaddress: default
                Iface Netdev: default
                SID: 2
                iSCSI Connection State: LOGGED IN
                iSCSI Session State: LOGGED_IN
                Internal iscsid Session State: NO CHANGE
                *********
                Timeouts:
                *********
                Recovery Timeout: 5
                Target Reset Timeout: 30
                LUN Reset Timeout: 30
                Abort Timeout: 15
                *****
                CHAP:
                *****
                username: <empty>
                password: ********
                username_in: <empty>
                password_in: ********
                ************************
                Negotiated iSCSI params:
                ************************
                HeaderDigest: None
                DataDigest: None
                MaxRecvDataSegmentLength: 262144
                MaxXmitDataSegmentLength: 262144
                FirstBurstLength: 65536
                MaxBurstLength: 262144
                ImmediateData: Yes
                InitialR2T: Yes
                MaxOutstandingR2T: 1
                ************************
                Attached SCSI devices:
                ************************
                Host Number: 3  State: running
                scsi3 Channel 00 Id 0 Lun: 0
                        Attached scsi disk sdb          State: running
        Current Portal: 10.3.1.2:3260,1
        Persistent Portal: 10.3.1.2:3260,1
                **********
                Interface:
                **********
                Iface Name: default
                Iface Transport: tcp
                Iface Initiatorname: iqn.2024-12.tp2.b3:data-chunk3:chunk3-initiator
                Iface IPaddress: 10.3.1.103
                Iface HWaddress: default
                Iface Netdev: default
                SID: 3
                iSCSI Connection State: LOGGED IN
                iSCSI Session State: LOGGED_IN
                Internal iscsid Session State: NO CHANGE
                *********
                Timeouts:
                *********
                Recovery Timeout: 5
                Target Reset Timeout: 30
                LUN Reset Timeout: 30
                Abort Timeout: 15
                *****
                CHAP:
                *****
                username: <empty>
                password: ********
                username_in: <empty>
                password_in: ********
                ************************
                Negotiated iSCSI params:
                ************************
                HeaderDigest: None
                DataDigest: None
                MaxRecvDataSegmentLength: 262144
                MaxXmitDataSegmentLength: 262144
                FirstBurstLength: 65536
                MaxBurstLength: 262144
                ImmediateData: Yes
                InitialR2T: Yes
                MaxOutstandingR2T: 1
                ************************
                Attached SCSI devices:
                ************************
                Host Number: 4  State: running
                scsi4 Channel 00 Id 0 Lun: 0
                        Attached scsi disk sdc          State: running
        Current Portal: 10.3.2.1:3260,1
        Persistent Portal: 10.3.2.1:3260,1
                **********
                Interface:
                **********
                Iface Name: default
                Iface Transport: tcp
                Iface Initiatorname: iqn.2024-12.tp2.b3:data-chunk3:chunk3-initiator
                Iface IPaddress: 10.3.2.103
                Iface HWaddress: default
                Iface Netdev: default
                SID: 4
                iSCSI Connection State: LOGGED IN
                iSCSI Session State: LOGGED_IN
                Internal iscsid Session State: NO CHANGE
                *********
                Timeouts:
                *********
                Recovery Timeout: 5
                Target Reset Timeout: 30
                LUN Reset Timeout: 30
                Abort Timeout: 15
                *****
                CHAP:
                *****
                username: <empty>
                password: ********
                username_in: <empty>
                password_in: ********
                ************************
                Negotiated iSCSI params:
                ************************
                HeaderDigest: None
                DataDigest: None
                MaxRecvDataSegmentLength: 262144
                MaxXmitDataSegmentLength: 262144
                FirstBurstLength: 65536
                MaxBurstLength: 262144
                ImmediateData: Yes
                InitialR2T: Yes
                MaxOutstandingR2T: 1
                ************************
                Attached SCSI devices:
                ************************
                Host Number: 5  State: running
                scsi5 Channel 00 Id 0 Lun: 0
                        Attached scsi disk sdd          State: running
        Current Portal: 10.3.2.2:3260,1
        Persistent Portal: 10.3.2.2:3260,1
                **********
                Interface:
                **********
                Iface Name: default
                Iface Transport: tcp
                Iface Initiatorname: iqn.2024-12.tp2.b3:data-chunk3:chunk3-initiator
                Iface IPaddress: 10.3.2.103
                Iface HWaddress: default
                Iface Netdev: default
                SID: 5
                iSCSI Connection State: LOGGED IN
                iSCSI Session State: LOGGED_IN
                Internal iscsid Session State: NO CHANGE
                *********
                Timeouts:
                *********
                Recovery Timeout: 5
                Target Reset Timeout: 30
                LUN Reset Timeout: 30
                Abort Timeout: 15
                *****
                CHAP:
                *****
                username: <empty>
                password: ********
                username_in: <empty>
                password_in: ********
                ************************
                Negotiated iSCSI params:
                ************************
                HeaderDigest: None
                DataDigest: None
                MaxRecvDataSegmentLength: 262144
                MaxXmitDataSegmentLength: 262144
                FirstBurstLength: 65536
                MaxBurstLength: 262144
                ImmediateData: Yes
                InitialR2T: Yes
                MaxOutstandingR2T: 1
                ************************
                Attached SCSI devices:
                ************************
                Host Number: 6  State: running
                scsi6 Channel 00 Id 0 Lun: 0
                        Attached scsi disk sde          State: running
mpatha (360014052a7dd789dd0f4d84b5ac075c9) dm-2 LIO-ORG,data-chunk3
size=1022M features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=active
| `- 3:0:0:0 sdb 8:16 active ready running
`-+- policy='service-time 0' prio=50 status=enabled
  `- 5:0:0:0 sdd 8:48 active ready running
mpathb (3600140546c6e841d9a34d94a03585995) dm-3 LIO-ORG,data-chunk3
size=1022M features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' prio=50 status=active
| `- 4:0:0:0 sdc 8:32 active ready running
`-+- policy='service-time 0' prio=50 status=enabled
  `- 6:0:0:0 sde 8:64 active ready running
NAME             MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sda                8:0    0   20G  0 disk
├─sda1             8:1    0    1G  0 part  /boot
└─sda2             8:2    0   19G  0 part
  ├─rl_vbox-root 253:0    0   17G  0 lvm   /
  └─rl_vbox-swap 253:1    0    2G  0 lvm   [SWAP]
sdb                8:16   0 1022M  0 disk
└─mpatha         253:2    0 1022M  0 mpath /mnt/data_chunk1
sdc                8:32   0 1022M  0 disk
└─mpathb         253:3    0 1022M  0 mpath /mnt/data_chunk2
sdd                8:48   0 1022M  0 disk
└─mpatha         253:2    0 1022M  0 mpath /mnt/data_chunk1
sde                8:64   0 1022M  0 disk
└─mpathb         253:3    0 1022M  0 mpath /mnt/data_chunk2
sr0               11:0    1 1024M  0 rom
```

<br>

# III. Distributed filesystem

🌞 **Installer les paquets nécessaires pour avoir un Moose Master**

```
[guigui@master ~]$ sudo dnf install moosefs-master
[sudo] password for guigui: 
MooseFS 9 - x86_64                                                                                                                                            17 kB/s | 6.1 kB     00:00    
Dependencies resolved.
=============================================================================================================================================================================================
 Package                                         Architecture                            Version                                              Repository                                Size
=============================================================================================================================================================================================
Installing:
 moosefs-master                                  x86_64                                  3.0.118-1.rhsystemd                                  MooseFS                                  379 k

Transaction Summary
=============================================================================================================================================================================================
Install  1 Package

Total download size: 379 k
Installed size: 873 k
Is this ok [y/N]: y
Downloading Packages:
moosefs-master-3.0.118-1.rhsystemd.x86_64.rpm                                                                                                                427 kB/s | 379 kB     00:00    
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Total                                                                                                                                                        424 kB/s | 379 kB     00:00     
MooseFS 9 - x86_64                                                                                                                                           1.7 MB/s | 1.8 kB     00:00    
Importing GPG key 0xCF82ADBA:
 Userid     : "MooseFS Development Team (Official MooseFS Repositories) <support@moosefs.com>"
 Fingerprint: 0F42 5A56 8FAF F43D EA2F 6843 0427 65FC CF82 ADBA
 From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-MooseFS
Is this ok [y/N]: y
Key imported successfully
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                                                                                                                                     1/1 
  Running scriptlet: moosefs-master-3.0.118-1.rhsystemd.x86_64                                                                                                                           1/1 
  Installing       : moosefs-master-3.0.118-1.rhsystemd.x86_64                                                                                                                           1/1 
  Running scriptlet: moosefs-master-3.0.118-1.rhsystemd.x86_64                                                                                                                           1/1 
  Verifying        : moosefs-master-3.0.118-1.rhsystemd.x86_64                                                                                                                           1/1 

Installed:
  moosefs-master-3.0.118-1.rhsystemd.x86_64                                                                                                                                                  

Complete!
[guigui@master ~]$ cd /etc/mfs
[guigui@master mfs]$ cp mfsmaster.cfg.sample mfsmaster.cfg
cp: cannot create regular file 'mfsmaster.cfg': Permission denied
[guigui@master mfs]$ sudo !!
sudo cp mfsmaster.cfg.sample mfsmaster.cfg
[guigui@master mfs]$ ll
total 48
-rw-r--r--. 1 root root  6441 Dec  8 21:05 mfsexports.cfg
-rw-r--r--. 1 root root  6441 Aug  2 17:18 mfsexports.cfg.sample
-rw-r--r--. 1 root root 11618 Dec  8 21:05 mfsmaster.cfg
-rw-r--r--. 1 root root 11618 Aug  2 17:18 mfsmaster.cfg.sample
-rw-r--r--. 1 root root  2588 Dec  8 21:05 mfstopology.cfg
-rw-r--r--. 1 root root  2588 Aug  2 17:18 mfstopology.cfg.sample
[guigui@master mfs]$ sudo cp mfsexports.cfg.sample mfsexports.cfg
```

🌞 **Démarrez les services du Moose Master**

```
[guigui@master ~]$ su -
Password: 
Last login: Thu Dec  5 14:45:01 CET 2024 on tty1
[root@chunk1 ~]# curl "https://repository.moosefs.com/RPM-GPG-KEY-Moosefs" > /etc/pki/rpm-gpg/RPM-GPG-KEY-Moosefs
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   196  100   196    0     0    715      0 --:--:-- --:--:-- --:--:--   715
[root@master ~]# curl "https://repository.moosefs.com/MooseFS-3-el9.repo" > /etc/yum.repos.d/MooseFS.repo
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   175  100   175    0     0    660      0 --:--:-- --:--:-- --:--:--   660
```

🌞 **Ouvrez les ports firewall**

```
[guigui@master ~]$ sudo firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: enp0s3 enp0s8 enp0s9
  sources: 
  services: cockpit dhcpv6-client ssh
  ports: 80/tcp 9425/tcp 9419/tcp 9420/tcp 9421/tcp
  protocols: 
  forward: yes
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
```

J'ai passé un bon moment à chercher sans succès.
pour installer les package

```
[guigui@chunk1 iscsidisk]$ sudo dnf install moosefs-chunkserver -y
Warning: failed loading '/etc/yum.repos.d/MooseFS-3-el9.repo', skipping.
MooseFS 4.x                                                100  B/s | 196  B     00:01
Errors during downloading metadata for repository 'moosefs':
  - Status code: 404 for https://repository.moosefs.com/MooseFS-4-el9/repodata/repomd.xml (IP: 46.17.113.51)
Error: Failed to download metadata for repo 'moosefs': Cannot download repomd.xml: Cannot download repodata/repomd.xml: All mirrors were tried
```

désolé de ne pas avoir pu terminer le tp bonne correction