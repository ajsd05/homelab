# Lab Journal

Append-only log. Newest entry at the top. Every change, every failure, every fix.

**Write the entry when the thing happens, not later.** A three-line entry written the same
day beats a polished one written from memory next week, and the details that make an entry
worth reading are the first ones to fade.

**Entries are allowed to be short.** "Rebooted the host, everything came back" is a valid
entry. The value is in the unbroken record, not in any single line.

---

# Entries

<!-- Newest first. Add above this line's successor, below the heading. -->

## 2026-09-06 — Wiped and mounted the FireCuda as lab storage

**What I did:** 
- Took the 2 TB into Proxmox host. Identified it as `/dev/sda` with
`lsblk -o NAME,SIZE,TYPE,TRAN,MODEL,FSTYPE,MOUNTPOINT`
- wiped the old NTFS partition table with `wipefs -a`
- wrote a GPT table and one full-disk partition with `parted`, formatted it
ext4 with `mkfs.ext4 -L media`
- mounted it at `/mnt/media`, and added an fstab entry keyed on the filesystem UUID with `nofail`.
- Tested the fstab line with `umount` + `mount -a` rather than by rebooting, then ran `systemctl daemon-reload`.

**Why:** I need storage for a Jellyfin library, and the obvious option carving space out of
the LVM thin pool is the wrong one. It would put my biggest, least valuable data on my most
constrained and least protected storage, competing with every future VM for the same 794 GB,
on a single disk with no backup. A second physical drive solves the media problem *and* gives
me somewhere to write Proxmox backups that isn't the disk being backed up. One drive I
already owned, two gaps closed.

**What broke / what surprised me:**

- `parted` wasn't installed. `command not found` — Proxmox ships a fairly minimal Debian.
  Nothing was broken; I just needed `apt install parted`.
- I ran `wipefs -a` before running the read-only `wipefs` preview. Wrong order. It worked out
  because the path was right, but a preview is worthless after the fact.
- After creating the new partition, `lsblk` still reported `FSTYPE=ntfs` on `sda1`, which
  looked like the wipe had failed. It hadn't. `wipefs` erased the *partition table* (2 bytes,
  `55 aa`, at offset 0x1fe); the NTFS metadata was inside the old `sda1`, which stopped
  existing the moment the table died, so those bytes were never touched. My new partition
  starts at nearly the same offset, so `lsblk` read the stale signature and honestly reported
  what it saw. `mkfs.ext4` then warned me the partition "contains a ntfs file system labelled
  'Seagate'" — which was actually useful, because that Windows label was a fifth independent
  confirmation I was on the right disk.
- `mount -a` printed a hint that systemd was still using the old fstab. Not an error — the
  mount worked. systemd parses fstab at boot and *generates* mount units from it, so editing
  the file by hand leaves those units stale until `systemctl daemon-reload`.

**What I learned:**

- **Disk vs partition.** `/dev/sda` is the whole physical disk; `/dev/sda1` is a declared
  region inside it. The partition table at the front of the disk is the map saying which
  regions exist. You wipe the disk to destroy the map; you format the partition to create a
  filesystem inside one region. I got this backwards when asked and would have wiped the wrong
  thing.
- **Partition vs filesystem.** A partition says *where*; a filesystem says *how it's
  organized* — the structure tracking files, directories, permissions, and which blocks hold
  which bytes. Two separate steps, `parted` then `mkfs`.
- **GPT vs MBR.** GPT is the modern partition table format. MBR can't address past 2 TB and
  caps at four primary partitions. My drive is exactly at that boundary, so GPT.
- **Removing the pointer to data isn't removing the data.** This is why "deleted" files are
  recoverable, why a quick format takes a second and a full one takes hours, and why I would
  not hand this drive to a stranger after only doing what I did.
- **Mounting.** Windows gives each filesystem a letter. Linux has one tree from `/`, and a
  filesystem gets attached to a directory inside it — that's a mount, and the directory is
  the mount point. After mounting, `/mnt/media` is a doorway into the FireCuda, not a folder
  on the NVMe.
