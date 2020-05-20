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
