# fedora-zswap-setup-skill

Replace Fedora's default zram swap with zswap plus a disk-backed swap file, so
heavy memory pressure has a disk-backed safety net. It reduces OOM kills and
multi-minute stalls; it guarantees neither.

> [!WARNING]
> This runbook edits `/etc/fstab`, changes the kernel command line, removes a
> package, and reboots. A bad `/etc/fstab` entry can drop the machine to an
> emergency shell. Read the prerequisite checks first, keep the rollback section
> handy, and make sure there is no unsaved work before rebooting.

## The problem this solves

Fedora configures swap in compressed RAM via zram, sized `min(ram, 8192)`. That
capacity is thin-provisioned: the device advertises its full size but can only
hold what fits in real RAM once pages are compressed, and there is no disk behind
it. Workloads on a memory-constrained machine therefore reach a hard ceiling with
little gradual degradation, and the kernel responds either by stalling or by
terminating processes.

zswap keeps a compressed pool in RAM that is backed by a real swap file, writing
colder pages out to disk before the pool fills. Total capacity is bounded by free
disk, so the kernel has somewhere to put cold pages. Under extreme pressure it
can still stall or OOM-kill: if the resident set exceeds RAM plus swap, or an
allocation burst outruns reclaim, no swap configuration can save the workload.

Both designs are reasonable. Fedora's keeps swapped data off persistent storage
entirely, which is the right trade-off for a great many systems. This
configuration prioritises keeping running applications alive instead, at the
cost of writing swapped pages to disk.

## Security trade-off

zram keeps swapped pages in RAM, so secrets that get swapped never touch
persistent storage. A disk swap file writes those pages to disk. On a default
Fedora install with LUKS full-disk encryption, the swap file is inside the
encrypted volume, so this is mostly covered. On an unencrypted disk, swap can
contain passwords, keys, and session tokens; use encrypted swap or accept that
trade-off.

## What it configures

| | |
|---|---|
| Kernel parameters | `zswap.enabled=1 zswap.compressor=zstd zswap.max_pool_percent=40` |
| Swap file | `/swap/swapfile`, 16 GiB, registered in `/etc/fstab` |
| zswap shrinker | Enabled, so cold pages spill to disk before the pool fills |
| zram | Disabled: `zram-generator-defaults` removed; `zram-generator` may stay with `zram-size = 0` |
| `systemd-oomd` | Left enabled; part of the safety net |
| `vm.swappiness` | Left at Fedora's default of 60 |

Keep `systemd-oomd` enabled. It watches memory pressure and can kill a runaway
cgroup before the kernel OOM killer fires; zswap and oomd solve different halves
of the problem.

## Getting started

Everything lives in **[`SKILL.md`](SKILL.md)**: prerequisite checks, the commands
in order, the failure modes worth knowing about, post-reboot verification, and a
full rollback procedure.

It is written as an [agent skill](https://code.claude.com/docs/en/skills), so the
file can be handed to an AI coding agent to carry out the setup, and it reads as
an ordinary runbook if you would rather work through it yourself. The Apply block
requires explicit approval: it removes a package, changes `/etc/fstab` and the
kernel command line, and reboots.

Start with the prerequisite and current-state checks at the top. They are
read-only, and they establish whether this configuration suits your machine
before anything is modified. `/etc/fstab` and the kernel command line are the two
things that change, and the rollback section returns both to Fedora's defaults.
The first check is whether swap is actually the problem: if one app is simply
too large, cap it or run fewer heavy apps; a bigger swap file will not fix that.

Requires Fedora with `grubby`, a kernel exposing the zswap shrinker, a Btrfs,
ext4 or xfs root on a non-rotating disk, and roughly 16 GiB of free space.
Validated on Fedora 44/45, kernel 7.1.9/7.3.3, Btrfs on NVMe, 8 GiB of RAM.

## Scope

A personal configuration, written down so the research behind it does not need to
be repeated. It targets Fedora specifically and does not attempt to cover other
distributions, filesystems or storage types, since those involve materially
different trade-offs.

Published as-is under the MIT license. Fork it and adapt it to your own setup
freely.

## Credits

With huge thanks to Chris Down, who maintains memory management in the Linux
kernel, for [zswap vs zram: when to use
what](https://chrisdown.name/2026/03/24/zswap-vs-zram-when-to-use-what.html). The
reasoning behind this configuration comes from that article, and it is worth
reading before `SKILL.md`.

Further reference: the [kernel zswap
documentation](https://docs.kernel.org/admin-guide/mm/zswap.html).

---

<p align="center">
	<img src="https://raw.githubusercontent.com/ITx-prash/floweave/main/assets/coder.png" height="150" alt="Coder illustration"/>
	<br/>
	<em>Crafted with 💚 on GNU/Linux</em>
	<br/>
	Copyright &copy; 2026-present <a href="https://github.com/ITx-prash" target="_blank">Prashant Adhikari</a>
	<br/><br/>
	<a href="LICENSE"><img src="https://img.shields.io/static/v1.svg?style=for-the-badge&label=License&message=MIT&logoColor=a6e3a1&colorA=1e1e2e&colorB=a6e3a1"/></a>
</p>
