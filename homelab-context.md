# Homelab Context Document

> Paste or attach this at the start of a Claude conversation to give full awareness of the setup.
> Everything in "Verified Hardware" was read directly off the machine — treat it as ground truth.

*Last verified: September 2026*

---

## Owner Profile

- **Experience level:** Beginner, but with a CS degree — comfortable with code, new to infrastructure
- **Goal (next 6 months):** Get genuinely good at systems/infra. Depth over resume padding.
- **Interests:** Data and dashboards; automation and self-healing systems
- **Management device:** Lenovo ThinkPad, Windows + WSL (user `andrew`)

---

## Access

| Item | Value |
|---|---|
| Proxmox web UI | https://192.168.1.50:8006/ |
| Proxmox login | `root`, realm: **Linux PAM standard authentication** |
| SSH | `ssh root@192.168.1.50` (from WSL) |
| Credentials | Stored in password manager — **not in this file** |

**Notes:**
- Self-signed certificate, so the browser warns on first visit. Expected.
- Wrong realm is the most common cause of a rejected login. Must be Linux PAM.
- Proxmox root and the WSL user should have **different** passwords. Split them if not already done.
- Long-term: SSH keys + disable root password login; non-root PVE user with an API token for automation.

---

## Verified Hardware — Mini PC (the lab)

### System
| Property | Value |
|---|---|
| Model | Lenovo ThinkCentre Tiny (`LENOVO TC-M1A`) — M710q/M910q family |
| Role | Proxmox VE hypervisor, bare metal |
| Hostname | `proxmox` |
| PVE version | 9.1.1, kernel 6.17.2-2-pve |
| VMs / LXC | None yet |

### CPU
| Property | Value |
|---|---|
| Model | Intel Core i7-7700T @ 2.90 GHz (Kaby Lake, 7th gen) |
| Cores / Threads | 4 cores / 8 threads |
| TDP | 35 W (low-power "T" part) |
| VT-x | ✅ Enabled and confirmed |
| VT-d / IOMMU | ✅ DMAR tables present — hardware supports passthrough |

### Memory
| Property | Value |
|---|---|
| RAM | 16 GB (15 GiB usable) |
| Swap | 8 GB (on LVM) |
| Typical idle usage | ~1.4 GiB |

### GPU
| Property | Value |
|---|---|
| GPU | Intel HD Graphics 630 (integrated, `00:02.0`) |
| Quick Sync | ✅ Yes — hardware H.264/HEVC encode+decode |
| Best use | `/dev/dri` passthrough into an **LXC** for Jellyfin/Plex transcoding |
| Not useful for | LLM inference — Ollama would run on CPU, slowly |

> Prefer LXC device passthrough over VM VFIO passthrough. Passing the iGPU to a VM takes it away from the host and is far more fragile.

### Storage — **931 GB NVMe, single disk**
| Property | Value |
|---|---|
| Physical | `nvme0n1`, 931.5 GB, no redundancy |
| Layout | EFI 1 GB → LVM (`pve` VG) |
| `pve-root` | 96 GB, ext4, mounted `/` |
| `pve-swap` | 8 GB |
| `pve-data` | 793.8 GB LVM **thin** pool |
| Filesystem type | **LVM-thin (not ZFS)** — no checksumming, no send/recv, no native compression |

**Proxmox storage pools:**

| Pool | Type | Size | Holds |
|---|---|---|---|
| `local` | dir | **96 GB** | ISOs, LXC templates, **backups**, snippets |
| `local-lvm` | lvmthin | **794 GB** | VM and container disks |

⚠️ **Two traps to remember:**
1. Backups default to `local` (96 GB). A few VM backups fill it, and the failures are confusing rather than obvious. Add external backup storage before relying on it.
2. `local-lvm` is thin-provisioned — disks can be over-allocated. If the thin pool actually fills, running guests can corrupt. **Thin pool utilization is a critical metric to monitor.**

### Known gaps
- ❌ **No backup target.** Single disk, no redundancy, no offsite. Highest-priority fix — a ~$60 external USB drive added as a PVE storage target covers most of the risk.
- ❌ `lm-sensors` not installed (`apt install lm-sensors && sensors-detect --auto`). Thermals matter in a Tiny chassis.

---

## Network

