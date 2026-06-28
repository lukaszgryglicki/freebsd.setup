# FreeBSD 15.1 Tuxedo Laptop Install Runbook

**Target:** FreeBSD 15.1-RELEASE on a Tuxedo laptop with 64 GB RAM and 2 × 4 TB disks.

**Final design:** FreeBSD package-base install, root-on-ZFS, two-disk ZFS mirror, GELI below ZFS, standard FreeBSD ZFS boot-environment dataset layout, no swap, `/tmp` on tmpfs, `/var/tmp` redirected to `/tmp`, and no manual ARC limits.

---

## 0. Non-negotiable warning

This runbook **destroys both internal disks**.

Before continuing, confirm that any data on the two internal disks has been backed up elsewhere and that the GELI passphrase and later GELI metadata backups are stored safely.

A ZFS mirror is not a backup. It protects against one disk failure. It does not protect against accidental deletion, theft, filesystem-level mistakes, corrupted writes replicated to both disks, or loss of the GELI keys/passphrase.

---

## 1. Final requirements implemented by this runbook

```text
FreeBSD version:          15.1-RELEASE amd64
Install method:           Packages (Tech Preview) / pkgbase
Partitioning method:      Auto (ZFS)
Disk topology:            2 × 4 TB disks as a ZFS mirror
Main pool:                zroot
Boot pool:                bootpool, unencrypted, required for GELI-root boot
Encryption:               GELI below ZFS
ZFS native encryption:    off
Compression:              zstd-10
Deduplication:            sha256
atime:                    off
4K sectors / ashift:      yes, ashift=12
Swap:                     none
/tmp:                     tmpfs, size=512m
/var/tmp:                 symlink to /tmp
Manual ARC limits:        none
Suspend/resume:           not configured / not required
```

The main pool capacity will be approximately one disk, so roughly 4 TB raw before ZFS metadata, EFI/boot partitions, reservation effects, and normal filesystem overhead.

---

## 2. Important ZFS terminology for this install

You will have **two pools**:

```text
bootpool   unencrypted boot pool, mirrored, contains boot files
zroot      encrypted main root pool, mirrored, contains the OS and data
```

Inside `zroot`, FreeBSD Auto ZFS will create multiple **datasets**, for example:

```text
zroot/ROOT/default
zroot/home
zroot/usr
zroot/usr/src
zroot/usr/ports
zroot/var
zroot/var/log
zroot/var/mail
zroot/var/tmp
zroot/tmp
```

These are **not separate pools** and they do **not** split the 4 TB capacity unless you explicitly add quotas or reservations later. They are logical ZFS filesystems inside the same `zroot` pool.

Keeping the standard dataset layout is intentional because it preserves FreeBSD boot-environment support with `bectl`.

After installation, this runbook removes only these temporary datasets:

```text
zroot/tmp
zroot/var/tmp
```

and replaces them with:

```text
/tmp      tmpfs, 512m
/var/tmp  -> /tmp
```

Do **not** destroy `zroot/ROOT`, `zroot/ROOT/default`, `zroot/home`, `zroot/usr`, `zroot/var`, `zroot/var/log`, `zroot/var/mail`, or similar persistent datasets.

---

## 3. Prepare the installer USB

Download:

```text
FreeBSD-15.1-RELEASE-amd64-memstick.img
```

Official SHA256 for the uncompressed amd64 memstick image:

```text
26c6d5de1156e7a99df920d047578b504c4899865d91b1156b1de1a0c5239eae
```

From another Unix-like system, write the image to a USB drive:

```sh
dd if=FreeBSD-15.1-RELEASE-amd64-memstick.img of=/dev/daX bs=4M conv=sync status=progress
sync
```

Replace `/dev/daX` with the USB device, **not** a partition.

---

## 4. Firmware setup

In the laptop firmware setup:

