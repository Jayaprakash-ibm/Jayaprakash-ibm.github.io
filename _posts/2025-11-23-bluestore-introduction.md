---
title: "BlueStore Introduction"
date: 2025-11-23 00:10:00 +0530
categories: [BlueStore, Ceph]
tags: [bluestore, ceph, storage, architecture, introduction]
image: /assets/img/bs-intro.webp
alt: "BlueStore Introduction"
---

### Introduction

BlueStore is a backend object store for the Ceph OSD daemons.
The original object store, FileStore, requires a file system on top of raw block devices.
Objects are then written to the file system. Unlike the original FileStore back end, BlueStore stores object directly on the block devices without any file system interface, which improves the performance of the cluster.
To learn more about BlueStore follow Red Hat Ceph [documentation](https://docs.redhat.com/en/documentation/red_hat_ceph_storage/3/html/administration_guide/osd-bluestore)


- In BlueStore Data is directly written to the raw block device and all metadata operations are managed by RocksDB.
- The device containing the OSD is divided between RocksDB metadata and the actual user data stored in the cluster. 
- User data objects are stored directly on the raw block device, once the data has been written to the block device, RocksDB metadata gets updated with the required details about the new data.

![BlueStore Under the Covers](/assets/img/bs.webp)

---

### Why BlueStore ?

The following are some of the main features of using BlueStore:

- Direct management of storage devices
    - BlueStore avoids any intervening layers of abstraction, such as local file systems like XFS, that might limit performance or add complexity.
- Metadata management with RocksDB
    - BlueStore uses RocksDB a key-value database to manage internal metadata
- Full data and metadata checksumming
    - By default all data and metadata written to BlueStore is protected by one or more checksums. No data or metadata are read from disk or returned to the user without verification.
- Efficient copy-on-write
   - The Ceph Block Device and Ceph File System snapshots rely on a copy-on-write clone mechanism that is implemented efficiently in BlueStore.
- No large double-writes
    - BlueStore first writes any new data to unallocated space on a block device, and then commits a RocksDB transaction that updates the object metadata to reference the new region of the disk. 
    - Only when the write operation is below a configurable size threshold, it falls back to a write-ahead journaling scheme, similar to what how FileStore operates.
- Multi-device support
    - BlueStore can use multiple block devices for storing different data, for example: Hard Disk Drive (HDD) for the data, Solid-state Drive (SSD) for metadata, Non-volatile Memory (NVM) RocksDB write-ahead log (WAL).

---

### BlueStore Devices

BlueStore manages either one, two, or three storage devices.

- primary
- WAL
- DB

In the simplest case, BlueStore consumes a single (primary) storage device. The storage device is partitioned into two parts that contain:
- OSD metadata 
    - A small partition formatted with XFS that contains basic metadata for the OSD. This data directory includes information about the OSD, such as its identifier, which cluster it belongs to, and its private keyring.
- data
    - A large partition occupying the rest of the device that is managed directly by BlueStore and that contains all of the OSD data
    - This primary device is identified by a block symbolic link in the data directory.

You can also use two additional devices:
- WAL (write-ahead-log) device
    - This WAL device is identified by a block.wal symbolic link in the data directory.
- DB Device
    - This DB device is identified by a block.db symbolic link in the data directory.

![Multi Device Support](/assets/img/bs-multidevice.webp)


---

### RocksDB for MetaData

RocksDB is an embedded high-performance key-value store that excels with flash-storage, RocksDB can’t directly write to the raw disk device, it needs and underlying filesystem to store it’s persistent data, this is where BlueFS comes in, BlueFS is a Filesystem developed with the minimal feature set needed by RocksDB to store its sst files.

RocksDB uses WAL as a transaction log on persistent storage, In bluestore we have two different datapaths for writes, one were data is written directly to the block device and the other were we use deferred writes, with deferred writes data gets written to the WAL device and later asynchronously flushed to disk.

Writing directly to the raw block device normally requires several heavyweight operations:

- Allocating space on disk  
- Updating BlueStore metadata  
- Updating RocksDB  
- Managing checksums

For small writes, this overhead becomes very expensive resulting in high latency.

---

### When writes go through the WAL (Write-Ahead Log)

Instead of touching the raw block device immediately, BlueStore can first append data to the WAL:

- The write is quickly appended to the WAL (sequential write, very fast)
- The data is now crash-safe
- BlueStore asynchronously flushes / compacts the data from WAL to the raw block device in the background

So the WAL is not permanent storage it is a temporary durable buffer used to protect data and speed up writes.
