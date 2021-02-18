# qnap_tricks
A collection of documents to help me understand QNAP internals.

## Introduction

I bought a TS-653A in July of 2017. By 2021 it is obsolete and out of warranty. 

I would not recommend that anyone play with their in-warranty QNAP, however I have considered my purchase to be a giant let-down from day 1, with my future plans being a NAS build to my specs rather than another chassis system.

## /etc/enclosure_*n*.conf

This appears to be where the Physical enclosure disks are defined.

   * In my system, two enclosures exist. 
   * Enclosure 0 is the physical drive enclosure, which can hold up to 6 disks.
   * Enclosure 33 is a generic enclosure ID for all external disks connected to the USB3 ports.

## /etc/external_vol.conf

This is where external USB disks are defined.

## /etc/volume.conf

This is where all of the known disk volumes are tracked. 

### Internal Logical (LV) Disk Example

```
[VOL_6]
volId = 6
volName = lvmlv
raidId = 1
raidName = /dev/md1
encryption = no
ssdCache = no
unclean = no
need_rehash = no
creating = no
mappingName = /dev/mapper/cachedev6
qnapResize = no
delayAlloc = yes
privilege = no
readOnly = no
writeCache = yes
invisible = no
raidLevel = 5
partNo = 3
status = 0
filesystem = 9
internal = 1
time = Tue May 14 19:38:11 2019
volType = 1
baseId = 6
baseName = /dev/mapper/cachedev6
inodeRatio = 16384
inodeCount = 327680
fsFeature = 0x3
```

### External USB Disk Example

```
[VOL_21]
volId = 21
volName = externalusb
raidId = -1
raidName = /dev/sdk1
encryption = no
mappingName = /dev/sdk1
qnapResize = no
delayAlloc = yes
privilege = no
readOnly = no
writeCache = yes
invisible = no
raidLevel = -2
partNo = 1
isGlobalSpare = no
diskId = 0x00210005
status = 0
filesystem = 9
internal = 0
time = Tue Jan 28 22:15:17 2020
unclean = no
need_rehash = no
creating = no
```

## How things work

### Caching

SSD Cache is implemented using facebook's ```flashcache```.

```dm-cache``` is used to mount the devices via a cachedev device, which allows caching to be turned on or off for individual Logical Volumes as required.

#### Cache Flush

For reasons I don't fully understand, the flush of dirty blocks to disk does not seem to work the way you would expect in Read/Write cache mode, and as a result what I believe is unnecessary risk is added to the caching mechanism. This is because dirty page evection **only** seems to occur during cache full events, despite the following settings:

```
dev.flashcache.CG0.fallow_clean_speed = 2
dev.flashcache.CG0.fallow_delay = 60
dev.flashcache.CG0.reclaim_policy = 1
```

These settings should, as I understand it, clean out dirty blocks using the LRU (least recent used) algorithm after they have not been written to for 1 minute. That said, other controls are available to control the caching behaviour, and I have leveraged these settings in my environment. In the past, I was manually flushing cache blocks to disk by disabling all caching for the NAS, which has a significant effect on performance as it will then direct all reads and writes to the same backing array that is being flushed to - causing significant IO bottlenecks.

Somewhat of a giveaway that the 
```
fallow_cleanings:       0
```

The first parameter to look at tuning is the percentage of dirty blocks (taken as a percentage of the overall SSD cache size) that may exist on the SSD cache before LRU dirty cache blocks are flushed to disk. By default this is set to:

```
dev.flashcache.CG0.dirty_low_thresh_pct = 20
dev.flashcache.CG0.dirty_thresh_pct = 60
```

Wow! That's a lot. So the default configuration is to allow dirty write blocks to consume up to 60% of the SSD cache. Breaking that down against my objectives we can see that it is not an optimal configuration for me:

   * I have a 1TB SSD cache sitting in front of a RAID 6 array of magnetic NAS disks
   * The RAID 6 benefits me in read speed but not in write speed, however the capacity of the magnetic disks (24TB in RAID) eclipse the SSD cache (1TB) and stic o I intend to leverage the advantages of each (SSD IOPS and Magnetic Capacity) to do more with my disk array.
   * What I would like the SSD Cache to do is to intercept random reads and writes, and attempt to service them from the SSD. If the read block is cached, it avoids a read against the backing array which incurs latency. If it is a write, I want to buffer that write for a short period of time in case it is again written to before being committed to disk. This increases IOPS for applications such as databases with indexes which may see repeated updates in a short period of time.
   * What I **don't** wan't an SSD cache doing is storing lots of dirty blocks that should be written to a more resilient backing store (in this case, a RAID 6 dual-parity array that can withstand 2 disk failures where my SSD cache cannot).
   