```text
Boot mode:        UEFI
Storage mode:     AHCI / native NVMe, not Intel RST / firmware RAID
Secure Boot:      disabled
Boot order:       USB installer first
```

Do not configure anything specifically for suspend/resume.

---

## 5. Boot the installer

Boot the FreeBSD USB memstick.

Choose:

```text
Install
```

Proceed through:

```text
Keymap
Hostname
```

Use a normal hostname, for example:

```text
tuxedo-freebsd
```

---

## 6. Select package-base install

When the installer asks for the installation type, choose:

```text
Packages (Tech Preview)
```

This is the package-base/pkgbase path. It installs the base system as packages and later allows base-system updates through `pkg upgrade` together with third-party packages.

When selecting components, use:

```text
lib32       yes
src         yes
ports       optional
```

Recommended selection for this laptop:

```text
lib32
src
```

Skip these unless you specifically need them:

```text
base-dbg
kernel-dbg
lib32-dbg
tests
ports
```

Because this is package-base, configure networking when prompted. The installer may need repository access.

---

## 7. Select partitioning method

At disk partitioning, choose:

```text
Auto (ZFS)
```

Do **not** choose Shell partitioning for this final setup. Auto ZFS is the safer option here because it creates the normal FreeBSD ZFS boot-environment layout used by `bectl`.

---

## 8. Identify the two target disks

Use the installer’s Disk Info screen if needed.

Common disk names for NVMe disks on FreeBSD are:

```text
nda0
nda1
```

or sometimes:

```text
nvd0
nvd1
```

In this runbook, examples assume:

```text
nda0
nda1
```

Replace those names if the installer shows different names.

You must select the two internal 4 TB disks. Do not proceed unless you are certain.

---

## 9. Configure Auto ZFS

In the ZFS configuration menu, set the following exactly.

### 9.1 Pool Type / Disks

Choose:

```text
Pool Type / Disks:    mirror
Disks:                nda0 nda1
```

Replace `nda0 nda1` with the actual two internal disk names.

Expected result:

```text
zroot mirror over two encrypted GELI providers
usable raw capacity around one disk
one disk may fail without immediate data loss
```

### 9.2 Pool Name

Set:

```text
Pool Name:    zroot
```

### 9.3 Force 4K Sectors

Set:

```text
Force 4K Sectors:    YES
```

This should produce:

```text
ashift=12
```

### 9.4 Encryption

Set:

```text
Encrypt Disks:    YES
```

This enables GELI encryption below ZFS.

Enter a strong GELI passphrase when prompted.

With this design, ZFS sees only encrypted `.eli` devices and does not know about encryption.

### 9.5 Partition Scheme

Set:

```text
Partition Scheme:    GPT / UEFI
```

### 9.6 Swap

Set:

```text
Swap Size:    0
```

If the menu still exposes these options, leave them disabled or irrelevant:

```text
Mirror Swap:     NO
Encrypt Swap:    NO
```

There must be no swap partition and no swap entry in `/etc/fstab`.

### 9.7 Boot Pool

Because root is encrypted with GELI below ZFS, FreeBSD needs an unencrypted boot pool.

Expected boot pool:

```text
bootpool
```

If the installer exposes boot pool size, set:

```text
Boot Pool Size:    4G
```

If it does not expose boot pool size, accept the default.

### 9.8 ZFS Pool Options

Open:

```text
ZFS Pool Options
```

Enter exactly:

```sh
-O compression=zstd-10 -O atime=off -O dedup=sha256
```

Do not omit the `-O` prefixes.

Do not add native ZFS encryption.

Do not add a manual ARC limit.

Do not set a global custom `recordsize`.

---

## 10. Final installer checklist before selecting Install

Before selecting `Install`, the Auto ZFS menu should effectively show:

```text
Pool Type / Disks:      mirror: nda0 nda1
Pool Name:              zroot
Force 4K Sectors:       YES
Encrypt Disks:          YES
Partition Scheme:       GPT / UEFI
Swap Size:              0
ZFS Pool Options:       -O compression=zstd-10 -O atime=off -O dedup=sha256
```

