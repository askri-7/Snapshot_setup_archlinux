# B-Tree 

---

## What Is a B-Tree?

A B-tree is a **self-balancing tree** that keeps data sorted and allows fast search, insertion, and deletion — all in **O(log n)** time.

It was invented in 1971 at Boeing Research Labs specifically for one problem: **disk is slow**. Reading from disk happens in large fixed-size blocks. A B-tree node is sized to match exactly one disk block — so every read fetches a full node in one operation.

![B-tree structure overview](https://upload.wikimedia.org/wikipedia/commons/6/65/B-tree.svg)

---

## Key Characteristics


- All **leaf nodes are at the same level** — the tree is always perfectly balanced

- Every internal node with N keys has exactly **N+1 children**
- A B tree of order M can have in each node **at most M children** and **M-1 keys**
-The minimum number of keys in each node **is half of the maximum capacity m**

This last point is what keeps the tree balanced during deletions.

---

## The Three Types of Nodes

![B-tree node types](https://github.com/askri-7/Snapshot_Btrfs_arch/blob/01666f5132a527b8442e9b7ebc3c135c7eb15e38/btree.png)

| Node | Role |
|---|---|
| **Root** | Entry point. At least 1 key, at least 2 children (unless it's also a leaf) |
| **Internal** | Holds keys and pointers to children. Between M/2 and M children |
| **Leaf** | Bottom level. Holds keys and data. No children. All at the same depth |

---

## Why Not Just Use a Binary Tree?

A binary tree node holds 1 key → tall tree → many disk reads.

A B-tree node holds 100+ keys → very flat tree → very few disk reads.

```
Binary tree with 1 billion keys → depth ~30 → 30 disk reads per lookup
B-tree with 1 billion keys     → depth ~3  → 3 disk reads per lookup
```

That difference is why every database and filesystem uses B-trees.

---




## In Summary

```
A B-tree is a sorted, balanced, multi-key tree sized to match disk blocks. It is self-balancing because, after a deletion or insertion, it dynamically adjusts itself to maintain the characteristics stated above.

Result: O(log n) for everything, always balanced, minimal disk reads
```
