# qnap_tricks
A collection of documents to help me understand QNAP internals.

## Introduction

I bought a TS-653A in July of 2017. By 2020 it is obsolete and out of warranty. 

I would not recommend that anyone play with their in-warranty QNAP, however I have considered my purchase to be a giant let-down from day 1, with my future plans being a NAS build to my specs rather than another chassis system.

## /etc/enclosure_x.conf

This appears to be where the Physical enclosure disks are defined.

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

### Entware

Installing the Entware qpkg allows for additional packages to be installed from the Entware repository using opkg;.

Some useful packages:

```
opkg install smartmontools
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

If you need to manually mount an external drive to both an external mountpoint and an NFS mountpoint, you can use the following commands:

```
mount -t ext4 -o rw,noacl,data=ordered,jqfmt=vfsv0,usrjquota=aquota.user /dev/sdf2 /share/external/DEV3306_2 
mount -t ext4 -o rw,noacl,data=ordered,jqfmt=vfsv0,usrjquota=aquota.user /dev/sdf2 /share/NFSv=4/sharename 
```

## Various tools

| Command                 | Arguments   | Purpose                                             |
| ----------------------- | ----------- | --------------------------------------------------- | 
| /sbin/get_hd_smartinfo  | -d [Disk #] | Prints the SMART details for the disk in this slot. | 
| /sbin/getcfg            | TBA         | TBA |
| /sbin/setcfg            | TBA         | TBA |