So I set the following parameters for what I'd accept as success criteria for the dirty block cache settings:

   * Target as low as possible a dirty cache limit parameter (so that I can easily regularly flush these blocks to disk without significant performance penalty)
   * Ensure that the write cache size is enough to handle sustained write activity by monitoring the write hit rate
   
Clearly, I don't need 600GB of space to buffer writes for this purpose. The write hit rate averaged 90% prior to reducing the write cache 

On my system, I was able to tune flashcache as low as the following parameters, which in turn improved my average read hit rate, assumedly because there is now more capacity for read caching on the cache disk, while remaining on average >90% for write cache hits.

```
sysctl -w dev.flashcache.CG0.dirty_low_thresh_pct=4
sysctl -w dev.flashcache.CG0.dirty_thresh_pct=8
```

Manual flush of the SSD write cache

```
sysctl -w dev.flashcache.CG0.do_sync=1
```

### Entware

Installing the Entware qpkg allows for additional packages to be installed from the Entware repository using opkg;.

Some useful packages:

```
opkg install smartd smartmontools smartmontools-drivedb
opkg install zabbix-agentd
```

### Mounting

#### Internal Storage Volumes

All drives are mounted to a location (the backing mount) such as /share/CACHEDEV7_DATA, as well as to a share directory if a share is created. 

The mount command for both the CACHEDEV mount and the share mounts are the same:

```
/dev/mapper/cachedev10 /share/CACHEDEV10_DATA ext4 rw,relatime,(null),noacl,stripe=256,data=ordered,jqfmt=vfsv0,usrjquota=aquota.user 0 0
/dev/mapper/cachedev10 /share/NFSv=4/sharename ext4 rw,relatime,(null),noacl,stripe=256,data=ordered,jqfmt=vfsv0,usrjquota=aquota.user 0 0
```

However there isn't a 1 to 1 relationship between the CACHEDEV mount and the NFS mounts. Firstly, there can be more than one NFS mount if you have multiple mounts pointing to the same volume, and secondly NFS mounts are **only** created if you are exporting that filesystem via NFS.

Samba shares use the CACHEDEV*n*_DATA mount to share filesystems.

You can determine the mapping of cachedev devices to volumes in the ```/etc/volume.conf``` file.

#### External Storage Volumes

External drives are different. They are mounted first to a mountpoint under /share/external/DEV*xxxx*, and then mounted to an NFS mount if NFS shares are enabled for the external volume.

The numbering appears to start from DEV3301 and increment from there for each disk inserted on my NAS, however others have mounted from 3500, 3600 or even just mounted the device name itself (/share/external/sdk1).

   * The device name appears to be related to the output of the ```qcli_storage -p``` command, where the WWN detection section of the output shows the RAID Expansion device to be ```REXP#33``` and then lists the drive IDs next to this.
   * This also aligns with the enclosure_33.conf file which has all of the external disks ever connected to the NAS defined.

If you need to manually mount an external drive to both an external mountpoint and an NFS mountpoint, you can use the following commands:

```
mount -t ext4 -o rw,noacl,data=ordered,jqfmt=vfsv0,usrjquota=aquota.user /dev/sdf2 /share/external/DEV3306_2 
mount -t ext4 -o rw,noacl,data=ordered,jqfmt=vfsv0,usrjquota=aquota.user /dev/sdf2 /share/NFSv=4/sharename 
```

   * Can external volumes be LVM Physical Volumes?
      * The LVM configuration file on the qnap allows only md or drbd devices to be used as LVM physical volumes.
   * What happens if an external disk is part of a RAID array?
      * TBA

### RAID

Every internal disk (whether it be a data or SSD cache disk) in the chassis participates in two RAID 1 arrays (md9 and md13). 

   * md1 is the first configured RAID array containing the storage for the data volumes.
   * md2 was created when adding an SSD cache to the NAS
   * md9 is mounted at ```/mnt/HDA_ROOT``` and contains the configuration and root filesystem for the NAS. ```/etc/config```, which contains a lot of the NAS configuration files, symlinks to a directory under here.
   * md13 is mounted at ```/mnt/ext``` and is used for the web interface and a lot of external packages such as mariadb, python, samba and so on. This appears to be where apps are installed when you install these from the app center.
   * md256 is a swap device, as is md321 and md322.
   * md321 is a swap device on the SSD cache drive, and is apparently only added in some cases, where it is needed
   * md322 is a swap device. These arrays are raid1 spanned across all data disks, but not SSD cache disks.

## Various tools

| Command                 | Arguments   | Purpose                                             |
| ----------------------- | ----------- | --------------------------------------------------- | 
| /sbin/get_hd_smartinfo  | -d [Disk #] | Prints the SMART details for the disk in this slot. | 
| /sbin/getcfg            | TBA         | TBA |
| /sbin/qcli_storage      | -p          | Shows a table of disks attached to the NAS and their status |
| /sbin/setcfg            | TBA         | TBA |