- **The shadowing trap.** If I write files into `/mnt/media` while nothing is mounted there,
  they land on `pve-root` (96 GB) and then become invisible once the real drive mounts over
  the top. Not deleted — shadowed, and still eating my system partition. Check `df -h` before
  writing.
- **Mount by UUID, not `/dev/sda1`.** Device names are handed out in the order the kernel
  finds disks, so they shuffle when hardware changes. The filesystem UUID is written inside
  the filesystem and travels with it.
- **`nofail` is mandatory for a USB disk in fstab.** Without it, a disk that's unplugged or
  slow to spin up can drop the host into an emergency shell at boot — and a ThinkCentre Tiny
  has no iDRAC/iLO, so that means walking over with a keyboard and monitor.
- **`mount -a` is the dry run for a reboot.** Test fstab at a shell with the machine still up,
  not by rebooting and hoping.
- **ext4's journal** logs intended changes before making them, so an interruption leaves the
  filesystem repairable. It protects the filesystem's structure, not the contents of the file
  being written — which matters because I have no UPS and every power flicker is a hard cut.
- **Superblock backups.** The superblock is the master record describing the filesystem; lose
  it and the disk is unreadable even though every byte is still there. ext4 scatters copies
  across the disk, and `mkfs` printed their block numbers. That's what you point `fsck` at.
- **Inodes** are per-file metadata records, allocated at a fixed count when the filesystem is
  created. A filesystem can run out of inodes while showing free space.
- **A `(y,N)` prompt with a capital N defaults to no.** Tools that can destroy data are built
  that way.
- I also had the rule for this drive backwards at first. I said "only re-downloadable stuff,
  nothing important." The real axis isn't important vs unimportant, it's **is this the only
  copy?** Backups belong on this drive precisely *because* they're second copies — that's the
  mitigation, not the risk. What doesn't belong is anything whose only copy would live here.

**New terms:** partition table, GPT vs MBR, filesystem, ext4, journal, superblock (and
superblock backups), inode, mount / mount point, fstab, UUID vs PARTUUID, filesystem label,
`nofail`, systemd generated mount units.

**Next step:** Create `/mnt/media/library` and `/mnt/media/backups`, then register `/mnt/media`
with Proxmox as a storage target — the mount alone doesn't close the backup gap; PVE has to be
told the storage exists. After that, one reboot test proves three things at once: LXC 101's
`onboot`, the fstab mount, and remote access from cellular. Then the Jellyfin container.

## 2026-09-06 - Set up Tailscale reboot-safe before standing up any server
**Did:**
- pct config 101 and found no onboot line
- Set it with pct set 101 -onboot 1
- Ran pct listsnapshot 101 — only current; no snapshots existed, took one named tailscale-working.

**Why:**
- Setting up a new service on Proxmox will eventually lead to a reboot, and we want all our data to be saved if I am working remotely
- This saves the hassle of having to go back into the server to restart any container.

## 2026-09-02 - Set up Tailscale subnet router in a linux container
**Did:** 
- Set up tailscale as a LXC in proxmox.
- Fixed apt on host. It was resolving the download link to IPv6 and my box has no IPv6, forced IPv4 with 'Acquire::ForceIPv4'

**Why:**
- So i can access my homelab from outside my home.

**Broke:** 
- Downloaded the arm64 template by mistake. The container built fine and failed at the
  very last step with "Exec format error" on /sbin/init. Had to pct destroy and rebuild.
- Ran the sysctl commands twice — once in the container, then again on the host after I'd
  exited without noticing. Had to undo it on the host.
- The first `tailscale up --advertise-routes` didn't register. Needed --reset.
- The admin console kept showing "This machine does not expose any routes" even though
  `tailscale debug prefs` showed AdvertiseRoutes was set. Restarting the container made
**What I learned:**
- Read logs by jumping to the first ERROR. Everything above it succeeded by definition.
  `pct start 101 --debug 2>&1 | grep -i error` beats scrolling.