Then select:

```text
Install
```

Confirm disk destruction only after verifying again that the selected disks are exactly the two internal 4 TB drives.

---

## 11. Let installation proceed

Let the installer perform these actions:

```text
partition both disks
create EFI partitions
create the unencrypted bootpool
initialize GELI
create zroot on .eli providers
create the standard FreeBSD ZFS dataset layout
install FreeBSD using Packages (Tech Preview)
```

Continue through:

```text
Root password
Network setup
Time zone
Services
Hardening
User creation
```

Use:

```text
Time zone: Europe/Warsaw
```

Services: select only what you need. A reasonable baseline is:

```text
sshd        optional
ntpd/ntp    yes, if offered and you want system time sync
powerd      optional
```

Do not enable anything specifically for suspend/resume.

---

## 12. Final shell before first reboot

At the end of installation, when the installer asks whether to open a shell in the installed system for final manual modifications, choose:

```text
Yes
```

Run the following block in that final shell.

This block:

```text
enables ZFS at boot
keeps no swap
sets /tmp to tmpfs size 512m
disables and removes the ZFS /tmp and /var/tmp datasets when possible
makes /var/tmp point to /tmp
re-asserts zstd-10, sha256, and atime=off on zroot datasets
leaves ARC defaults untouched
```

Copy and paste:

```sh
set -eux

# Enable ZFS at boot.
sysrc zfs_enable=YES

# Back up fstab.
cp /etc/fstab /etc/fstab.before-final-tmp-noswap

# Remove any existing /tmp, /var/tmp, or swap entries.
awk '($2 != "/tmp") && ($2 != "/var/tmp") && ($3 != "swap") { print }' \
  /etc/fstab.before-final-tmp-noswap > /etc/fstab

# Mount /tmp as volatile tmpfs, capped at 512 MiB.
cat >> /etc/fstab <<'EOF_FSTAB'
tmpfs   /tmp    tmpfs   rw,mode=1777,size=512m   0   0
EOF_FSTAB

# Load tmpfs at boot if needed.
grep -q '^tmpfs_load=' /boot/loader.conf 2>/dev/null || \
  echo 'tmpfs_load="YES"' >> /boot/loader.conf

# Re-assert root-pool ZFS properties across all zroot filesystems.
# The installer pool options should already have applied before package installation.
# This makes the final state explicit, including child datasets such as zroot/var/mail.
for ds in $(zfs list -H -o name -t filesystem -r zroot); do
  zfs set compression=zstd-10 "$ds"
  zfs set atime=off "$ds"
  zfs set dedup=sha256 "$ds"
done

# Keep bootpool simple for loader compatibility.
zfs set compression=off bootpool 2>/dev/null || true
zfs set dedup=off bootpool 2>/dev/null || true
zfs set atime=off bootpool 2>/dev/null || true

# Disable the installer-created temporary datasets.
# They are normal ZFS datasets, not pools.
for ds in zroot/tmp zroot/var/tmp; do
  if zfs list -H "$ds" >/dev/null 2>&1; then
    zfs set canmount=off "$ds" 2>/dev/null || true
    zfs set mountpoint=none "$ds" 2>/dev/null || true
    zfs unmount -f "$ds" 2>/dev/null || true
  fi
done

# Try to destroy the temporary datasets before first boot.
# If one is busy, it will be cleaned up after first boot by the post-install cleanup step.
for ds in zroot/tmp zroot/var/tmp; do
  if zfs list -H "$ds" >/dev/null 2>&1; then
    zfs destroy "$ds" 2>/dev/null || true
  fi
done

# Create the underlying /tmp directory inside the root filesystem.
mkdir -p /tmp
chmod 1777 /tmp

# If /var/tmp is not covered by a mounted dataset, make it a symlink to /tmp.
# If this fails because the old dataset is still mounted/busy, run the post-first-boot cleanup section later.
if ! zfs list -H zroot/var/tmp >/dev/null 2>&1; then
  rm -rf /var/tmp
  ln -s /tmp /var/tmp
fi

# Confirm no ARC limits were added.
grep -E '^vfs\.zfs\.arc\.' /boot/loader.conf 2>/dev/null && \
  echo 'WARNING: ARC limit found in /boot/loader.conf; remove it if you want defaults.' || true

# Show final fstab and dataset state.
echo '--- /etc/fstab ---'
cat /etc/fstab

echo '--- zfs list ---'
zfs list -o name,mountpoint,compression,dedup,atime
```

