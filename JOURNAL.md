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