- "Exec format error" means wrong CPU architecture.
- Read the prompt before pressing enter. root@tailscale and root@proxmox are different
  machines
- Advertising a route isn't having one. A human has to approve it, because a node claiming
  a whole subnet is a big claim.
- Prefs saved locally aren't the same as state pushed to the control plane.
- Hard-refresh first. If the UI still disagrees with what you expect, the state is probably
  genuinely wrong, not just displayed wrong.
- Restart the service, not the whole container. Restarting the container took down
  everything in it to fix one daemon — fine today with one service, not fine later.

**New terms:**
- **LXC vs VM** — container shares the host's kernel, VM boots its own. Tailscale reported
  my kernel as 6.17.2-2-pve, the host's, which proves it.
- **Unprivileged container** — root inside maps to a harmless high-numbered user on the
  host. Why /dev/net/tun showed as `nobody nogroup`.
- **TUN** — kernel feature letting a program pretend to be a network card. What a VPN
  tunnel physically is.
- **Bind mount** — same file made visible at a second path. Not a copy.
- **cgroup device rule** — `c 10:200 rwm` allows character device 10:200 (TUN) with
  read/write/mknod.
- **Character vs block device** — streams bytes vs addresses fixed-size chunks.
- **amd64 / x86_64** vs **arm64 / aarch64** — two names each for two instruction sets.
- **Subnet router** — a Tailscale node that forwards traffic for a whole LAN.
- **100.64.0.0/10** — CGNAT range Tailscale uses so its addresses never collide with a
  home or hotel network.
- **ip_forward / sysctl** — kernel won't route packets between interfaces unless told to.
- 
**Next:** SSH keys to the Proxmox host and disable password auth — last gap before I can automate
anything. Then a DHCP reservation so the container stops being a moving target at
192.168.1.211, and a backup target.


## 2026-09-02 - Documentation is live
**Did:** Setup github repo and git in WSL. 

**Why:** I wanted to document my homelab progress in some way. I thought the best way was a journal. 

**Broke:** N/A

**Thought it was:**N/A

**Actually was:** N/A

**Time lost:**N/A

**Next:** I want to setup a VPN for my ProxMox server so I can access and work on it remotely.

## 2026-09-02 — Hardware audit and context rebuild

**Did:** Ran a full hardware inventory on the Proxmox node. Rewrote the context document
from verified output instead of assumptions. Established this repo: rules, journal,
decision records.

**Why:** Setup had gone untouched for months and the existing notes had drifted from
reality.

**Found:** Several documented facts were wrong. Disk is 931 GB, not "under 500 GB" —
which puts a media server back on the table. CPU is an i7-7700T (4c/8t, 35 W) in a Lenovo
ThinkCentre Tiny. Intel HD 630 with Quick Sync, so hardware transcoding is available.
VT-x confirmed active; DMAR tables present so VT-d exists in hardware.

**Exposed:** No backup target of any kind. Single NVMe, no redundancy, nothing offsite.
`lm-sensors` not installed, so no thermal data on a small-chassis 35 W machine. Unconfirmed
whether the router holds a DHCP reservation for 192.168.1.50.

**Next:** External backup target before anything of value gets built. Then first LXC.

---

## Example entry — delete this once there are real ones

## 2026-09-09 — Prometheus container refused to start after host reboot

**Did:** Rebooted the host to apply a kernel update.

**Broke:** CT 110 stayed down. Grafana showed a gap in every metric.

**Thought it was:** Corrupted container after an unclean shutdown.

**Actually was:** `onboot` was never set on the container, so it simply wasn't asked to
start. It had been started by hand at creation and had never survived a reboot because it
had never seen one.

**Time lost:** ~40 minutes, most of it looking in the wrong place.

**Next:** Audit `onboot` on every existing guest. Add "starts on boot" to the deployment
checklist — a service that doesn't survive a reboot isn't deployed.