Expected `/etc/fstab` should contain the tmpfs line:

```fstab
tmpfs   /tmp    tmpfs   rw,mode=1777,size=512m   0   0
```

Expected `/etc/fstab` should **not** contain any swap line.

If `zroot/tmp` or `zroot/var/tmp` still exists at this point, continue anyway. The post-first-boot cleanup section below will handle it.

Exit the final shell:

```sh
exit
```

Let the installer finish. Remove the USB memstick when prompted and reboot.

---

## 13. First boot

At boot, you should get a GELI passphrase prompt.

Enter the GELI passphrase.

Expected boot chain:

```text
UEFI firmware
  -> EFI loader
  -> unencrypted bootpool
  -> GELI unlock prompt
  -> import encrypted zroot from .eli mirror members
  -> boot FreeBSD from zroot/ROOT/default
```

---

## 14. First-boot verification

Log in as root or your admin user.

### 14.1 Verify FreeBSD version

```sh
freebsd-version -kru
uname -a
```

Expected:

```text
15.1-RELEASE
```

### 14.2 Verify pools

```sh
zpool list
zpool status
```

Expected pools:

```text
bootpool
zroot
```

Expected `zroot` topology should contain a mirror over `.eli` devices, for example:

```text
zroot
  mirror-0
    nda0pXpY.eli
    nda1pXpY.eli
```

Exact partition names may differ. The important part is that `zroot` uses `.eli` providers.

### 14.3 Verify GELI

```sh
geli status
```

Expected: two attached encrypted providers for the root pool.

### 14.4 Verify 4K sectors / ashift

```sh
zdb -C zroot | grep ashift
zdb -C bootpool | grep ashift
```

Expected:

```text
ashift: 12
```

### 14.5 Verify ZFS properties

```sh
zfs get compression,dedup,atime -r zroot
```

Expected for persistent `zroot` datasets:

```text
compression    zstd-10
dedup          sha256
atime          off
```

### 14.6 Verify `/tmp` and `/var/tmp`

```sh
mount | grep ' /tmp '
df -h /tmp
ls -ld /tmp /var/tmp
readlink /var/tmp || true
```

Expected:

```text
tmpfs mounted on /tmp
/tmp size approximately 512M
/var/tmp -> /tmp
```

### 14.7 Verify no swap

```sh
swapinfo -h
grep -i swap /etc/fstab || true
```

Expected:

```text
no swap devices
no swap line in /etc/fstab
```

### 14.8 Verify package-base

```sh
pkg info | grep '^FreeBSD-' | head
```

Expected: FreeBSD base packages, for example `FreeBSD-runtime`, `FreeBSD-kernel-*`, or similar package-base entries.

### 14.9 Verify boot environments

```sh
bectl list
```

Expected initial boot environment, usually:

```text
default
```

Optional test:

```sh
bectl create test-be
bectl list
bectl destroy test-be
```

If this works, the standard FreeBSD ZFS boot-environment layout is intact.

---

## 15. Post-first-boot temporary dataset cleanup

Run this after the first successful boot.

It ensures there are no persistent ZFS temp datasets and that all normal temp paths point to the `/tmp` tmpfs.

