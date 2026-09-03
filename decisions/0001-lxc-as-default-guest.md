# 0001 — LXC containers as the default guest type

**Date:** 2026-09-02
**Status:** Accepted

## Context

The lab runs on a single Lenovo ThinkCentre Tiny: i7-7700T, 4 cores / 8 threads, 16 GB
RAM, one 931 GB NVMe. Proxmox offers two kinds of guest — full VMs (KVM) and system
containers (LXC) — and the choice needed making once rather than per service.

The planned workload is roughly a dozen small always-on services: monitoring, DNS, a
reverse proxy, a media server, a few applications. Most are a single daemon plus a config
file.

RAM is the binding constraint. A monitoring stack alone is budgeted at 4–6 GB, which is a
third of the machine before anything else runs.

## Options considered

**Full VMs for everything.** Strong isolation — each guest has its own kernel, so a
compromise or a kernel panic stays contained. Snapshots and live migration work
predictably. But every VM carries a full OS: ~512 MB to 1 GB of RAM before the workload
starts, plus disk for a complete root filesystem and a real boot cycle on every restart.
Twelve services this way would exhaust 16 GB on overhead alone.

**LXC containers for everything.** Containers share the host kernel. A running service can
sit in 100–200 MB, starts in about a second, and takes only the disk its files need. The
cost is weaker isolation — root in a privileged container is close enough to root on the
host to matter — and anything needing its own kernel or non-Linux OS simply can't run.

**LXC by default, VM by exception.** Containers for the ordinary case, VMs where something
specific demands one.

## Decision

LXC is the default. A VM requires a stated reason, recorded in the journal entry that
creates it.

Containers are unprivileged unless a specific device passthrough requires otherwise.

## Consequences

**Easier:** Roughly four to six times more services fit in 16 GB. Restarts are fast enough
that deliberate breakage stays cheap, which matters because breaking things weekly is part
of the plan. Intel Quick Sync passthrough for media transcoding is dramatically simpler
via `/dev/dri` in an LXC than via VFIO into a VM — VFIO would take the iGPU away from the
host entirely and is notoriously brittle.

**Harder:** Isolation is weaker. Every container shares the host kernel, so a kernel-level
vulnerability is a lab-wide problem rather than a per-guest one. This is an accepted risk
in a single-user lab on a trusted LAN; it would not be acceptable for anything
internet-exposed or multi-tenant.

**Ruled out:** No non-Linux guests without an explicit VM. Nested virtualization and
Docker-in-guest need care and may justify a VM case by case. Kernel-level experiments —
custom modules, kernel tuning, anything that could panic — must go in a VM, since a
container panic takes the host down with it.

## Revisit when

- Anything gets exposed directly to the internet rather than reached over Tailscale
- RAM goes above 32 GB, which makes VM overhead affordable
- A workload needs its own kernel or kernel-level isolation
- Someone other than the owner runs workloads here
