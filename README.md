# Arch Linux Installation Notes

> [!NOTE]
> This note is intended for reference and is not intended as a tutorial.


I used the Archinstall script included with the official ISO. I’ve done manual installations before, but Archinstall offered clarity and introduced concepts I previously ignored — like disk encryption and unified kernel images (UKI). While manual installation teaches you the fundamentals, Archinstall ties together components that the Arch Wiki spreads across multiple pages.

The Arch Wiki remains essential, but its installation guide intentionally keeps things minimal and references many sub-pages (partitioning, encryption, bootloaders, etc.). Archinstall gives a structured overview of these without removing control.

Still, I recommend going through the wiki at least once. Understanding what each option means is vital for troubleshooting or customizing later.

---

# Pre-Installation using archinstall

## Disk & Filesystem Configuration

### Disk Partitioning

In archinstall, I performed **manual partitioning** with the following layout:

| Mount Point | Size         | Type                 | Filesystem | Description                                                        |
| ----------- | ------------ | -------------------- | ---------- | ------------------------------------------------------------------ |
| `/efi`      | 1 GB         | EFI System Partition | vfat       | Mounted at `/efi` instead of `/boot` (current Arch recommendation) |
| `/`         | Rest of disk | LUKS Encrypted       | btrfs      | Main system volume                                                 |

Encryption was configured with **LUKS** on the root partition.
The decrypted device is mapped as `/dev/mapper/root`.

---

I went with Btrfs as the filesystem due to its Copy-on-Write (CoW) nature and native snapshot support. It simplifies backup and rollback workflows significantly.

### Btrfs Configuration
By default, Archinstall enables compression on all subvolumes. However, I reviewed openSUSE’s Btrfs recommendations, which suggest disabling CoW (nodatacow) for write-heavy subvolumes like /var. I followed that approach for better stability and performance.

- **Filesystem:** `btrfs`
- **Compression:** `zstd:3`
- **SSD-optimized mount options:** `ssd,space_cache=v2,relatime`
- **Special mount options:**
  - `/swap`, `/var`, `/var/log` use `nodatacow` to prevent CoW overhead.

