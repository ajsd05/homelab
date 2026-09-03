# Lab Journal

Append-only log. Newest entry at the top. Every change, every failure, every fix.

**Write the entry when the thing happens, not later.** A three-line entry written the same
day beats a polished one written from memory next week, and the details that make an entry
worth reading are the first ones to fade.

**Entries are allowed to be short.** "Rebooted the host, everything came back" is a valid
entry. The value is in the unbroken record, not in any single line.

---

## Format

```markdown
## YYYY-MM-DD — Short title

**Did:** what changed, in a sentence or two.

**Why:** the reason, if it isn't obvious.

**Broke:** what went wrong, if anything.

**Thought it was:** the initial diagnosis.

**Actually was:** the real cause.

**Time lost:** rough duration between "something is wrong" and "it is fixed."

**Next:** anything this created or exposed.
```

Drop any field that doesn't apply. The `thought it was` / `actually was` pair is the
important one — the gap between those two lines is the record of how diagnostic instinct
develops, and it's invisible in a git log.

---

# Entries

<!-- Newest first. Add above this line's successor, below the heading. -->

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
