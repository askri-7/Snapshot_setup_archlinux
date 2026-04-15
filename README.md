![Arch Linux](./arch_logo.png)

# Snapshot Setup on Arch Linux

This repo started from a personal experience — I set up snapshots on my Arch Linux system, found it genuinely interesting, and decided to dive deep into understanding how btrfs actually works under the hood. So I documented everything I learned and built along the way.

---

## What's in This Repo

### 📖 [Arch Linux: Snapper Setup Guide](./Arch%20Linux_%20Snapper%20Setup%20Guide.md)
A practical step-by-step guide to setting up Snapper for automatic snapshots on Arch Linux with a btrfs filesystem.

### 🌲 [How Btrfs Works](./btrfs_explained.md)
A deep dive into btrfs internals — how it manages data, what copy-on-write really means, how the tree structure works, and how snapshots are handled at the filesystem level.

### 🔍 [What Is a B-Tree](./btree.md)
A focused explanation of B-trees — the data structure that btrfs (and most databases) are built on. Covers the key characteristics, and how insertion and deletion affect the tree.

---

## Why I Made This

Most guides just tell you what commands to run. I wanted to understand *why* those commands work — what is actually happening on disk when a snapshot is created, why it is instant, and why it takes almost no extra space.

If you are curious about the same things, this repo is for you.

---

## Where to Start

If you just want to set up snapshots → go to the **Snapper Setup Guide**

If you want to understand what is happening under the hood → start with **How Btrfs Works**, then read **What Is a B-Tree**