- **References I looked into**
  - [Arch Linux Wiki: Btrfs](https://wiki.archlinux.org/title/Btrfs)
  - [Opensuse Wiki: SDB:BTRFS](https://en.opensuse.org/SDB:BTRFS)
  - [BTRFS specific mount options](https://btrfs.readthedocs.io/en/latest/ch-mount-options.html)

| Mount Point             | Subvolume         | Options         | Notes                   |
| ----------------------- | ----------------- | --------------- | ----------------------- |
| `/`                     | `@`               | compress=zstd:3 | Root filesystem         |
| `/.snapshots`           | `@snapshots`      | compress=zstd:3 | System snapshots        |
| `/boot`                 | `@boot`           | compress=zstd:3 | Boot files on Btrfs     |
| `/home`                 | `@home`           | compress=zstd:3 | User home directory     |
| `/home/.snapshots`      | `@home_snapshots` | compress=zstd:3 | Home snapshots          |
| `/opt`                  | `@opt`            | compress=zstd:3 | Optional software       |
| `/root`                 | `@root`           | compress=zstd:3 | Root user’s home        |
| `/srv`                  | `@srv`            | compress=zstd:3 | Server data             |
| `/swap`                 | `@swap`           | nodatacow       | Btrfs swap directory    |
| `/tmp`                  | `@tmp`            | compress=zstd:3 | Temporary files         |
| `/usr/local`            | `@usr_local`      | compress=zstd:3 | Local programs          |
| `/var`                  | `@var`            | nodatacow       | Logs, databases, caches |
| `/var/cache/pacman/pkg` | `@pkg`            | compress=zstd:3 | Pacman cache            |
| `/var/log`              | `@log`            | nodatacow       | System logs             |

Arch wiki suggests a minimal subvolume structure, but separating mount points into distinct subvolumes means you can snapshot or rollback those areas independently. Having a dedicated /.snapshots and /home/.snapshots allows for independent clean snapshots of each area.

For subvolumes where a lot of small, frequent writes occur (e.g., /var/log, /var), or where snapshots are not needed (e.g., a swapfile), using CoW (copy-on-write) can impose overhead and interfere with some features. The Arch wiki notes: “Btrfs do not support mounting subvolumes from the same partition with different settings regarding COW." By designating /var, /var/log, /swap as nodatacow, you reduce snapshot overhead and avoid performance/tracking issues in those high-write/persistent contexts.

---

### Example

```
# /etc/fstab

# ───────────────────────────────────────────────
# Btrfs subvolumes on LUKS root
# ───────────────────────────────────────────────
/dev/mapper/root  /                   btrfs  rw,relatime,compress=zstd:3,ssd,space_cache=v2,subvol=/@                 0  0
/dev/mapper/root  /.snapshots         btrfs  rw,relatime,compress=zstd:3,ssd,space_cache=v2,subvol=/@snapshots        0  0
/dev/mapper/root  /boot               btrfs  rw,relatime,compress=zstd:3,ssd,space_cache=v2,subvol=/@boot             0  0
/dev/mapper/root  /home               btrfs  rw,relatime,compress=zstd:3,ssd,space_cache=v2,subvol=/@home             0  0
/dev/mapper/root  /home/.snapshots    btrfs  rw,relatime,compress=zstd:3,ssd,space_cache=v2,subvol=/@home_snapshots    0  0
/dev/mapper/root  /opt                btrfs  rw,relatime,compress=zstd:3,ssd,space_cache=v2,subvol=/@opt              0  0
/dev/mapper/root  /root               btrfs  rw,relatime,compress=zstd:3,ssd,space_cache=v2,subvol=/@root             0  0
/dev/mapper/root  /srv                btrfs  rw,relatime,compress=zstd:3,ssd,space_cache=v2,subvol=/@srv              0  0
/dev/mapper/root  /swap               btrfs  rw,relatime,nodatacow,ssd,space_cache=v2,subvol=/@swap                   0  0
/dev/mapper/root  /tmp                btrfs  rw,relatime,compress=zstd:3,ssd,space_cache=v2,subvol=/@tmp              0  0
/dev/mapper/root  /usr/local          btrfs  rw,relatime,compress=zstd:3,ssd,space_cache=v2,subvol=/@usr_local        0  0
/dev/mapper/root  /var                btrfs  rw,relatime,nodatacow,ssd,space_cache=v2,subvol=/@var                    0  0
/dev/mapper/root  /var/cache/pacman/pkg  btrfs  rw,relatime,compress=zstd:3,ssd,space_cache=v2,subvol=/@pkg           0  0
/dev/mapper/root  /var/log            btrfs  rw,relatime,nodatacow,ssd,space_cache=v2,subvol=/@log                    0  0

# ───────────────────────────────────────────────
# EFI System Partition
# ───────────────────────────────────────────────
/dev/nvme0n1p1   /efi                vfat   rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=ascii,shortname=mixed,utf8,errors=remount-ro  0  2

# ───────────────────────────────────────────────
# Swapfile inside Btrfs subvolume
# ───────────────────────────────────────────────
/swap/swapfile   none                swap   defaults  0  0

```

---

## Swap Configuration
- Most modern setups recommend zram for performance, but zswap is already enabled in the default Arch kernel. While zram might offer marginal speed improvements, the difference is minimal on SSDs/NVMe.
- **No dedicated swap partition**. Btrfs makes managing swapfiles easy, so I opted for btrfs-based swapfile. It's flexible and can be resized as needed while I can still opt for zram in case needed. With performance improvements in ssd and nvme, I don't think there's a need for a dedicated swap partition.
- Created **Btrfs-based swap file** under `/swap/swapfile`.
- Added to `/etc/fstab` after installation.
- Mounted with `nodatacow` for stability.

**zram:** disabled during installation.
**zswap:** enabled later with btrfs swapfile as the preferred compression-based swap optimization for 16 GB RAM systems.

> [!NOTE]
> Zswap and Zram shouldn't be used together. You can find more on [Zram](https://wiki.archlinux.org/title/Zram) and [Zswap](https://wiki.archlinux.org/title/Zswap). It is noted that if the related zswap kernel feature remains enabled, it will prevent zram from being used effectivily. This is because zswap functions as a swap cache in front of zram, intercepting and compressing evicted memory pages before they can reach zram. Most people opted for Zram because of it's easy use but disabling zswap seems more troublesome. It's also a reason why I choosed Zswap and btrfs swapfile method.

---

## Bootloader - systemd-boot + UKI
I switched from GRUB (which I used with grub-btrfs) to systemd-boot. My experience with grub-btrfs was that I rarely used the boot menu snapshots — if the system failed to boot, I still needed to arch-chroot and fix things manually. Since rollback through Snapper works fine once booted, a more streamlined boot setup made sense.

Systemd-boot integrates neatly with Unified Kernel Images (UKI), which combine the kernel, initramfs, and metadata into one file. This improves security and simplifies Secure Boot setup. With UKI, signing kernels for Secure Boot was straightforward.

---

# Post-Installation Setup

I skipped the pre-defined Archinstall profiles and additional package sets. My environment restoration is handled through a custom Niri script, which installs my base packages and configures my dotfiles automatically. This ensures I can rebuild my environment consistently without relying on profiles.
- Skipped **Profiles** and **Additional Packages** in `archinstall`.
- Uses a **custom install script** to:
  - Install essential packages.
  - Configure the **niri** Wayland compositor.
  - Apply dotfiles from a personal repository.

The setup avoids desktop environments and uses a minimal, modular approach based on personal configuration. The install script can be found at [niri-install](https://github.com/auralisx/dotfiles/blob/main/niri_install.sh)

---

## Snapper & Snapshot Management

### Overview

Btrfs snapshot management is handled through **Snapper** and **snap-pac**.

- **snapper** provides automatic timeline and pre/post snapshot functionality.
- **snap-pac** integrates with Pacman to automatically take snapshots before and after package transactions.
- Configure snapper for both `/` (system) and `/home` (user data).
- Each subvolume was prepared for snapper manually to ensure clean integration with the existing Btrfs layout.

more information about snapper and snap-pac can be found at [snapper](https://wiki.archlinux.org/title/Snapper).

---

### Snapper Configuration Setup

Since `/` and `/home` subvolumes were already created during installation, Snapper needed to be reinitialized cleanly for both.

**Steps performed:**

```bash
sudo rm -rf /.snapshots /home/.snapshots

# Created Snapper configurations for both system and home:

sudo snapper -c root create-config /
sudo snapper -c home create-config /home

# Deleted the automatically created /.snapshots and /home/.snapshots directories again:

sudo rm -rf /.snapshots /home/.snapshots

# Recreated empty directories and remounted according to /etc/fstab:

sudo mkdir /.snapshots /home/.snapshots
sudo mount -a

```

This ensured Snapper was using the correct, pre-defined subvolumes (@snapshots and @home_snapshots).

Snapper Configuration Files

Located in /etc/snapper/configs/:

```
/etc/snapper/configs/root

/etc/snapper/configs/home
```

Typical adjustments made:

Option Value Notes
SUBVOLUME / and /home Matches fstab layout

```
TIMELINE_CREATE	yes	Enable automatic timeline snapshots
TIMELINE_CLEANUP	yes	Allow periodic cleanup
NUMBER_CLEANUP	yes	Maintain fixed number of snapshots
TIMELINE_LIMIT_HOURLY	10	Keep up to 10 hourly snapshots
TIMELINE_LIMIT_DAILY	7	Keep up to 7 daily snapshots
TIMELINE_LIMIT_WEEKLY	0	Disabled
TIMELINE_LIMIT_MONTHLY	0	Disabled
TIMELINE_LIMIT_YEARLY	0	Disabled
```