```sh
set -eux

# Ensure /tmp is mounted as tmpfs before redirecting /var/tmp.
mount | grep ' /tmp '

# Disable, unmount, and destroy the installer-created temp datasets if they still exist.
for ds in zroot/tmp zroot/var/tmp; do
  if zfs list -H "$ds" >/dev/null 2>&1; then
    zfs set canmount=off "$ds" 2>/dev/null || true
    zfs set mountpoint=none "$ds" 2>/dev/null || true
    zfs unmount -f "$ds" 2>/dev/null || true

    if zfs destroy "$ds" 2>/dev/null; then
      echo "Destroyed $ds"
    else
      echo "Could not destroy $ds directly; checking snapshots"
      zfs list -t snapshot -r "$ds" || true
      echo "Dry run recursive destroy for $ds:"
      zfs destroy -nrv "$ds" || true
      echo "If the dry run only lists temporary-dataset snapshots you accept deleting, run:"
      echo "  zfs destroy -rv $ds"
    fi
  fi
done

# Recreate /tmp mountpoint permissions.
mkdir -p /tmp
chmod 1777 /tmp

# Make /var/tmp volatile by pointing it at /tmp.
# Do not use /tmp/var; that would require separate boot-time directory creation.
rm -rf /var/tmp
ln -s /tmp /var/tmp

# Final verification.
mount | grep ' /tmp '
df -h /tmp
ls -ld /tmp /var/tmp
readlink /var/tmp
zfs list -H -o name | egrep '^(zroot/tmp|zroot/var/tmp)$' || echo 'OK: no ZFS temp datasets remain'
```

Final desired temp layout:

```text
/tmp                 tmpfs, 512M, volatile
/var/tmp             symlink to /tmp
zroot/tmp            absent
zroot/var/tmp        absent
all other datasets   unchanged
```

Do not redirect or delete these as “temporary” paths:

```text
/var/log
/var/mail
/var/crash
/var/audit
/var/db
/var/cache
/var/spool
/usr/src
/usr/ports
/usr/ports/distfiles
/usr/obj
```

Those are not equivalent to `/tmp` and should remain persistent unless you later make a deliberate workload-specific change.

---

## 16. Optional but recommended: duplicate EFI loader to both ESPs

Auto ZFS should install boot files, but EFI System Partitions are FAT partitions, not ZFS mirrors. On a two-disk mirror laptop, keep a loader on both disks.

First inspect partition layout:

```sh
gpart show -p
```

Usually the EFI partitions are the first partition on each disk, for example:

```text
/dev/nda0p1
/dev/nda1p1
```

Adjust the variables below if your EFI partitions differ:

```sh
set -eux

ESP0=/dev/nda0p1
ESP1=/dev/nda1p1

mkdir -p /tmp/esp0 /tmp/esp1

mount -t msdosfs "$ESP0" /tmp/esp0
mkdir -p /tmp/esp0/EFI/BOOT
mkdir -p /tmp/esp0/EFI/FreeBSD
cp -p /boot/loader.efi /tmp/esp0/EFI/BOOT/BOOTX64.EFI
cp -p /boot/loader.efi /tmp/esp0/EFI/FreeBSD/loader.efi
umount /tmp/esp0

mount -t msdosfs "$ESP1" /tmp/esp1
mkdir -p /tmp/esp1/EFI/BOOT
mkdir -p /tmp/esp1/EFI/FreeBSD
cp -p /boot/loader.efi /tmp/esp1/EFI/BOOT/BOOTX64.EFI
cp -p /boot/loader.efi /tmp/esp1/EFI/FreeBSD/loader.efi
umount /tmp/esp1
```

Repeat this after future base-system upgrades if `/boot/loader.efi` changes.

---

## 17. GELI metadata backup

Do this after confirming the system boots correctly.

First identify the attached providers:

```sh
geli status
```

Example output may include:

```text
nda0p3.eli
nda1p3.eli
```

The underlying devices are then:

