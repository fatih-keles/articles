![images](/resources/Transformation_1.png)

# Is it getting easier? I resized my boot volume in 15 minutes!

I have a Linux virtual machine (one of many) on our cloud environment that I use for Docker experiments. I try to keep it small and clean. Inevitably at some point I couldn't fit into the initial disk space. I needed to increase storage capacity. I had options like attaching another block volume but I wanted a spacious root partition so decided to resize my boot volume.

I used to work with on premises systems most of the time in the past and to be honest so far I have asked for more resources from sysadmins countless times but I have never done this on my own. Until now! Sysadmins took care of everything within their resources. They gave it back when it is ready to use. I had no idea of what was going on and how long it takes to do such thing. Whenever I needed something I asked from sysadmins. I can say running on cloud is no different than managing your own datacenter. Except from everything is ready, almost impossible to make a mistake (unless you ignore the signs), best of the breed technology, unlimited resources (if you are willing to pay). I must say! Grow, scale, pretty easy, pretty impressive.

6 steps to achieve this:

1. I stopped the VM instance, created a clone of boot volume. Then detached it from the server and resized it to double the capacity.

2. I attached boot volume as an iSCSI Block Volume to another VM. At this point Oracle Cloud warns us to execute OS level iSCSI commands in order to extend the capacity and be able to use it.

![images](/resources/attach-block-volume.png)
![images](/resources/attach-block-volume-warning.png)

3. I found the necessary iSCSI commands in 3 dots menu, the commands are ready to run, configured according to my storage instance.

![images](/resources/iscsi-commands.png)

4. I ssh'ed to guest OS and executed the commands below to register the new volume, configure it to survive after a reboot (I guess it was not necessary in this case), and login to storage.

*I have changed the real IP address of storage with {IP Address} in the following scripts.*

```console
(base) [root@fkeles-scrapyd-server ~]# sudo iscsiadm -m node -o new -T iqn.2015-02.oracle.boot:uefi -p {IP Address}:3260
New iSCSI node [tcp:[hw=,ip=,net_if=,iscsi_if=default] {IP Address},3260,-1 iqn.2015-02.oracle.boot:uefi] added

(base) [root@fkeles-scrapyd-server ~]# sudo iscsiadm -m node -o update -T iqn.2015-02.oracle.boot:uefi -n node.startup -v automatic

(base) [root@fkeles-scrapyd-server ~]# sudo iscsiadm -m node -T iqn.2015-02.oracle.boot:uefi -p {IP Address}:3260 -l
Logging in to [iface: default, target: iqn.2015-02.oracle.boot:uefi, portal: {IP Address},3260] (multiple)
Login to [iface: default, target: iqn.2015-02.oracle.boot:uefi, portal: {IP Address},3260] successful.

(base) [root@fkeles-scrapyd-server ~]# fdisk -l
Disk /dev/sdb: 105.2 GB, 105226698752 bytes, 205520896 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 1048576 bytes
Disk label type: dos
Disk identifier: 0x00000000
 
   Device Boot      Start         End      Blocks   Id  System
/dev/sdb1               1    97677311    48838655+  ee  GPT
Partition 1 does not start on physical sector boundary.
```

5. Then I had to extend the partition and grow the file system. To be honest this part is the most complex piece of work. I am not a Linux guru, I found the documentation extremely helpful. 

```console
(base) [root@fkeles-scrapyd-server ~]# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sdb      8:16   0   98G  0 disk
├─sdb2   8:18   0    8G  0 part
├─sdb3   8:19   0 38.4G  0 part
└─sdb1   8:17   0  200M  0 part
sda      8:0    0 46.6G  0 disk
├─sda2   8:2    0    8G  0 part [SWAP]
├─sda3   8:3    0 38.4G  0 part /
└─sda1   8:1    0  200M  0 part /boot/efi

(base) [root@fkeles-scrapyd-server ~]# parted /dev/sdb
GNU Parted 3.1
Using /dev/sdb
Welcome to GNU Parted! Type 'help' to view a list of commands.

(parted) unit s

(parted) print
Error: The backup GPT table is not at the end of the disk, as it should be.
This might mean that another operating system believes the disk is smaller.
Fix, by moving the backup to the end (and removing the old backup)?
Fix/Ignore/Cancel? Fix

Warning: Not all of the space available to /dev/sdb appears to be used, you can
fix the GPT to use all of the space (an extra 107843584 blocks) or continue with
the current setting?
Fix/Ignore? Fix

Model: ORACLE BlockVolume (scsi)
Disk /dev/sdb: 205520896s
Sector size (logical/physical): 512B/4096B
Partition Table: gpt
Disk Flags:
Number  Start      End        Size       File system     Name                 Flags
 1      2048s      411647s    409600s    fat16           EFI System Partition  boot
 2      411648s    17188863s  16777216s  linux-swap(v1)
 3      17188864s  97675263s  80486400s  xfs

(parted) rm 3

(parted) mkpart
Partition name?  []?
File system type?  [ext2]? xfs
Start? 17188864s
End? 100%

(parted) quit
Information: You may need to update /etc/fstab.

(base) [root@fkeles-scrapyd-server ~]# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sdb      8:16   0   98G  0 disk
├─sdb2   8:18   0    8G  0 part
├─sdb3   8:19   0 89.8G  0 part
└─sdb1   8:17   0  200M  0 part
sda      8:0    0 46.6G  0 disk
├─sda2   8:2    0    8G  0 part [SWAP]
├─sda3   8:3    0 38.4G  0 part /
└─sda1   8:1    0  200M  0 part /boot/efi
```

