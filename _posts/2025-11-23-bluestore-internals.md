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

```cpp

struct BlueStore : public ObjectStore {
    struct Collection;
    struct Onode;
    struct Blob;
    struct ExtentMap;
    struct Extent;
    ...
};

```

![BlueStore Object to Disk Mapping](/assets/img/bs_obj_to_disk.png)

---

### Collection

```cpp

struct Collection : public CollectionImpl {
    
    BlueStore* store;
    BufferCacheShard *cache;
    bluestore_cnode_t cnode;
    SharedBlobSet shared_blob_set;
    OnodeSpace onode_space;
    
    BlobRef new_blob();

    Collection(BlueStore *ns, OnodeCacheShard *oc, BufferCacheShard *bc, coll_t c);

    ...
};

```

Collection in BlueStore is equivalent to "PG Directory" in FileStore

Collection Contains :

- all Onodes (metadata for Objects)
- all Blobs supporting Onodes
- Shared Blobs
- Caching Structures needed to manage them

```cpp
/// A generic Cache Shard
struct CacheShard {
    std::atomic<uint64_t> max = {0};
    std::atomic<uint64_t> num = {0};
    boost::circular_buffer<std::shared_ptr<int64_t>> age_bins;

    void set_max(uint64_t max_);
    virtual void _trim_to(uint64_t new_size) = 0;
    void flush();
    ...
};
```
```CacheShard``` is a mini LRU cache with size counters.
A small, lock-protected chunk of the in-memory cache.

#### `BufferCacheShard *cache` 

```cpp
/// A Generic buffer Cache Shard
struct BufferCacheShard : public CacheShard {
    std::atomic<uint64_t> num_extents = {0};
    std::atomic<uint64_t> num_blobs = {0};
    uint64_t buffer_bytes = 0;
    ...
};
```

The Local Cache Shard for this Collection.
BlueStore Splits its buffer cache into multiple shards for concurrency
A Collection uses one shard to cache reads/writes for objects inside its PG.
Collection uses this Shard to cache Extents and Blobs.

#### ```OnodeCacheShard``` 
Cache for Onodes
It Manages Pinned Onodes, Reference Count etc.,

```cpp
/// A Generic onode Cache Shard
struct OnodeCacheShard : public CacheShard {
    std::array<std::pair<ghobject_t, ceph::mono_clock::time_point>, 64> dumped_onodes;

    //The following methods prefixed with '_' to be called under
    // Shard's lock
    virtual void _add(Onode* o, int level) = 0;
    virtual void _rm(Onode* o) = 0;
    virtual void _move_pinned(OnodeCacheShard *to, Onode *o) = 0;

    virtual void maybe_unpin(Onode* o) = 0;
    virtual void add_stats(uint64_t *onodes, uint64_t *pinned_onodes) = 0;
    bool empty();
    ...
};

```

#### ```OnodeSpace onode_space```

```cpp
struct OnodeSpace {
    OnodeCacheShard *cache;
private:
    mempool::bluestore_cache_meta::unordered_map<ghobject_t,OnodeRef> onode_map;
    ...
}
```

Holds ```OnodeCacheShard``` and ```onode_map``` which is a map for ```ghobject_t``` Cephs Internal Object ID to an ```OnodeRef``` intrusive pointer to its Onode.

#### ```SharedBlobSet shared_blob_set```

A per-Collection registry of all “shared blobs” (blobs that are used by more than one object or extent).

Normally, a Blob belongs to one object (Onode).
But BlueStore supports sharing the same underlying disk block between multiple logical references

- when doing ```clone_range```  

- when using ```copy-on-write```

- when multiple extents point to the same physical block

These Blobs become ```SharedBlobs```.

#### ```bluestore_cnode_t cnode```

The persisted metadata record of the Collection.

```cnode``` stores the Collection’s state as seen in BlueStore’s on-disk
and encoded into RocksDB

When BlueStore starts up:

- It decodes and loads ```cnode``` from RocksDB

- reconstructs the in-memory Collection structure

- keeps updates consistent as mutations happen

- ensures crash consistency for Collection-level metadata
