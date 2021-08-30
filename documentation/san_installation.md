# Configuring Raspberry PI as SAN for the lab cluster

The idea is to configure one of the Raspberry PIs as a SAN, connecting some SSD Disk to it (through USB 3.0 ports) and providing LUNs to all the cluster nodes through iSCSI.

A storage on a network is called iSCSI Target, a Client which connects to iSCSI Target is called iSCSI Initiator. In my home lab, `gateway` will be the iSCSI Target and `node1-node4` will be the iSCSI Initiators.

```
+----------------------+         |             +----------------------+
| [   iSCSI Target   ] |10.0.0.1 | 10.0.0.11-14| [ iSCSI Initiator  ] |
|        gateway       +---------+-------------+        node1-4       |
|                      |                       |                      |
+----------------------+                       +----------------------+

```

LIO, [LinuxIO](http://linux-iscsi.org/wiki/Main_Page), has been the Linux SCSI target since kernel version 2.6.38.
It support sharing different types of storage fabrics and backstorage devices, including block devices (including LVM logical volumes and physical devices).
[targetcli](http://linux-iscsi.org/wiki/Targetcli) is the single-node LinuxIO management shell developed by Datera, that is part of most linux distributions

**LUN is a Logical Unit Number**, which shared from the iSCSI Storage Server. The Physical drive of iSCSI target server shares its drive to initiator over TCP/IP network. A Collection of drives called LUNs to form a large storage as SAN (Storage Area Network). 

In real environment LUNs are defined in LVM, if so it can be expandable as per space requirements.

![LUNs-on-LVM](images/Creating_LUNs_using_LVM.png "Creating LUNS using LVM")

## iSCSI Qualifier Names (iqn)

Unique identifier are asigned to iSCSI Initiators and iSCSI targets
Format iqn.yyyy-mm.reverse_domain_name:any

In my case I will use hostname to make iqn unique

    iqn.2021-07.com.ricsanfre.picluster:<hostname>

## Hardware

A Kingston A400 480GB SSD Disk and a SATA Disk USB3.0 Case will be used for building Raspberry PI cluster.

## Preparing SSD Disk and LVM configuration for LUNS

### Step 1. Connect SSD Disk through USB 3.0 port

### Step 2. Partition HDD with `fdisk`

Add one primary partition (type LVM) using all disk space available.

```
sudo fdisk -c -u /dev/sdb

Welcome to fdisk (util-linux 2.34).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0x5721e48c.

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1): 1
First sector (2048-41943039, default 2048):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-41943039, default 41943039):

Created a new partition 1 of type 'Linux' and of size 20 GiB.

Command (m for help): t
Selected partition 1
Hex code (type L to list all codes): L

 0  Empty           24  NEC DOS         81  Minix / old Lin bf  Solaris
 1  FAT12           27  Hidden NTFS Win 82  Linux swap / So c1  DRDOS/sec (FAT-
 2  XENIX root      39  Plan 9          83  Linux           c4  DRDOS/sec (FAT-
 3  XENIX usr       3c  PartitionMagic  84  OS/2 hidden or  c6  DRDOS/sec (FAT-
 4  FAT16 <32M      40  Venix 80286     85  Linux extended  c7  Syrinx
 5  Extended        41  PPC PReP Boot   86  NTFS volume set da  Non-FS data
 6  FAT16           42  SFS             87  NTFS volume set db  CP/M / CTOS / .
 7  HPFS/NTFS/exFAT 4d  QNX4.x          88  Linux plaintext de  Dell Utility
 8  AIX             4e  QNX4.x 2nd part 8e  Linux LVM       df  BootIt
 9  AIX bootable    4f  QNX4.x 3rd part 93  Amoeba          e1  DOS access
 a  OS/2 Boot Manag 50  OnTrack DM      94  Amoeba BBT      e3  DOS R/O
 b  W95 FAT32       51  OnTrack DM6 Aux 9f  BSD/OS          e4  SpeedStor
 c  W95 FAT32 (LBA) 52  CP/M            a0  IBM Thinkpad hi ea  Rufus alignment
 e  W95 FAT16 (LBA) 53  OnTrack DM6 Aux a5  FreeBSD         eb  BeOS fs
 f  W95 Ext'd (LBA) 54  OnTrackDM6      a6  OpenBSD         ee  GPT
10  OPUS            55  EZ-Drive        a7  NeXTSTEP        ef  EFI (FAT-12/16/
11  Hidden FAT12    56  Golden Bow      a8  Darwin UFS      f0  Linux/PA-RISC b
12  Compaq diagnost 5c  Priam Edisk     a9  NetBSD          f1  SpeedStor
14  Hidden FAT16 <3 61  SpeedStor       ab  Darwin boot     f4  SpeedStor
16  Hidden FAT16    63  GNU HURD or Sys af  HFS / HFS+      f2  DOS secondary
17  Hidden HPFS/NTF 64  Novell Netware  b7  BSDI fs         fb  VMware VMFS
18  AST SmartSleep  65  Novell Netware  b8  BSDI swap       fc  VMware VMKCORE
1b  Hidden W95 FAT3 70  DiskSecure Mult bb  Boot Wizard hid fd  Linux raid auto
1c  Hidden W95 FAT3 75  PC/IX           bc  Acronis FAT32 L fe  LANstep
1e  Hidden W95 FAT1 80  Old Minix       be  Solaris boot    ff  BBT
Hex code (type L to list all codes): 8e
Changed type of partition 'Linux' to 'Linux LVM'.

Command (m for help): p
Disk /dev/sdb: 20 GiB, 21474836480 bytes, 41943040 sectors
Disk model: VBOX HARDDISK
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x5721e48c

Device     Boot Start      End  Sectors Size Id Type
/dev/sdb1        2048 41943039 41940992  20G 8e Linux LVM

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```

### Step 3. Create LVM Physical Volume

    sudo pvcreate /dev/sdb1

### Step 4. Create LVM Volumen Group for iSCSI

    sudo vgcreate vg_iscsi /dev/sdb1

### Step 5. Create LVM Logical Volume associated to LUNs

    sudo lvcreate -L 4G -n lv_iscsi_1 vg_iscsi

    sudo lvcreate -L 4G -n lv_iscsi_2 vg_iscsi

    ...


### Step 6. Check Logical volumes

List the Physical volume, Volume group, logical volumes to confirm:

- List phisical volume
    `sudo pvs` or `sudo pvdisplay`
- List volume group
    `sudo vgs` or `sudo vgdisplay`
- List logical volumes
    `sudo lvs` or `sudo lvdisplay`

## Configuring Target iSCSI

### Step 1. Installing `targetcli`

`targetcli` is a command shell for managing the Linux LIO kernel target

    sudo apt install targetcli-fb

### Step 2. Execute `targetcli`

    sudo targetcli


### Step 3. Disable auto addind of mapped LUNs

By default all LUNs configured at target level are assigned automatically to any iSCSI initiator which is created (Target ACL). To avoid this and enabling manual allocation of LUNs a targetcli global preference must be setting

   set global auto_add_mapped_luns=false


### Step 4. Create an iSCSI Target and Target Port Group (TPG)

   cd iscsi/
   create iqn.2021-07.com.ricsanfre.picluster:iscsi-server

```
sudo targetcli
Warning: Could not load preferences file /root/.targetcli/prefs.bin.
targetcli shell version 2.1.51
Copyright 2011-2013 by Datera, Inc and others.
For help on commands, type 'help'.

/> cd iscsi
/iscsi> create iqn.2021-07.com.ricsanfre.picluster:iscsi-server
Created target iqn.2021-07.com.ricsanfre.picluster:iscsi-server.
Created TPG 1.
Global pref auto_add_default_portal=true
Created default portal listening on all IPs (0.0.0.0), port 3260.
```

### Step 5. Create a backstore (bock devices associated to LVM Logical Volumes created before).


    cd /backstores/block
    create <block_id> <block_dev_path>

```
/> cd /backstores/block
/backstores/block> create block0 /dev/vg_iscsi/lv_iscsi_0
Created block storage object block0 using /dev/vg_iscsi/lv_iscsi_0.
```

### Step 6. Create LUNs

    cd /iscsi/<target_iqn>/tpg1/luns
    create storage_object=<block_storage> lun=<lun_id>


```
/> cd /iscsi/iqn.2021-07.com.ricsanfre.vbox:iscsi-server/tpg1/luns
/iscsi/iqn.20...ver/tpg1/luns> create /backstores/block/iscsi-client-vol1
Created LUN 0.
```
### Step 7. Create an Access Control List (ACL) for security and access to the Target.

In the Initiator server check the iqn (iSCSI Qualifier Name) within the file `/etc/iscsi/initiatorname.iscsi`

> NOTE: Assign unique iqn (iSCSI Initiator Qualifier Name) to each cluster node (`node1-4`). See section [Configuring iSCSI Inititator](#configuring-iscsi-initiator)

Create ACL for the iSCSI Initiator. 


    cd /iscsi/<target_iqn>/tpg1/acls
    create <initiator_iqn>


Specify userid and password for initiator and target (mutual authentication)

    cd /iscsi/<target_iqn>/tpg1/acls/<initiator_iqn>
    set auth userid=<initiator_iqn>
    set auth password=<initiator_password>
    set auth mutual_userid=<target_iqn>
    set auth mutual_passwird=<target_password>


```
sudo targetcli
/> cd iscsi
/iscsi> create iqn.2021-07.com.ricsanfre.vbox:iscsi-server
Created target iqn.2021-07.com.ricsanfre.vbox:iscsi-server.
Created TPG 1.
Global pref auto_add_default_portal=true
Created default portal listening on all IPs (0.0.0.0), port 3260.
/iscsi> cd /backstores/block
/backstores/block> create iscsi-client-vol1 /dev/vg_iscsi/lv_iscsi_1
Created block storage object iscsi-client-vol1 using /dev/vg_iscsi/lv_iscsi_1.
/backstores/block> cd /iscsi/iqn.2021-07.com.ricsanfre.vbox:iscsi-server/tpg1/luns
/iscsi/iqn.20...ver/tpg1/luns> create /backstores/block/iscsi-client-vol1
Created LUN 0.
/iscsi/iqn.20...ver/tpg1/luns> cd ../acls
/iscsi/iqn.20...ver/tpg1/acls> create iqn.2021-07.com.ricsanfre.vbox:iscsi-client
Created Node ACL for iqn.2021-07.com.ricsanfre.vbox:iscsi-client
Created mapped LUN 0.
/iscsi/iqn.20...ver/tpg1/acls> cd iqn.2021-07.com.ricsanfre.vbox:iscsi-client/
/iscsi/iqn.20...:iscsi-client> set auth userid=user1
Parameter userid is now 'user1'.
/iscsi/iqn.20...:iscsi-client> set auth password=s1cret0
Parameter password is now 's1cret0'.
/iscsi/iqn.20...:iscsi-client> exit
Global pref auto_save_on_exit=true
Last 10 configs saved in /etc/rtslib-fb-target/backup/.
Configuration saved to /etc/rtslib-fb-target/saveconfig.json
```

### Step 8. Assing mapped LUNs to initiators

    cd /iscsi/<target_iqn>/tpg1/acls/<initiator_iqn>
    create mapped_lun=<mapped_lunid> tpg_lun_or_backstore=<lunid> write_protect=<0/1>

write_protect=1, means read-only lun

### Step 9. Save config

Upon exiting targetcli configuration is saved automatically.
If configuration has been executed through command line without entering targetcli shell (i.e: sudo targetcli command), changes need to be saved

    sudo targetcli saveconfig

### Step 10. Load configuration on startup

    sudo systemctl enable rtslib-fb-targetctl 

### Step 11. Configure firewall rules

Enable incoming traffic on port TCP 3260.

## Configuring iSCSI Initiator

### Step 1. Ensure package open-iscsi is installed
In order to communicate and connect to iSCSI volume, we need to install open-iscsi package.

    sudo apt install open-iscsi

### Step 2. Configure iSCI Intitiatio iqn 

Edit iqn assigned to the server in the file `/etc/iscsi/initiatorname.conf`.

```
InitiatorName=iqn.2021-07.com.ricsanfre.picluster:<host_name>
```
### Step 3. Configure iSCSI Authentication

Edit file `/etc/iscsi/iscsid.conf`

Unncomment and add the proper values to the following entries:

```
node.session.auth.authmethod = CHAP
node.session.auth.username = user1
node.session.auth.password = s1cret0
```

### Step 4. Restart open-iscsi service

    sudo systemctl restart iscsid
    sudo systemclt enable iscsid

### Step 5. Connect to iSCSI Target

Discover the iSCSI Target.


    sudo iscsiadm -m discovery -t sendtargets -p 192.168.100.100

```
sudo iscsiadm -m discovery -t sendtargets -p 192.168.56.100
192.168.56.100:3260,1 iqn.2021-07.com.ricsanfre.vbox:iscsi-server
```

Login to the iSCSI target

    sudo iscsiadm --mode node --targetname <iqn-target> --portal <iscsi-server-ip> --login

```
sudo iscsiadm --mode node --targetname iqn.2021-07.com.ricsanfre.vbox:iscsi-server --portal 192.168.56.100 --login
Logging in to [iface: default, target: iqn.2021-07.com.ricsanfre.vbox:iscsi-server, portal: 192.168.56.100,3260](multiple)
Login to [iface: default, target: iqn.2021-07.com.ricsanfre.vbox:iscsi-server, portal: 192.168.56.100,3260] successful.
```

Check the discovered iSCSI disks

    sudo iscsiadm -m session -P 3


```
sudo iscsiadm -m session -P 3
iSCSI Transport Class version 2.0-870
version 2.0-874
Target: iqn.2021-07.com.ricsanfre:iscsi-target (non-flash)
        Current Portal: 192.168.0.11:3260,1
        Persistent Portal: 192.168.0.11:3260,1
                **********
                Interface:
                **********
                Iface Name: default
                Iface Transport: tcp
                Iface Initiatorname: iqn.2021-07.com.ricsanfre:iscsi-initiator
                Iface IPaddress: 192.168.0.12
                Iface HWaddress: <empty>
                Iface Netdev: <empty>
                SID: 1
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
                username: iqn.2021-07.com.ricsanfre:iscsi-initiator
                password: ********
                username_in: iqn.2021-07.com.ricsanfre:iscsi-target
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
                Host Number: 2  State: running
                scsi2 Channel 00 Id 0 Lun: 0
                        Attached scsi disk sdb          State: running
                scsi2 Channel 00 Id 0 Lun: 1
                        Attached scsi disk sdc          State: running

```

At the end of the ouput the iSCSI attached disks can be found (sdb and sdc)


Command `lsblk` can be used to list SCSI devices

    sudo lsblk -S

```
lsblk -S
NAME HCTL       TYPE VENDOR   MODEL      REV TRAN
sda  2:0:0:0    disk ATA      VBOX_HARDDISK 1.0  sata
sda  2:0:0:0    disk LIO-ORG  lun_node1 4.0  iscsi
sdb  2:0:0:1    disk LIO-ORG  lun_node3 4.0  iscsi
```

> NOTE: TRANS column let us separate iSCSI disk (trans=iscsi) from local SCSI (TRANS=sata)
> MODEL column shows LUN name configured in the target



Check the connected iSCSI disks with command `fdisk -l`:

```
sudo fdisk -l
....
    
Device      Start     End Sectors  Size Type
/dev/sda1  227328 4612062 4384735  2,1G Linux filesystem
/dev/sda14   2048   10239    8192    4M BIOS boot
/dev/sda15  10240  227327  217088  106M EFI System

Partition table entries are not in disk order.


Disk /dev/sdb: 4 GiB, 4294967296 bytes, 8388608 sectors
Disk model: iscsi-client-vo
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 33550336 bytes

```

### Step 6. Configure automatic login 

    sudo iscsiadm --mode node --op=update -n node.conn[0].startup -v automatic
    sudo iscsiadm --mode node --op=update -n node.startup -v automatic


### Step 7. Format and mount iSCSI disk

The new iSCSI disk can be partitioned with `fdisk`/`parted` and formated with `mkfs.ext4` and mount as any other disk

Also it can be used with LVM as a physical volume for createing Logical Volumes

- Create a physical volume

    sudo pvcreate /dev/sdb

- Create a volume group

    sudo vgcreate vgcreate vg_iscsi /dev/sdb

- Create Logical Volume

    sudo lvcreate vg_iscsi -l 100%FREE -n lv_iscsi

- Format Logical Volume

    sudo mkfs.ext4 /dev/vg_iscsi/lv_iscsi

- Mount the disk

    sudo mkdir /data
    sudo mount /dev/vg_iscsi/lv_iscsi /data


### Step 8. Mount iSCSI disk on startup

Modify `/etc/fstab` to mount iSCSI disk on startup

First find the volume UUID.

    sudo blkid


```
sudo blkid

...
/dev/sdb: UUID="xj6V9b-8uo6-RACn-MTqB-7siH-nvjT-Aw9B0V" TYPE="LVM2_member"
/dev/mapper/vg_iscsi-lv_iscsi: UUID="247a2c91-4af8-4403-ac5b-a99116dac96c" TYPE="ext4"
```


Add the following line to `/etc/fstab`
```
/dev/disk/by-uuid/[iSCSI_DISK_UUID] [MOUNT_POINT] ext4 _netdev 0 0
```

```
#iSCSI data
UUID=247a2c91-4af8-4403-ac5b-a99116dac96c /data            ext4    _netdev    0  0
```


> Important is to specify _netdev option (mount filesystem after network boot is completed)




# Preparing SSD disk for gateway node

`gateway` node will be acting as SAN server. It will boot from USB using a SSD disk.

SSD Disk will be partitioned to install the operating system and for the iSCSI LUNS: 32 GB will be reserved for the OS and the rest of the disk will be used for LUNs.
Initial partitions (boot and OS) will be created during initial image burning process. Partitions need to be reconfigured before the first boot.

## Step 1. Burn Ubuntu 20.04 server to SSD disk using Balena Etcher

## Step 2. Boot Raspberry PI with Raspberry OS

### Step 3. Connect SSD Disk to USB 3.0 port.

Check the disk

   sudo fdisk -l

```
sudo fdisk -l

Disk /dev/mmcblk0: 29.7 GiB, 31914983424 bytes, 62333952 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x9c46674d

Device         Boot  Start      End  Sectors  Size Id Type
/dev/mmcblk0p1        8192   532479   524288  256M  c W95 FAT32 (LBA)
/dev/mmcblk0p2      532480 62333951 61801472 29.5G 83 Linux


Disk /dev/sda: 447.1 GiB, 480103981056 bytes, 937703088 sectors
Disk model: Generic
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: dos
Disk identifier: 0x4ec8ea53

Device     Boot  Start     End Sectors  Size Id Type
/dev/sda1  *      2048  526335  524288  256M  c W95 FAT32 (LBA)
/dev/sda2       526336 6366175 5839840  2.8G 83 Linux
```


## Step 4. Repartition with parted

After flashing the disk the root partion size is less than 3 GB. On first boot this partition is automatically extended to occupy 100% of the available disk space.
Since I want to use the SSD disk not only for the Ubuntu OS, but providing iSCSI LUNS. Before the first boot, I will repartition the SSD disk.

- Extending the root partition to 32 GB Size
- Create a new partition for storing iSCSI LVM LUNS


```
pi@gateway:~ $ sudo parted /dev/sda
GNU Parted 3.2
Using /dev/sda
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) print
Model: JMicron Generic (scsi)
Disk /dev/sda: 480GB
Sector size (logical/physical): 512B/4096B
Partition Table: msdos
Disk Flags:

Number  Start   End     Size    Type     File system  Flags
 1      1049kB  269MB   268MB   primary  fat32        boot, lba
 2      269MB   3259MB  2990MB  primary  ext4

(parted) resizepart
Partition number? 2
End?  [3259MB]? 32500
(parted) print
Model: JMicron Generic (scsi)
Disk /dev/sda: 480GB
Sector size (logical/physical): 512B/4096B
Partition Table: msdos
Disk Flags:

Number  Start   End     Size    Type     File system  Flags
 1      1049kB  269MB   268MB   primary  fat32        boot, lba
 2      269MB   32.5GB  32.2GB  primary  ext4

(parted) mkpart
Partition type?  primary/extended? primary
File system type?  [ext2]? ext4
Start? 32501
End?
End? 100%
(parted) print
Model: JMicron Generic (scsi)
Disk /dev/sda: 480GB
Sector size (logical/physical): 512B/4096B
Partition Table: msdos
Disk Flags:

Number  Start   End     Size    Type     File system  Flags
 1      1049kB  269MB   268MB   primary  fat32        boot, lba
 2      269MB   32.5GB  32.2GB  primary  ext4
 3      32.5GB  480GB   448GB   primary  ext4         lba

(parted) set 3 lvm on
(parted) print
Model: JMicron Generic (scsi)
Disk /dev/sda: 480GB
Sector size (logical/physical): 512B/4096B
Partition Table: msdos
Disk Flags:

Number  Start   End     Size    Type     File system  Flags
 1      1049kB  269MB   268MB   primary  fat32        boot, lba
 2      269MB   32.5GB  32.2GB  primary  ext4
 3      32.5GB  480GB   448GB   primary  ext4         lvm, lba

(parted) quit
```

## Step 5. Checking USB-SATA Adapter

Checking that the USB SATA adapter suppors UASP.

```
lsusb -t

/:  Bus 02.Port 1: Dev 1, Class=root_hub, Driver=xhci_hcd/4p, 5000M
    |__ Port 1: Dev 2, If 0, Class=Mass Storage, Driver=uas, 5000M
/:  Bus 01.Port 1: Dev 1, Class=root_hub, Driver=xhci_hcd/1p, 480M
    |__ Port 1: Dev 2, If 0, Class=Hub, Driver=hub/4p, 480M
        |__ Port 3: Dev 3, If 0, Class=Human Interface Device, Driver=usbhid, 1.5M
        |__ Port 3: Dev 3, If 1, Class=Human Interface Device, Driver=usbhid, 1.5M
        |__ Port 4: Dev 4, If 0, Class=Human Interface Device, Driver=usbhid, 1.5M

```
> Driver=uas indicates that the adpater supports UASP


Check USB-SATA adapter ID

```
sudo lsusb
Bus 002 Device 002: ID 174c:55aa ASMedia Technology Inc. Name: ASM1051E SATA 6Gb/s bridge, ASM1053E SATA 6Gb/s bridge, ASM1153 SATA 3Gb/s bridge, ASM1153E SATA 6Gb/s bridge
Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 001 Device 004: ID 0000:3825
Bus 001 Device 003: ID 145f:02c9 Trust
Bus 001 Device 002: ID 2109:3431 VIA Labs, Inc. Hub
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```

> NOTE: In this case ASMedia TEchnology ASM1051E has ID 152d:0578


## Step 6. Modify USB partitions following instrucions described [here](./installing_ubuntu.md)