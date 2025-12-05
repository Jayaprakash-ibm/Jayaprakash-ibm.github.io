---
title: "BlueStore Internals"
date: 2025-11-23 15:33:00 +0530
categories: [BlueStore, Ceph, Internals]
tags: [bluestore, ceph, storage, architecture, internals]
image: /assets/img/bs-internals.webp
alt: "BlueStore Introduction"
---

### Data Structures in BlueStore

BlueStore maintains a sophisticated data model built around structures like Onode, Blob, and ExtentMap.
These components work together to map high-level Ceph objects to their physical locations on disk

Here are some useful Core Structures:

```C++

struct BlueStore : public ObjectStore {
    struct Collection;
    struct Onode;
    struct Blob;
    struct ExtentMap;
    struct Extent;
}

```

![BlueStore Object to Disk Mapping](/assets/img/bs_obj_to_disk.png)

---

### Collection

```C++

struct Collection : public CollectionImpl {
    
    BlueStore* store;
    BufferCacheShard *cache;
    bluestore_cnode_t cnode;
    SharedBlobSet shared_blob_set;
    OnodeSpace onode_space;
    
    BlobRef new_blob();

    Collection(BlueStore *ns, OnodeCacheShard *oc, BufferCacheShard *bc, coll_t c);
};

```
