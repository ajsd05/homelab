# Homelab

A single-node Proxmox lab, run deliberately as production: everything defined in code,
everything monitored, everything documented, and something broken on purpose every week.

The goal isn't the services. It's operational practice — building things that recover on
their own, and getting fast at diagnosing them when they don't.

**Status:** Bare hypervisor, empty. First build target is a self-healing monitoring loop.

---

## Hardware

Lenovo ThinkCentre Tiny — Intel i7-7700T (4c/8t, 35 W), 16 GB RAM, 931 GB NVMe.
Proxmox VE 9.1.1 on bare metal. Single 1 GbE NIC, static IP on a Linux bridge.

Full verified inventory: [`homelab-context.md`](homelab-context.md)

---

## What's here

| File | What it is |
|---|---|
| [`RULES.md`](RULES.md) | Standing conventions — naming, IP scheme, backup and change discipline |
| [`JOURNAL.md`](JOURNAL.md) | Append-only log of every change, failure, and fix |
| [`decisions/`](decisions/) | Decision records — choices with real tradeoffs, and their reasoning |
| [`homelab-context.md`](homelab-context.md) | Verified hardware and network inventory |

---

## The project

A closed loop that watches the lab, notices when something breaks, fixes it, and records
what it did — so the action log becomes a dataset about the infrastructure's own behaviour.

Build order, each step justified by a failure of the one before it:

1. One instrumented service and a crude health-check script on cron
2. Weekly deliberate breakage, exposing what the crude version misses
3. Prometheus, Grafana, and Alertmanager replacing the crude checks
4. Structured event log of every automated action, with a dashboard over it
5. Terraform and Ansible, so the whole lab rebuilds from this repo onto bare Proxmox

---

## Operating principles

- **LXC by default, VM by exception** — 16 GB is the binding constraint ([0001](decisions/0001-lxc-as-default-guest.md))
- **Nothing is deployed until it's monitored**
- **Nothing is done until a restore has been tested** — an untested backup is a guess
- **Break something every week** — the failures write the roadmap
- **No secrets in this repo, ever**

---

## Journal, briefly

The journal records what broke, what it looked like, what it actually was, and how long the
gap between those two lasted. That gap is the only honest measure of diagnostic skill, and
it doesn't show up anywhere in a commit history.