```
Internet
   |
[ISP Combo Unit — Modem + Router]  192.168.1.254
   |
[Network Switch — unmanaged (assumed)]
   |
[Mini PC — Proxmox]  192.168.1.50
```

| Property | Value |
|---|---|
| Subnet | 192.168.1.0/24 |
| Gateway | 192.168.1.254 |
| Proxmox IP | 192.168.1.50 (static, set on `vmbr0`) |
| Bridge | `vmbr0` → `enp0s31f6` (onboard NIC) |
| NIC speed | 1000 Mbps (1 GbE) |
| Wireless | `wlp2s0` present but **DOWN and unused** — leave it that way |
| Upstream DNS | 1.1.1.1 (Cloudflare) — repoint here once Pi-hole/AdGuard runs |
| DHCP reservation for .50 | ❌ Not confirmed — should be set on the router |

> `vmbr0` is a Linux bridge, so new VMs/LXCs attached to it land directly on the 192.168.1.0/24 LAN and get IPs from the ISP router's DHCP. Plan static IPs or reservations for anything that matters.

---

## Service Status

| Service | Status | Notes |
|---|---|---|
| Proxmox VE 9.1.1 | ✅ Running | Empty — no guests yet |
| Docker | ❌ | Planned inside an LXC or VM |
| Monitoring (Prometheus/Grafana) | ❌ | **First build target** |
| DNS (Pi-hole / AdGuard) | ❌ | Planned |
| VPN (Tailscale) | ❌ | Planned — best remote-access option, no port forwarding |
| Reverse proxy | ❌ | Caddy preferred over NPM for automatic TLS |
| Media server (Jellyfin/Plex) | ❌ | **Now viable** — 794 GB available + Quick Sync |
| Nextcloud | ❌ | Planned |
| Vaultwarden | ❌ | Planned |
| Backups | ❌ | **Highest priority gap** |

---

## Current Project

**A self-healing monitoring loop.** A system that watches the lab, notices when something breaks, fixes it, and logs what it did — so the action log becomes a dataset about the infrastructure's own behavior.

Build order:
1. One instrumented service + a crude ~50-line Python health-check script (cron)
2. Break things deliberately, weekly; let each failure justify the next feature
3. Prometheus + Grafana + Alertmanager replacing the crude checks
4. Structured event log of every automated action → dashboard over it
5. Terraform (`bpg/proxmox` provider) + Ansible so the whole lab rebuilds from git

**Metrics worth collecting early:** LVM thin pool utilization, `local` (96 GB) free space, CPU package temperature, container up/down, memory growth trends.

**Habits:** weekly deliberate breakage; a plain-markdown lab journal recording what broke, what was suspected, what was actually wrong, and how long the gap was.

---

## Constraints & Open Questions

**Constraints to design around:**
- 16 GB RAM — a full monitoring stack takes 4–6 GB. Budget accordingly.
- Single 1 GbE NIC — fine for now; no link aggregation.
- Unmanaged switch + ISP router — **VLAN segmentation is not currently possible.** Network security work needs a managed switch and likely a separate router/firewall (e.g. OPNsense).
- No GPU suitable for LLM work.

**Still unknown:**
- [ ] Router make/model — determines DHCP reservations, port forwarding, custom DNS
- [ ] Switch: managed or unmanaged?
- [ ] Public IP: static or dynamic?
- [ ] Internet up/down speed (fast.com)
- [ ] Whether a DHCP reservation exists for 192.168.1.50
- [ ] Any external storage available for backups?
- [ ] Budget over the next 6 months

---

## Useful Commands

### Proxmox shell
```bash
lscpu                          # CPU details
free -h                        # memory
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT
pvesm status                   # storage pools + usage
lvs                            # LVM volumes; watch thin pool Data%
pveversion                     # PVE version
ip -br a                       # interfaces, brief
cat /etc/network/interfaces    # static IP / bridge config
pct list                       # LXC containers
qm list                        # VMs
sensors                        # temps (after installing lm-sensors)
journalctl -xe                 # recent system errors
```

### From WSL on the ThinkPad
```bash
ping 192.168.1.50
ssh root@192.168.1.50
ssh-keygen -t ed25519          # once
ssh-copy-id root@192.168.1.50  # then key-based login
```
