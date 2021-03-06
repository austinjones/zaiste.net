+++
title = "How To Create a Torrent Box on Raspberry Pi"
+++

```bash
sudo apt-get install transmission-daemon
```

Mount the network drive

```bash
sudo mkdir -p /media/NASHDD1/torrent-inprogress
sudo mkdir -p /media/NASHDD1/torrent-complete
```

```bash
sudo nano /etc/transmission-daemon/settings.json
```

```bash
"incomplete-dir": "/media/NASHDD1/torrent-inprogress",
"incomplete-dir-enabled": true,
"download-dir": "/media/NASHDD1/torrent_complete",
```

```bash
sudo service transmission-daemon reload
```

rpc-password and rpc-username located respectively on Lines 51 and 54 of the configuration file. Using them, we can set the "login" data that will be used to access the transmission web interface: by default the value of both is "transmission". The value we see on rpc-password in the configuration file is the result of the hashing of the plain text password: we insert our password in the field, and it will be automatically hashed once the daemon starts.

Speaking of ports, the default transmission peer-port is 51413, as defined on Line 32. Opening this port on the firewall (and allowing port forwarding in the router) is not strictly necessary for the applications to work correctly, however it is needed for it to work in active mode, and so to be able to connect to more peers.

```bash
sudo ufw allow 9091,51413/tcp
```

# How to Install ZFS File System on Ubuntu 18.04

ZFS is a combined file system and logical volume manager that is scalable and includes features such as protection against data corruption. It also offers high storage capacity, allows for snapshots and copy-on-write clones.

ZFS is a combined file system and logical volume manager originally designed and implemented by a team at Sun Microsystems led by Jeff Bonwick and Matthew Ahrens. Features of ZFS include protection against data corruption, high storage capacity (256 ZiB), snapshots and copy-on-write clones and continuous integrity checking to name but a few.

ZFS can self heal and recover data automatically. Complex algorithms, hashes and Merkle trees guarantee data integrity.

## Install ZFS

```bash
sudo apt-get install zfsutils-linux
```

Striped pool, where a copy of data is stored across all drives.

```bash
sudo zpool create files /dev/vda /dev/vdb
```

Mirrored pool, where a single, complete copy of data is stored on all drives.

```bash
sudo zpool create files mirror /dev/vda /dev/vdb
```

## List ZFS Pool

```bash
sudo zpool list
```

## Write to the pool

By default, /files mount point is only writeable by the user root. If you want to make /files writable by your own user and group, you can do so by running the following command:

```bash
sudo chown -Rfv USERNAME:GROUPNAME /files
```

## Different mount point

mount files ZFS pool to /var/www

```bash
sudo zfs set mountpoint=/var/www files
```

## Remove a Pool

the pool, you can remove it. Beware that this will also remove any files that you have created within the pool!

## ZFS Snapshot

A snapshot is simply an exact picture of the state of your data at a certain point in time.

```bash
sudo zfs snapshot data@snap1
```

where ```
undefined
``` is the name of the pool

```bash
sudo zfs rollback data@snap1
```

When you take a snapshot of a given dataset (‘dataset’ is the ZFS term for a filesystem), ZFS just records the timestamp when the snapshot was made. That is it! No data is copied and no extra storage is consumed. It works because of the copy-on-write mechanism.

```bash
zfs list -rt all zroot/usr/src
```

## ZFS Clone

While snapshots are basically frozen data states that you can return to, clones are like branches that start from a common point.

## Copy-on-write

The copy-on-write mechanism ensures that whenever a block is modified, instead of modifying the block directly, it makes a copy of the block with the required modifications done on the new block.

## Add Disk to the Pool

```bash
zpool add files ada4
```

## Virtual Devices (VDev)

Vdevs are the building blocks of a zpool, most of the redundancy and performance depends on the way in which your disks are grouped into vdevs.

### RAID 0 (Stripes)

Each disk acts as its own vdev. No data redundancy, and the data spread across all the disks. Also known as striping. A single disk’s failure would mean that the entire zpool is rendered unusable. Usable storage is equal to the sum of all available storage devices.

### RAID 1 (Mirror)

Data is mirrored between ndisks. The actual capacity of the vdev is limited by the raw capacity of the smallest disk in that n-disk array. Data is mirrored between n disks, this means that you can withstand the failure of n-1 disks.

You may want to add extra disk, say ada4, to mirror same the data.

```bash
zpool attach tank ada1 ada4
```

add an extra vdev to increase the capacity of zpool.

```bash
zpool add tank mirror ada4 ada5 ada6
```

### RAID Z1

must use at least 3 disks and the vdev can tolerate the demise of one only of those disks.

RAID-Z configurations don’t allow attaching disks directly onto a vdev. But you can add more vdevs, using zpool add, such that the pool’s capacity can keep on increasing.

RAID-Z is a data/parity distribution scheme like RAID-5, but it uses a dynamic stripe width: every block has it's own RAID stripe, regardless of the block size, resulting in every RAID-Z write being a full-stripe write.

## ZFS Send / Receive

Using the zfs send and zfs receive we can perform full and incremental backup of a dataset and we can also use output redirection to store the backup in gzip format, a tape, a disk or in a remote file system.

```bash
zfs send datapool/project1@monday | gzip -9 > projet1.gz
```

```bash
zfs send datapool/project1@monday | ssh sol02 zfs recv testpool/project1
```

### Full Replication

```bash
zfs send datapool/project1@monday | zfs receive -Fv rpool/project1

```

### Incremental

```bash
zfs send -I datapool/project1@monday datapool/project1@tuesdayIncr | zfs receive -Fv rpool/project1
```

