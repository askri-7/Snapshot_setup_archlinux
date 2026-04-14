# Btrfs — How It Works

![a forest where each tree has a role](https://images.unsplash.com/photo-1448375240586-882707db888b?w=1200)

> Btrfs is a forest of B_tree a self balancing data structure .

---

## The Core Rule

**Never overwrite. Always write to a new empty block , then update the pointer.**

This is called **Copy-on-Write (CoW)**. Everything in btrfs  snapshots, crash safety, checksums , -> is a consequence of this one rule.

---

## The Architecture
```
Superblock
  └── **Tree of Tree Roots**
        ├── Chunk Tree         
        ├── Extent Tree     
        ├── FS Tree(s)         
        ├── Checksum Tree      
        └── Log Tree         
```

### Superblock

The entry point. The kernel reads this first on boot. 
**Physical locations on disk** (hardcoded, not in any tree):
- Copy 1: offset `64 KiB`
- Copy 2: offset `64 MiB`
- Copy 3: offset `256 MiB`
- Copy 4: offset `1 TiB` (on large disks only)

Three copies exist for redundancy. If the start of your disk degrades physically,
the kernel can still find a valid superblock further in. After every committed
transaction, all copies are updated. The one with the **highest generation
number** is the authoritative one.

**Key fields inside the superblock:**

| Field | Size | Purpose |
|---|---|---|
| Magic | 8 bytes | `_BHRfS_M` — identifies this as btrfs |
| Generation | u64 | Transaction ID — increments on every commit |
| Root tree root | u64 | `bytenr` of the tree-of-tree-roots node |
| Chunk tree root | u64 | `bytenr` of the chunk tree root (see below why) |
| Log tree root | u64 | `bytenr` of the fsync journal tree |
| Total bytes | u64 | Full device size |
| Bytes used | u64 | Currently allocated bytes |
| Node size | u32 | B-tree node size (default 16384 bytes) |
| Sector size | u32 | Smallest addressable unit |
| FSID | 128 bits | UUID identifying this filesystem |
| Device UUID | 128 bits | UUID of this specific disk (matters in RAID) |
| Incompat flags | u64 | Features the kernel *must* understand to mount |
| Compat RO flags | u64 | Features safe to ignore for read-only mount |
| Checksum | 32 bytes | CRC of the rest of the superblock |

### Chunk Tree

Translates logical addresses -> physical disk locations . 

### How Data Is Stored — Extents

Btrfs does not store files byte-by-byte in fixed-size blocks like ext4 does.
Instead it uses **extents**.

An extent is a **contiguous region of disk space** described by two numbers:
- `bytenr` — the starting byte address (in btrfs's logical address space)
- `length` — how many bytes long it is

A single file can be made up of multiple extents scattered across the disk.

 Here is how an extent works: It uses dynamic delayed allocation by storing data in RAM until it reaches a certain threshold. Then, the system finds a contiguous block of free space on the hard disk and writes the raw data. Finally, it dynamically updates the metadata using self-balancing B-tree data structures: it updates the extent pointer in the FS Tree (the subvolume), which is tracked by the master Root Tree, while the Chunk Tree keeps track of where everything is physically stored.

```
File: photo.jpg  (3.2 MB)
  Extent 1: logical address 0x4A000000, length 1MB
  Extent 2: logical address 0x7F200000, length 2MB
  Extent 3: logical address 0x1C800000, length 0.2MB
```



### Subvolume = An FS Tree

In Btrfs, a subvolume is literally just an independent File System B-Tree (FS Tree). It holds all the directories, inodes (metadata leaves), and extent pointers for that specific namespace.

### The Root Tree Entry = The Pointer 

How does the system keep track of all these subvolumes? It uses a master tree called the "Tree of Tree Roots" that holds pointers to the root nodes of every individual subvolume (FS Tree) on the disk.

### Checksum Tree

Stores a checksum for every data extent. On every read, btrfs recomputes and compares. Corruption is caught silently.




### A reference counter (refcount)
 counts the number of B-trees pointing to the same raw data. When it reaches 0, the system marks that space on the hard disk as free and available to be overwritten

## Simulating a File Insertion

Let's trace exactly what happens when you run:

```bash
cp photo.jpg /mnt/data/
```

---

**Step 1 — Write the data**

Btrfs finds free space via the Extent Tree and writes the raw bytes of `photo.jpg` to a fresh extent on disk.

```
Extent E1
  logical address: 0x4A000000
  length: 3MB
  ref count: 0  ← not yet referenced
```

---

**Step 2 — Create the inode**

Btrfs creates an inode item in the FS Tree for `photo.jpg`:

```
Inode 1337
  size: 3MB
  permissions: 644
  owner: karappo
  timestamps: now
```

---

**Step 3 — Link the extent to the inode**

An extent data item is added to the FS Tree, connecting inode 1337 to extent E1:

```
Inode 1337
  └── extent data → E1 (logical 0x4A000000, length 3MB)
```

The ref count of E1 in the Extent Tree is incremented to 1.

---

**Step 4 — Create the directory entry**

A dir item is added to the parent directory's inode in the FS Tree:

```
/mnt/data/  (inode 256)
  └── "photo.jpg" → inode 1337
```

---

**Step 5 — Write the checksum**

Btrfs computes a CRC32C checksum of the raw bytes of E1 and stores it in the Checksum Tree.

```
Checksum Tree
  └── 0x4A000000 → 0xDEADBEEF
```

---

**Step 6 — Commit the transaction**

All the changes above happened in memory. Now btrfs flushes them to disk, writes new B-tree nodes via CoW from leaf to root, and atomically updates the superblock with a new generation number.

```
Superblock gen 41 → gen 42
```

The file now exists permanently on disk. If the machine had crashed before step 6, none of it would be visible — the old superblock (gen 41) would still be valid.

---

## Simulating a Snapshot

Now let's trace what happens when you run:

```bash
btrfs subvolume snapshot -r /mnt/data /snapshots/data-backup
```

---

**Before the snapshot**

```
Subvolume 256 (@data)
  └── FS Tree root → node R1
        ├── photo.jpg → Extent E1 (ref count: 1)
        ├── video.mp4 → Extent E2 (ref count: 1)
        └── notes.txt → Extent E3 (ref count: 1)
```

---

**Step 1 — Create a new subvolume entry**

Btrfs allocates a new subvolume ID (e.g. 300) in the Tree of Tree Roots.

---

**Step 2 — Point it at the same FS Tree root**

The new subvolume 300 starts with its root pointing to the **exact same node R1**.

```
Subvolume 256 (@data)       Subvolume 300 (@data-backup)
  └── FS Tree root → R1 ←──────── FS Tree root
```

No data copied. No new extents. Just a new pointer.

---

**Step 3 — Increment all reference counts**

Every extent reachable from R1 has its ref count incremented.

```
Extent E1 (photo.jpg)   ref count: 1 → 2
Extent E2 (video.mp4)   ref count: 1 → 2
Extent E3 (notes.txt)   ref count: 1 → 2
```

Both subvolumes now legally own these extents. Neither will free them alone.

---

**Step 4 — Commit**

Superblock generation increments. The snapshot is permanent. Total time: milliseconds. Total extra space: nearly zero.

---

**What happens when you modify photo.jpg in the original**

CoW kicks in:

```
Step 1: new bytes written to fresh Extent E1'
Step 2: FS Tree of subvolume 256 updated via path copy → new root R2
Step 3: E1' ref count → 1  (only subvolume 256 owns it)
        E1  ref count → 1  (only subvolume 300 still owns it)
```

```
Subvolume 256 (@data)       Subvolume 300 (@data-backup)
  └── FS Tree root → R2       └── FS Tree root → R1
        └── photo.jpg → E1'         └── photo.jpg → E1  ← original preserved
```

The snapshot still sees the old `photo.jpg`. The original has the new version. Nothing was duplicated — only the diff costs new space.

---

**What happens when you delete the snapshot**

```bash
btrfs subvolume delete /snapshots/data-backup
```

- Subvolume 300 is removed from the Tree of Tree Roots
- Ref counts of E1, E2, E3 are decremented
- E2 and E3 drop to 0 → freed
- E1 stays at 1 → still owned by subvolume 256 → not freed

---

## Summary

```
One rule:   never overwrite → write new → update pointer (CoW)

Insertion:  write extent → create inode → link extent → add dir entry
            → store checksum → commit transaction

Snapshot:   new subvolume ID → same FS tree root → increment ref counts
            → commit → done, zero data copied

Divergence: modify original → CoW creates new nodes and extents
            → snapshot keeps old root → only diff costs space
```
