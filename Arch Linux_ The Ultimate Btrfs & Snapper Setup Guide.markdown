<p align="center">
  <img src="assets/arch_logo.png" alt="Arch Linux Logo" width="400">
</p>

# Arch Linux: The Ultimate Btrfs & Snapper Setup Guide

This guide provides a comprehensive walkthrough for installing Arch Linux using `archinstall` and configuring a robust, snapshot-capable Btrfs filesystem using **Snapper** and **GRUB**. By following this setup, you will have a system that can be easily rolled back to a previous state directly from the boot menu.

---

## Helpful Arch Wiki Links

| Resource | Description |
| :--- | :--- |
| [Archinstall](https://wiki.archlinux.org/title/Archinstall) | Official Arch Linux installation script documentation. |
| [Btrfs](https://wiki.archlinux.org/title/Btrfs) | Detailed guide on Btrfs filesystem features and management. |
| [Snapper](https://wiki.archlinux.org/title/Snapper) | Tool for managing Btrfs snapshots. |
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

### User Customization
1. **Set Display Name:** Update your display name in the `/etc/passwd` file:
   ```bash
   chfn -f "Your Full Name" your_username
   ```
2. **Configure Bash Aliases:** Edit your `.bashrc` to add preferred aliases:
   ```bash
   echo "alias ls='ls -lh --color=auto'" >> ~/.bashrc
   ```

### Relocate GRUB to /boot
To ensure rollback consistency, the GRUB directory should reside in `/boot` (on the Btrfs root) rather than the EFI partition.

1. **Remove the existing GRUB directory from the EFI path:**
   ```bash
   rm -rf /efi/grub
   ```
2. **Install GRUB into the root file system:**
   ```bash
   grub-install --target=x86_64-efi --efi-directory=/efi --boot-directory=/boot --bootloader-id=Arch
   ```
3. **Generate the GRUB configuration:**
   ```bash
   grub-mkconfig -o /boot/grub/grub.cfg
   ```
4. **(Optional) Manual Boot Entry:** If a boot entry is not automatically created, use `efibootmgr`:
   ```bash
   efibootmgr --create --disk /dev/sda --part 1 --label "Arch Linux" --loader /EFI/Arch/grubx64.efi
   ```

**Exit Chroot:** Once finished, type `exit` and then `reboot`.

---

## 3. Installing & Configuring Snapper

Boot into your new Arch system and open a terminal. Install the tools required to manage snapshots and integrate them into the bootloader.

### Install Packages
Install an AUR helper like `yay`, then install the snapshot stack:
```bash
sudo pacman -S snapper snap-pac grub-btrfs inotify-tools
yay -S btrfs-assistant
```

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

### Undo Pacman Changes (Live)
If a system update breaks a package, you can revert it live:
1. **Check snapshot IDs:** `snapper -c root list`.
2. **Review changes:** `snapper -c root status 1..2`.
3. **Revert changes:** `snapper -c root undochange 1..2`.

### Full System Rollback
If the system becomes entirely unbootable:
1. Reboot and select **"Arch Linux Snapshots"** from the GRUB menu.
2. Boot into a known-working snapshot.
3. Once logged in, perform the rollback:
   ```bash
   snapper -c root rollback
   ```
4. Reboot normally.
5. Update GRUB to sync the menu: `sudo grub-mkconfig -o /boot/grub/grub.cfg`.
