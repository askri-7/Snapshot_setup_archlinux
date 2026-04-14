![](https://github.com/askri-7/Snapshot_Btrfs_arch/blob/64f532fb2c1e124567b5b9d13f3a5ef15e7611e2/arch_logo.png)

# Arch Linux: The Ultimate Btrfs & Snapper Setup Guide

This guide provides a comprehensive walkthrough for installing Arch Linux using `archinstall` and configuring a robust, snapshot-capable Btrfs filesystem using **Snapper** and **GRUB**. By following this setup, you will have a system that can be easily rolled back to a previous state directly from the boot menu.

---

## Helpful Arch Wiki Links

| Resource | Description |
| :--- | :--- |
| [Archinstall](https://wiki.archlinux.org/title/Archinstall) | Official Arch Linux installation script documentation. |
| [Btrfs](https://wiki.archlinux.org/title/Btrfs) | Detailed guide on Btrfs filesystem features and management. |
| [Snapper](https://wiki.archlinux.org/title/Snapper) | Tool for managing Btrfs subvolume snapshots. |
| [GRUB](https://wiki.archlinux.org/title/GRUB) | The GRUB bootloader configuration and usage. |

---

## 1. Pre-Installation & Subvolume Setup

Before starting, ensure you have an active internet connection.

### Network Configuration
If you are on Wi-Fi, connect before running the installer:
1. **Check connection:** `ip a` and test with `ping google.com`.
2. **List wireless interfaces:** `iw dev`.
3. **List available networks:** `iw dev wl0 scan | grep SSID`.
4. **Connect:** `iwctl --passphrase "your_password" station wl0 connect`.

### Disk Partitioning via Archinstall
Launch the installer by typing `archinstall`. Follow the initial prompts for language, locale, and mirrors.

1. **Manual Partitioning:**
   - Create a **1GiB EFI partition** (Format: `FAT32`, Mount point: `/boot` initially).
   - Create a **Btrfs root partition** using the remainder of the disk.
   - Enable **zstd compression** for the Btrfs partition to optimize space and performance.
2. **Define Subvolumes:**
   - Create standard subvolumes: `@` (root), `@home`, `@var/log`, `@var/cache`, `@var/lib/libvirt/images`.
   - *Note: Segmenting these directories prevents logs and VM images from being overwritten during a system rollback.*
3. **Disable UKI Mode:**
   - In the `archinstall` menu, ensure **Unified Kernel Image (UKI)** mode is **disabled**. This ensures standard `initramfs` generation via `mkinitcpio`, which is required for our custom hooks.

### Save and Edit Configuration
To properly set up GRUB to boot from snapshots, the EFI partition should be mounted at `/efi`, keeping `/boot` inside the Btrfs root.

1. Select **"Save configuration"** to `/root/config.json` in the installer menu.
2. Exit the installer.
3. Edit the JSON file: `nano /root/config.json`.
4. Change the mount path for the FAT32 partition from `/boot` to `/efi`.
5. Re-run the installer using your custom config: `archinstall --config /root/config.json`.
6. Continue configuring user accounts, desktop environment (e.g., KDE Plasma), and NetworkManager via the TUI menu.

---

## 2. Chroot Configuration (Before First Reboot)

When the installation finishes, select **"Yes"** to chroot into the newly installed system to make crucial adjustments.

### Disable Copy-on-Write (CoW) for VMs
Virtual machine images perform poorly on CoW filesystems. Disable CoW for the libvirt directory:
```bash
chattr +C /var/lib/libvirt/images
```

### Reconfigure GRUB for Snapshots
To allow GRUB to read Btrfs snapshots, `/boot` must exist on the Btrfs partition, and the bootloader files go to `/efi`.

1. **Remove the old path:** `rm -rf /efi/grub`.
2. **Re-install GRUB:**
   ```bash
   grub-install --target=x86_64-efi --efi-directory=/efi --boot-directory=/boot --bootloader-id=Arch
   ```
3. **Generate the config:**
   ```bash
   grub-mkconfig -o /boot/grub/grub.cfg
   ```
4. **(Optional) Fix EFI boot entry:**
   ```bash
   efibootmgr --create --disk /dev/sda --part 1 --label "Arch" --loader /EFI/Arch/grubx64.efi
   ```
5. Type `exit` and then `reboot`.

---

## 3. Installing & Configuring Snapper

Boot into your new Arch system and open a terminal. Install the tools required to manage snapshots and integrate them into the bootloader.

### Install Packages
Install an AUR helper like `yay`, then install the snapshot stack:
```bash
sudo pacman -S snapper snap-pac grub-btrfs inotify-tools
yay -S btrfs-assistant
```
- **snap-pac:** Automates pacman pre/post snapshots.
- **grub-btrfs:** Automatically adds snapshots to the GRUB boot menu.

### Initialize Snapper
Create the base configurations for your root and home directories:
```bash
sudo snapper -c root create-config /
sudo snapper -c home create-config /home
```

### Optimize Permissions & Search
1. **User Permissions:** Allow your user to manage root snapshots without sudo:
   ```bash
   sudo snapper -c root set-config ALLOW_USERS="your_username" SYNC_ACL="yes"
   ```
2. **Search Indexing:** Prevent the `locate` command from indexing snapshot data by adding `.snapshots` to the `PRUNENAMES` list inside `/etc/updatedb.conf`.

### Enable Read-Only Snapshot Booting (OverlayFS)
If you boot into a read-only snapshot from GRUB, the system might crash because it cannot write temporary files. We fix this with an initramfs hook.

1. Edit `/etc/mkinitcpio.conf`.
2. Add `grub-btrfs-overlayfs` to the end of the `HOOKS=(...)` array.
3. Regenerate the initramfs: `sudo mkinitcpio -P`.
4. Enable the GRUB auto-update daemon: `sudo systemctl enable --now grub-btrfsd`.

---

## 4. Snapshot Management & Recovery

### Automating Snapshots
Ensure you always have a restore point by enabling timeline timers:
```bash
sudo systemctl enable --now snapper-timeline.timer
sudo systemctl enable --now snapper-cleanup.timer
```
Note: To save disk space, it is highly recommended to disable automatic timeline snapshots for your home directory while keeping them enabled for the root system. Run this command to update the configuration:
```
 sudo snapper -c home set-config TIMELINE_CREATE="no".
```
### Full System Rollback

  If the system becomes entirely unbootable:
---


*(This highlights the exact remount command needed before running the rollback or opening the GUI.)*
### Full System Rollback (The Manual "Subvolid 5" Method)
If the system becomes entirely unbootable, or if `snapper rollback` fails, you can manually replace the broken root subvolume by mounting the top-level Btrfs tree.

1. Reboot and select **"Arch Linux Snapshots"** from the GRUB menu.
2. Boot into a known-working snapshot. *(Note: Because you booted into a snapshot, your current `/` directory is actually the good snapshot!)*
3. **Mount the top-level Btrfs tree (subvolid=5):** Open a terminal and mount your main Arch partition to `/mnt`. *(Change `/dev/sda2` to match your actual drive, e.g., `/dev/nvme0n1p2`)*:
```
  sudo mount -t btrfs -o subvolid=5 /dev/sda2 /mnt
  ```


List the contents to verify you see your subvolumes (like @, @home, etc.)
:
```
   ls -l /mnt
```
Move the broken root out of the way: Rename your corrupted @ subvolume so the system no longer uses it:
 ```  
   sudo mv /mnt/@ /mnt/@-broken
```
Clone the snapshot: Take a read-write snapshot of your currently running system (/) and place it exactly where the old root used to be (/mnt/@):
```
   sudo btrfs subvolume snapshot / /mnt/@
```
Cleanup and Reboot: Unmount the top-level tree and reboot the computer. It will naturally boot into your newly restored @ subvolume!
```
sudo umount /mnt
reboot
```
