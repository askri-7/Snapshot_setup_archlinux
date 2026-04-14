# B-Tree 

---

## What Is a B-Tree?

A B-tree is a **self-balancing tree** that keeps data sorted and allows fast search, insertion, and deletion — all in **O(log n)** time.

It was invented in 1971 at Boeing Research Labs specifically for one problem: **disk is slow**. Reading from disk happens in large fixed-size blocks. A B-tree node is sized to match exactly one disk block — so every read fetches a full node in one operation.

![B-tree structure overview](https://upload.wikimedia.org/wikipedia/commons/6/65/B-tree.svg)

---

## Key Characteristics

- Each node holds **multiple keys** — not just one like a binary tree
- All **leaf nodes are at the same level** — the tree is always perfectly balanced
- Keys inside each node are kept in **sorted order**
- Every internal node with N keys has exactly **N+1 children**
- A node of order M can have **at most M children** and **M-1 keys**
- Every non-root node must be **at least half full**

This last point is what keeps the tree balanced during deletions.

---

## The Three Types of Nodes

![B-tree node types](https://upload.wikimedia.org/wikipedia/commons/thumb/6/65/B-tree.svg/800px-B-tree.svg.png)

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

## Insertion — What Happens

New keys are **always inserted at the leaf level**. The tree then fixes itself upward if needed.

### Case 1 — The leaf has space

Find the right leaf, insert the key in sorted order. Done.

![Simple insertion into B-tree](https://upload.wikimedia.org/wikipedia/commons/thumb/8/8b/Insertion-3.png/640px-Insertion-3.png)

### Case 2 — The leaf is full → Split

When a leaf is full and a new key arrives:

1. The node **splits in two**
2. The **middle key is pushed up** to the parent
3. Left half stays, right half becomes a new node
4. If the parent is also full → it splits too → propagates upward
5. If the root splits → a **new root is created** and the tree grows by one level

![B-tree node splitting](https://upload.wikimedia.org/wikipedia/commons/thumb/b/b7/B-tree-splitt.png/640px-B-tree-splitt.png)

This is why the tree always stays balanced — growth only happens at the root, and every leaf stays at the same depth.

---

## Deletion — What Happens

Deletion is more complex than insertion. There are two main cases.

### Case 1 — Delete from a leaf with enough keys

Find the key, remove it. The node still has at least M/2 - 1 keys. Done.

![Deletion from B-tree step 1](https://upload.wikimedia.org/wikipedia/commons/thumb/9/9a/Delete-key-from-Btree-1.png/640px-Delete-key-from-Btree-1.png)

### Case 2 — The node becomes too empty → Borrow or Merge

After deletion, if a node falls below the minimum number of keys, btrfs has two options:

**Option A — Borrow from a sibling**

If a neighboring sibling has extra keys, rotate one key through the parent.

![Deletion borrow from sibling](https://upload.wikimedia.org/wikipedia/commons/thumb/6/60/Delete-key-from-Btree-2.png/640px-Delete-key-from-Btree-2.png)

**Option B — Merge nodes**

If no sibling has spare keys, merge the node with a sibling and pull a key down from the parent. This can propagate upward. If the root loses its last key, it is deleted and the tree shrinks by one level.

![Deletion merge nodes](https://upload.wikimedia.org/wikipedia/commons/thumb/f/f2/Delete-key-from-Btree-3.png/640px-Delete-key-from-Btree-3.png)

### Case 3 — Delete from an internal node

You can't just remove a key from an internal node because it acts as a separator between children. Instead:

1. Replace it with its **in-order successor** (the smallest key in the right subtree)
2. Delete that successor from the leaf level
3. Apply borrowing or merging if needed

---

## The Effect on Tree Shape

| Operation | Effect |
|---|---|
| Insert into non-full leaf | No shape change |
| Insert causes split | Tree may grow by one level at the root |
| Delete from full-enough leaf | No shape change |
| Delete causes merge | Tree may shrink by one level at the root |
| Delete causes borrow | Parent key rotates, shape stays the same |

The tree **only ever grows or shrinks at the root** — this is what keeps all leaves at the same depth at all times.

---

## In Summary

```
B-tree = sorted, balanced, multi-key tree sized to match disk blocks

Insertion → always at the leaf → split upward if full → tree grows at root
Deletion  → always at the leaf → borrow or merge if too empty → tree shrinks at root

Result: O(log n) for everything, always balanced, minimal disk reads
```
