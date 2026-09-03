# Lab Rules

Standing conventions for this homelab. These exist so settled questions stay settled.

Rules change rarely. When one does change, note it in `JOURNAL.md` and — if the change
involved a real tradeoff — write a decision record in `decisions/`.

*Last revised: September 2026*

---

## 1. Guests

**LXC by default. VM only with a reason.**

Containers start in seconds, use a fraction of the RAM, and share the host kernel — which
matters a lot on a 16 GB box. Reach for a VM only when the workload genuinely needs it:
a different kernel, a non-Linux OS, Docker-in-a-guest with tricky requirements, or
anything doing kernel-level networking.

If a VM is used, the reason goes in the journal entry that created it.

**Unprivileged containers by default.** Privileged containers only when a specific device
passthrough requires it, and then it gets a decision record.

---

## 2. Naming and numbering

**Container/VM ID maps to the IP's last octet.** CT `110` lives at `192.168.1.110`. No
lookup table, no guessing, and `pct list` doubles as a network map.

**ID ranges:**

| Range | Purpose |
|---|---|
| 100–149 | Infrastructure — monitoring, logging, automation |
| 150–199 | Network services — DNS, VPN, reverse proxy |
| 200–249 | Applications — media, cloud storage, password manager |
| 900–999 | Scratch and experiments — assume these get destroyed |

**Hostnames:** lowercase, role-based, no numbers unless there are genuinely several.
`prometheus`, `grafana`, `jellyfin`. Not `prom-01-final`.

**Before assigning a static IP**, confirm it sits outside the router's DHCP pool. Two
devices claiming one address produces some of the most confusing failures in networking.

---

## 3. The host stays clean

**Nothing gets installed on the Proxmox host except tools that manage the host.**

`lm-sensors`, `htop`, backup agents — fine. Databases, web servers, Docker, anything that
serves a workload — no, it goes in a guest. A hypervisor that accumulates services becomes
impossible to reason about and impossible to rebuild.

The test: if reinstalling Proxmox from scratch would lose it, it's in the wrong place.

---

## 4. Storage

**`local` is 96 GB. `local-lvm` is 794 GB.** They are not interchangeable.

- Guest disks → `local-lvm`
- ISOs, templates, backups → `local`, and `local` fills fast

**Thin pool utilization is a monitored metric, not something to check occasionally.**
`local-lvm` is thin-provisioned, so allocated disk can exceed physical disk. If the pool
actually fills, running guests can corrupt. Watch `Data%` in `lvs`.

**Alert thresholds:** warn at 75%, alarm at 85%. Do not wait for 95%.

---

## 5. Backups

**Nothing counts as "done" until it is backed up and a restore has been tested.**

A backup that has never been restored is a guess. Restore tests go in the journal with the
date and how long the restore took.

- Anything holding data that would hurt to lose gets a scheduled backup
- Backups do not live only on the same physical disk as the thing they back up
- Snapshot before any risky change; delete the snapshot once the change is confirmed good

Snapshots are not backups. They live on the same disk and die with it.

---

## 6. Secrets

**No credentials in this repo. Ever.**

Passwords, API tokens, keys, certificates — password manager only. The repo may reference
*where* a secret lives, never the secret.

Before the first commit and before every push: check for anything that looks like a
credential. Git history is effectively permanent, and scrubbing a leaked secret from a
pushed repo is unpleasant.

**SSH keys over passwords** wherever supported. Root password login gets disabled once key
access is confirmed working.

---

## 7. Manual first, then code

Doing something by hand the first time is fine and often faster to learn from. But:

**Anything done twice by hand gets automated.** Anything that would need doing again after
a rebuild becomes Terraform or Ansible.

When something is built by hand, the journal entry records enough to reproduce it. The
target state is that the whole lab rebuilds from this repo onto bare Proxmox.

---

## 8. Change discipline

**One change at a time.** Two simultaneous changes mean an unclear cause when something
breaks, which wastes the most valuable part of the exercise.

**Every change gets a journal entry**, even a one-liner. Especially the small ones — those
are the changes nobody remembers making and everybody later blames.

**If a change involved a real tradeoff, write a decision record.** Not every change does.
The test: would future-me wonder "why did I do it that way?"

---

## 9. Deliberate breakage

**Once a week, break something on purpose.**

Kill a service. Fill a disk. Blackhole DNS for a container. Pull the network. Then watch
what the monitoring does about it, and fix whichever gap that exposes.

Every planned breakage gets a journal entry: what was broken, what was expected, what
actually happened. Divergence between the last two is the entire point.

---

## 10. Monitoring

**A service is not deployed until it is monitored.**

Minimum for anything that matters: up/down, memory over time, disk usage. If it can't be
observed, it can't be trusted to run unattended, and running things unattended is the
whole objective.

Baseline host metrics that are always collected: thin pool utilization, `local` free
space, CPU package temperature, RAM usage, guest up/down.
