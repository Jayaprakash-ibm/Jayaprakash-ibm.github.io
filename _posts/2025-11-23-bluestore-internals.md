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

Some Core Data Structures used in a Collection:

---

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

---

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

---

#### ```OnodeCacheShard``` 
Cache for Onodes. It Manages Pinned Onodes, Reference Count etc.,

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

---

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

---

#### ```SharedBlobSet shared_blob_set```

A per-Collection registry of all “shared blobs” (blobs that are used by more than one object or extent).

Normally, a Blob belongs to one object (Onode).
But BlueStore supports sharing the same underlying disk block between multiple logical references

- when doing ```clone_range```  

- when using ```copy-on-write```

- when multiple extents point to the same physical block

These Blobs become ```SharedBlobs```.

---

#### ```bluestore_cnode_t cnode```

The persisted metadata record of the Collection.

```cnode``` stores the Collection’s state as seen in BlueStore’s on-disk
and encoded into RocksDB

When BlueStore starts up:

- It decodes and loads ```cnode``` from RocksDB

- reconstructs the in-memory Collection structure

- keeps updates consistent as mutations happen

- ensures crash consistency for Collection-level metadata

---

### Onode

```cpp
struct Onode {
    MEMPOOL_CLASS_HELPERS();

    std::atomic_int nref = 0;      ///< reference count
    std::atomic_int pin_nref = 0;  ///< reference count replica to track pinning
    Collection *c;
    ghobject_t oid;
    ...

    bluestore_onode_t onode;  ///< metadata stored as value in kv store
    ...

    ExtentMap extent_map;
    BufferSpace bc;             ///< buffer cache

    // track txc's that have not been committed to kv store (and whose
    // effects cannot be read via the kvdb read methods)
    std::atomic<int> flushing_count = {0};
    std::atomic<int> waiting_count = {0};
    /// protect flush_txns
    ceph::mutex flush_lock = ceph::make_mutex("BlueStore::Onode::flush_lock");
    ceph::condition_variable flush_cond;   ///< wait here for uncommitted txns
    std::shared_ptr<int64_t> cache_age_bin;  ///< cache age bin
    ...
    
    static void decode_raw(
      BlueStore::Onode* on,
      const bufferlist& v,
      ExtentMap::ExtentDecoder& dencoder,
      bool use_onode_segmentation);
    ...
};
```

An **Onode** is the in-memory representation of a single RADOS object inside BlueStore.
Each object stored in a Placement Group (Collection) has exactly one Onode in memory when loaded.
It acts as the **runtime control structure** for that object.
The following sections describe the internal structure of an Onode and the role of its key components.

---

#### Reference Management

```cpp
std::atomic_int nref = 0;      ///< reference count
std::atomic_int pin_nref = 0;  ///< reference count replica to track pinning
```

- **`nref`** — Normal intrusive reference count.  
  When it drops to zero, the Onode can be removed from cache.

- **`pin_nref`** — Special reference count used for pinning.  
  Pinned Onodes:
  - cannot be evicted from cache  
  - are actively being modified  
  - are part of ongoing transactions  

Pinning prevents eviction while the object is busy.

---

#### Ownership and Identity

- **`Collection *c;`**  
  Back pointer to the `Collection` that owns this Onode.  
  Through this pointer, the Onode can access:
  - cache shard  
  - shared blobs  
  - BlueStore instance  
  - other PG-level structures  

- **`ghobject_t oid;`**  
  Ceph’s internal object identifier.  
  This uniquely identifies the RADOS object inside the cluster.

---

#### Persistent Metadata

- **`bluestore_onode_t onode;`**  
  The persistent metadata of the object stored in RocksDB.  
  It contains information such as:
  - object size  
  - attributes  
  - flags  
  - allocation hints  

  The `Onode` struct is the in-memory wrapper, while  
  `bluestore_onode_t` is the serialized on-disk representation.

---

#### Logical to Physical Mapping

- **`ExtentMap extent_map;`**  
  Maintains the mapping from object logical offsets to physical disk locations.

  It represents:
    - Object Logical Offset &rarr; Extent &rarr; Blob &rarr; Disk

This is effectively the block map of the object.

---

#### Per-Object Buffer Cache

- **`BufferSpace bc;`**  
  Per-Onode read/write cache attached to the object.

  It stores:
  - buffered writes not yet committed  
  - small dirty regions  
  - in-flight modifications  

  This allows BlueStore to:
  - batch writes  
  - reduce disk I/O  
  - delay persistence until transaction commit  
  - maintain consistency before flushing to RocksDB  

  Think of this as a small write buffer owned by the object itself.

---

#### Flush Tracking for Object

```cpp
std::atomic<int> flushing_count;
std::atomic<int> waiting_count;
ceph::mutex flush_lock;
ceph::condition_variable flush_cond;
```

When a transaction modifies an Onode, the changes are not immediately visible in RocksDB.
They must first be flushed (committed).

These fields:

- `flushing_count` — Number of transactions currently being flushed
- `waiting_count` — Number of waiters blocked on flush completion
- `flush_lock` — Protects flush-related state
- `flush_cond` — Condition variable to wait for commit completion

They ensure:
- tracking uncommitted update
- block readers if required
- ordering between transactions is preserved
- consistency is maintained

This effectively acts as a per-object transaction coordination mechanism.

---

#### Cache Aging

- **`std::shared_ptr<int64_t> cache_age_bin;`**

  Connects the Onode to the LRU aging system inside `OnodeCacheShard`.

  It allows the cache shard to:

  - track how recently the Onode was accessed  
  - place it into an aging bucket  
  - evict older Onodes first  

  During eviction:

  - older Onodes are removed first  
  - pinned Onodes (`pin_nref > 0`) are protected  

  This integrates the object into BlueStore’s sharded LRU cache logic.

---

#### `decode_raw()`

`decode_raw()` loads an Onode from its serialized value stored in RocksDB.

It performs the following steps:

1. Reads metadata from the KV store  
2. Decodes it into `bluestore_onode_t`  
3. Reconstructs the `ExtentMap`  
4. Rebuilds the full in-memory `Onode` structure  

This function is called when an object is loaded from disk into memory.

### Blob

```cpp
/// in-memory blob metadata and associated cached buffers (if any)
struct Blob {
    MEMPOOL_CLASS_HELPERS();

    std::atomic_int nref = {0};     ///< reference count
    int16_t id = -1;                ///< id, for spanning blobs only, >= 0
    int16_t last_encoded_id = -1;   ///< (ephemeral) used during encoding only
    CollectionRef collection;

    void set_shared_blob(SharedBlobRef sb) {
      ceph_assert((bool)sb);
      ceph_assert(!shared_blob);
      ceph_assert(sb->collection = collection);
      shared_blob = sb;
      ceph_assert(get_cache());
    }
private:
    SharedBlobRef shared_blob;      ///< shared blob state (if any)
    mutable bluestore_blob_t blob;  ///< decoded blob metadata
    /// refs from this shard.  ephemeral if id<0, persisted if spanning.
    bluestore_blob_use_tracker_t used_in_blob;
    ...

};

typedef boost::intrusive_ptr<Blob> BlobRef;
typedef mempool::bluestore_cache_meta::map<int,BlobRef> blob_map_t;
```