I could see that capacity is doubled in *lsblk* command output. Now I had to handle the file system.

```console
(base) [root@fkeles-scrapyd-server ~]# xfs_repair /dev/sdb3
Phase 1 - find and verify superblock...
Phase 2 - using internal log
        - zero log...
        - scan filesystem freespace and inode maps...
        - found root inode chunk
Phase 3 - for each AG...
        - scan and clear agi unlinked lists...
        - process known inodes and perform inode discovery...
        - agno = 0
        - agno = 1
        - agno = 2
        - agno = 3
        - process newly discovered inodes...
Phase 4 - check for duplicate blocks...
        - setting up duplicate extent list...
        - check for inodes claiming duplicate blocks...
        - agno = 0
        - agno = 1
        - agno = 2
        - agno = 3
Phase 5 - rebuild AG headers and trees...
        - reset superblock...
Phase 6 - check inode connectivity...
        - resetting contents of realtime bitmap and summary inodes
        - traversing filesystem ...
        - traversal finished ...
        - moving disconnected inodes to lost+found ...
Phase 7 - verify and correct link counts...

Done

(base) [root@fkeles-scrapyd-server ~]# df -kh
Filesystem      Size  Used Avail Use% Mounted on
devtmpfs        7.2G     0  7.2G   0% /dev
tmpfs           7.3G   68M  7.2G   1% /dev/shm
tmpfs           7.3G  578M  6.7G   8% /run
tmpfs           7.3G     0  7.3G   0% /sys/fs/cgroup
/dev/sda3        39G   14G   26G  35% /
/dev/sda1       200M  9.8M  190M   5% /boot/efi

tmpfs           1.5G     0  1.5G   0% /run/user/1000

(base) [root@fkeles-scrapyd-server ~]# mkdir /mnt/tmp_bv

(base) [root@fkeles-scrapyd-server ~]# mount /dev/sdb3 /mnt/tmp_bv/ -o nouuid

(base) [root@fkeles-scrapyd-server ~]# df -kh
Filesystem      Size  Used Avail Use% Mounted on
devtmpfs        7.2G     0  7.2G   0% /dev
tmpfs           7.3G   68M  7.2G   1% /dev/shm
tmpfs           7.3G  578M  6.7G   8% /run
tmpfs           7.3G     0  7.3G   0% /sys/fs/cgroup
/dev/sda3        39G   14G   26G  35% /
/dev/sda1       200M  9.8M  190M   5% /boot/efi
tmpfs           1.5G     0  1.5G   0% /run/user/1000

/dev/sdb3        39G  9.7G   29G  26% /mnt/tmp_bv
```

New volume mounted to /mnt/tpm_bv and is displaying as *29G*.

```console
(base) [root@fkeles-scrapyd-server ~]# xfs_growfs -d /mnt/tmp_bv/
meta-data=/dev/sdb3              isize=256    agcount=4, agsize=2515200 blks
         =                       sectsz=4096  attr=2, projid32bit=1
         =                       crc=0        finobt=0 spinodes=0 rmapbt=0
         =                       reflink=0
data     =                       bsize=4096   blocks=10060800, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal               bsize=4096   blocks=4912, version=2
         =                       sectsz=4096  sunit=1 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 10060800 to 23541248


(base) [root@fkeles-scrapyd-server ~]# df -kh
Filesystem      Size  Used Avail Use% Mounted on
devtmpfs        7.2G     0  7.2G   0% /dev
tmpfs           7.3G   68M  7.2G   1% /dev/shm
tmpfs           7.3G  578M  6.7G   8% /run
tmpfs           7.3G     0  7.3G   0% /sys/fs/cgroup
/dev/sda3        39G   14G   26G  35% /
/dev/sda1       200M  9.8M  190M   5% /boot/efi
tmpfs           1.5G     0  1.5G   0% /run/user/1000

/dev/sdb3        90G  9.7G   81G  11% /mnt/tmp_bv
```
I saw new volume is fully resized and usable (*81G*).

```console
(base) [root@fkeles-scrapyd-server ~]# umount /dev/sdb3

(base) [root@fkeles-scrapyd-server ~]# sudo iscsiadm -m node -T iqn.2015-02.oracle.boot:uefi -p {IP Address}:3260 -u
Logging out of session [sid: 2, target: iqn.2015-02.oracle.boot:uefi, portal: {IP Address},3260]
Logout of [sid: 2, target: iqn.2015-02.oracle.boot:uefi, portal: {IP Address},3260] successful.


(base) [root@fkeles-scrapyd-server ~]# sudo iscsiadm -m node -o delete -T iqn.2015-02.oracle.boot:uefi -p {IP Address}:3260
```

I unmounted the volume, logged out from the storage in order to safely detach it from host server.

6. And finally I detached the volume from host server and attached it back to my virtual machine. Pretty easy, with a few mouse clicks and executing some commands I was able to extend my root partition of my Linux server. Impressive.