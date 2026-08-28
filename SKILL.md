---
name: fedora-zswap-setup-skill
description: >-
  Replace Fedora's default zram swap with zswap plus a disk swap file, so heavy
  memory pressure causes slowness instead of OOM-killed apps. Use when a Fedora
  machine's OOM killer is terminating editors, browsers or containers, or when
  asked to enable zswap, disable zram, or add a swap file on Btrfs/ext4/xfs.
license: MIT
metadata:
  version: "3.0.0"
  updated: "2026-08-28"
  validated_on: "Fedora 44, kernel 7.1.9, Btrfs on NVMe, 8GiB RAM"
---

# Fedora: zswap + disk swap file

Fedora ships zram: swap in compressed RAM, sized `min(ram, 8192)`, with no disk
behind it. That size is thin-provisioned, a bet on your pages compressing well.
When the bet loses there is nowhere to go, so the machine either stalls for
minutes or the OOM killer picks a victim.

zswap is a compressed pool in RAM *backed by a real swap file*. Cold pages are
written out to disk before the pool fills, so capacity is bounded by the disk,
not by RAM.

**Read this for the full picture:** Chris Down (kernel memory-management
maintainer), [zswap vs zram: when to use
what](https://chrisdown.name/2026/03/24/zswap-vs-zram-when-to-use-what.html).
Covers thin provisioning, LRU inversion, and why Fedora's zram default is a
deliberate choice rather than a mistake.

Target config, validated on Fedora 44 / kernel 7.1.9 / Btrfs on NVMe / 8GiB RAM:

| | |
|---|---|
| kernel args | `zswap.enabled=1 zswap.compressor=zstd zswap.max_pool_percent=40` |
| swap file | `/swap/swapfile`, 16GiB, in `/etc/fstab` |
| shrinker | `Y` (already the Fedora kernel default) |
| zram | removed |
| `vm.swappiness` | untouched, Fedora's 60 |

## Before doing anything

```bash
# Is there a real problem? No OOM kills means do not do this.
journalctl -k --grep "Out of memory" | tail

# Is it already done? enabled=Y with compressor=zstd means stop, nothing to do.
grep . /sys/module/zswap/parameters/* | sed 's|.*parameters/||'
swapon --show; zramctl

# Prerequisites.
. /etc/os-release; echo "$ID $VERSION_ID"; uname -r    # Fedora
ls /sys/module/zswap/parameters/shrinker_enabled       # must exist (landed in 6.8)
findmnt -n -o SOURCE,FSTYPE --target /                 # btrfs, ext4 or xfs
lsblk -no ROTA "$(findmnt -n -o SOURCE --target / | sed 's/\[.*\]$//')"   # 0 = SSD
df -h /                                                # ~16GiB free
command -v grubby
btrfs --version                                        # Btrfs only: need >= 6.1.3
```

If a prerequisite fails, stop and report it. Do not adapt around it.

## Apply

Order matters: the swap file has to work before the reboot.

```bash
# 1. Swap file. On Btrfs this one command handles NOCOW, sizing and mkswap.
#    Do not hand-roll it (see Gotchas).
sudo mkdir -p /swap
sudo btrfs filesystem mkswapfile -s 16g /swap/swapfile     # Btrfs
# ext4/xfs instead:
#   sudo fallocate -l 16G /swap/swapfile && sudo chmod 600 /swap/swapfile \
#     && sudo mkswap /swap/swapfile

# 2. Persist it.
sudo cp -a /etc/fstab /etc/fstab.bak
echo '/swap/swapfile none swap defaults 0 0' | sudo tee -a /etc/fstab

# 3. Activate NOW, before rebooting. A bad fstab entry discovered at boot means
#    an emergency shell. If this fails, delete the line from /etc/fstab and stop.
sudo swapon --all && swapon --show

# 4. Remove zram: the package and the override, because the package can return
#    as a weak dependency of a later dnf transaction.
sudo swapoff /dev/zram0 2>/dev/null
sudo systemctl stop systemd-zram-setup@zram0.service 2>/dev/null
sudo dnf remove -y zram-generator-defaults
printf '[zram0]\nzram-size = 0\n' | sudo tee /etc/systemd/zram-generator.conf

# 5. Keep the shrinker on across kernels that do not default it on.
echo 'w /sys/module/zswap/parameters/shrinker_enabled - - - - Y' \
  | sudo tee /etc/tmpfiles.d/zswap-shrinker.conf
sudo systemd-tmpfiles --create /etc/tmpfiles.d/zswap-shrinker.conf

# 6. Kernel args. ONE quoted string (see Gotchas). ALL so upgrades keep them.
sudo grubby --update-kernel=ALL \
  --args="zswap.enabled=1 zswap.compressor=zstd zswap.max_pool_percent=40"

sudo reboot
```

## Verify after the reboot

Kernel args do nothing until the reboot. **Do not report success before this
passes.**

```bash
grep -o 'zswap[^ ]*' /proc/cmdline          # all three args
grep . /sys/module/zswap/parameters/* | sed 's|.*parameters/||'
                                            # enabled:Y  compressor:zstd
                                            # max_pool_percent:40  shrinker_enabled:Y
swapon --show                               # /swap/swapfile, 16G
zramctl                                     # empty
grep zswap /etc/kernel/cmdline              # so future kernels inherit them
```

Then leave it alone for a few days. The claim is "no OOM kills under a real
workload", and only time tests that.

## Gotchas that cost real debugging time

- **`grubby --args` takes one string.** `--args a=1 b=2` binds only `a=1` and
  silently discards the rest. Exit code 0, no warning. Quote all three together.
- **Btrfs swap files must be NOCOW, and `chattr +C` only works on a zero-length
  file.** `btrfs filesystem mkswapfile` does it correctly. The subcommand landed
  in btrfs-progs 6.0.2, but `-s/--size` was fixed in 6.1.3, so treat 6.1.3 as
  the floor. Anything that runs `fallocate` before `chattr` produces a file
  `swapon` rejects with "Invalid argument". A `compress=zstd:1` root is fine,
  because NOCOW files are never compressed.
- **`swapon --all` before rebooting.** Otherwise a bad fstab line surfaces at
  boot, in an emergency shell.
- **The unit is `systemd-zram-setup@zram0.service`.** There is no
  `zram-swap.service`.
- **Fedora's kernel already sets `CONFIG_ZSWAP_SHRINKER_DEFAULT_ON=y`**, so the
  tmpfiles line is insurance, not the fix. It also ships
  `# CONFIG_ZSWAP_DEFAULT_ON is not set` and
  `CONFIG_ZSWAP_COMPRESSOR_DEFAULT="lzo"`, so `enabled=1` and `compressor=zstd`
  are both load-bearing. Check with `grep ZSWAP /boot/config-$(uname -r)`.
- **Do not set `zswap.zpool`.** Old guides say `z3fold` or `zbud`; both are
  removed from the upstream kernel. The default `zsmalloc` is correct.
- **Do not lower `vm.swappiness`.** Lowering it makes the kernel drop page cache
  instead of swapping, but the premise here is that swapping is now cheap.
  Fedora's 60 is what the validated machine runs.
- **Btrfs will not snapshot a subvolume holding an active swap file.** If
  snapper or timeshift covers `/`, give the file its own subvolume:
  `btrfs subvolume create /swapvol`, mount it at `/swap` via fstab, create the
  file there.
- **Hibernation:** an existing `resume=` partition keeps working. This swap file
  is not a hibernate target without `resume_offset=` and a new initramfs.

## On a machine with different RAM

Nothing here is tied to 8GiB.

- `max_pool_percent=40` is a percentage, it scales by itself. Leave it.
- The 16GiB file is sized to how far the workload overflows RAM, not to RAM.
  Ignore "swap = 2x RAM". Shrink it only if the disk is tight.
- More RAM means reaching the file less often, not needing a smaller one.

## Rollback

```bash
sudo swapoff /swap/swapfile          # pushes pages back to RAM, can be slow
sudo sed -i '\#^/swap/swapfile#d' /etc/fstab
sudo rm -f /swap/swapfile /etc/tmpfiles.d/zswap-shrinker.conf \
           /etc/systemd/zram-generator.conf
sudo grubby --update-kernel=ALL \
  --remove-args="zswap.enabled zswap.compressor zswap.max_pool_percent"
sudo dnf install -y zram-generator-defaults
sudo reboot
```

If a bad fstab entry ever drops you to an emergency shell:
`mount -o remount,rw /`, delete the `/swap/swapfile` line, reboot.

## Troubleshooting

| Symptom | Cause |
|---|---|
| Wrong compressor or pool percent | Not rebooted yet. Check `grep zswap /proc/cmdline`. |
| Args gone after a kernel upgrade | `/etc/kernel/cmdline` lacks them. Re-run step 6. |
| `swapon: Invalid argument` on Btrfs | File is CoW or compressed. Delete it, use `mkswapfile`. |
| zram came back | A dnf transaction pulled the weak dependency in. Confirm `/etc/systemd/zram-generator.conf` still exists. |
| Still getting kills | Find out who: `journalctl -k --grep "Out of memory"` is the kernel, `journalctl -u systemd-oomd` is userspace. If swap used is near 16GiB, the workload needs RAM, not swap. |
| Stalls for minutes instead of killing | Check `full avg10` in `/proc/pressure/memory`. Fast multi-GiB allocation bursts outrun any write-back path. |