```text
/dev/nda0p3
/dev/nda1p3
```

Adjust the variables below before running:

```sh
set -eux

GELI0=/dev/nda0p3
GELI1=/dev/nda1p3

mkdir -m 700 /root/geli-backup

geli backup "$GELI0" /root/geli-backup/$(basename "$GELI0").geli
geli backup "$GELI1" /root/geli-backup/$(basename "$GELI1").geli

cp -p /boot/encryption.key /root/geli-backup/ 2>/dev/null || true

tar -C /root -cpf /root/geli-backup-freebsd151-tuxedo.tar geli-backup
chmod 600 /root/geli-backup-freebsd151-tuxedo.tar
```

Copy this archive to external encrypted storage.

Do not leave the only copy on the laptop.

---

## 18. Operational checks

### Dedup monitoring

You intentionally enabled `dedup=sha256`. Monitor it:

```sh
zpool list zroot
zpool status -D zroot
```

Useful values:

```text
DEDUP ratio
DDT histogram
```

The ratio will probably start near:

```text
1.00x
```

That is normal immediately after install.

### Compression monitoring

```sh
zfs get compressratio,logicalused,used -r zroot
```

Expected benefit is workload-dependent:

```text
source code, logs, text, build trees:    usually good compression
ROMs, archives, videos, images, ISOs:    often already compressed, limited gain
```

### Scrubs

Run periodic scrubs:

```sh
zpool scrub zroot
zpool scrub bootpool
zpool status
```

### Boot environments

Before major changes:

```sh
bectl create before-big-change
bectl list
```

Rollback/switching behavior can be handled with `bectl` as usual for FreeBSD ZFS boot environments.

---

## 19. Expected final state checklist

After all steps:

```sh
zpool list
zpool status
```

Expected:

```text
bootpool    ONLINE, mirror
zroot       ONLINE, mirror over .eli providers
```

```sh
zfs list -H -o name | sort
```

Expected: standard FreeBSD datasets under `zroot`, but no temp datasets:

```text
bootpool
zroot
zroot/ROOT
zroot/ROOT/default
zroot/home
zroot/usr
zroot/usr/ports
zroot/usr/src
zroot/var
zroot/var/audit
zroot/var/crash
zroot/var/log
zroot/var/mail
```

`zroot/tmp` and `zroot/var/tmp` should be absent.

```sh
mount | grep ' /tmp '
df -h /tmp
ls -ld /var/tmp
readlink /var/tmp
```

Expected:

```text
/tmp is tmpfs, approximately 512M
/var/tmp -> /tmp
```

```sh
swapinfo -h
grep -i swap /etc/fstab || true
```

Expected:

```text
no swap
```

```sh
zdb -C zroot | grep ashift
```

Expected:

```text
ashift: 12
```

```sh
zfs get compression,dedup,atime -r zroot
```

Expected persistent dataset properties:

```text
compression    zstd-10
dedup          sha256
atime          off
```

```sh
bectl list
```

Expected:

```text
working FreeBSD boot environments
```

---

## 20. Source notes

This runbook was prepared against FreeBSD 15.1-era official documentation:

- FreeBSD 15.1-RELEASE announcement and checksums.
- FreeBSD Handbook, Installing FreeBSD, including package-base / Packages Tech Preview and Auto ZFS.
- `bsdinstall(8)`, including `zfsboot`, `ZFSBOOT_POOL_CREATE_OPTIONS`, `ZFSBOOT_FORCE_4K_SECTORS`, `ZFSBOOT_GELI_ENCRYPTION`, boot pool behavior, swap size, and default ZFS datasets.
- FreeBSD Handbook ZFS chapter, including Zstd and deduplication behavior.
- `zfsprops(7)`, including `compression`, `dedup`, and `atime` properties.
- `tmpfs(5)`, including tmpfs volatility and use for `/tmp`.
- `geli(8)`, including encrypted root support and metadata backup.
