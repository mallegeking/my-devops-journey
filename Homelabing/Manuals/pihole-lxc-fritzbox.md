# Pi-hole on Proxmox LXC with FritzBox

Layer: network DNS
Status: running, v1
Built: 07.09.2026
Host: `mypve` (Proxmox VE 8.x, kernel 6.8.12-9-pve)

---

## 1. Architecture

```
Client (DHCP lease from FritzBox)
   |
   |  DNS server = 192.168.178.9 (v4) / fd92:...:8a96 (v6)
   v
Pi-hole (CT 110)
   |
   +-- reverse lookups + *.fritz.box --> FritzBox 192.168.178.1
   |
   +-- everything else --> upstream resolver
```

The FritzBox keeps DHCP. Pi-hole does DNS only. The FritzBox hands out Pi-hole's
address in every lease, so each client appears individually in the dashboard.

Deliberately **not** done:

- Pi-hole is not the FritzBox's upstream DNS (`Internet > Zugangsdaten > DNS-Server`
  left untouched). Combining that with conditional forwarding creates a DNS loop.
- No secondary DNS server is handed out. A fallback resolver means clients
  round-robin to it and silently bypass all blocking.
- Pi-hole is not in the k3s cluster. DNS is the dependency everything else has;
  it must not go down during an Argo CD sync or a node reboot. Full reasoning,
  along with why this is an LXC and not a VM, is in section 2.

---

## 2. Container

### Why LXC, and why not the cluster

**What an LXC actually is**, since this is the first and only one here. It is a
*system* container: a full Debian userland with its own init, systemd, package
manager, SSH daemon, and network stack, that shares the Proxmox host's kernel
instead of booting one of its own. There is no hardware virtualization in the
path. The processes inside appear in `ps` on the host as ordinary processes,
fenced off by kernel namespaces (it sees its own PIDs, mounts, network) and
cgroups (it gets its own slice of CPU and RAM). That shared kernel is the whole
reason it starts in seconds and fits in 512 MB where a VM needs 2 GB.

Against the other two things in this homelab that look similar:

- a **VM** (400-404, 901) boots a real kernel of its own under KVM. Full
  isolation, full cost: a guest kernel, a boot sequence, its own memory floor.
- a **Kubernetes/Docker container** is an *application* container. One process,
  built from an image, thrown away and replaced rather than maintained. Its
  config arrives from outside, and its identity is not meant to be stable.
- an **LXC** is a machine you keep. You SSH in, `apt update`, `pihole -up`. That
  is why section 7 reads like ordinary sysadmin work rather than the GitOps loop
  everything on the cluster goes through.

`--unprivileged 1` in the create command means root inside the container is
mapped to an unprivileged UID on the host, so a breakout lands on a nobody
account rather than on root of `mypve`. It is the default for good reason and
worth never turning off for something this exposed to the network.

Everything else in this homelab is either a VM cloned from template 901 or a pod
on the k3s cluster those VMs form. Pi-hole is the only LXC and the only service
deliberately kept outside the cluster. The reason is the same in both directions:
DNS is not an application here, it is the thing every other application assumes
is already working.

**It cannot depend on anything it serves.** The k3s agents take their resolver
from a FritzBox DHCP lease, and that lease now hands out `192.168.178.9`, this
container. As a pod, a cold cluster start would need to pull the Pi-hole image
from `docker.io`, which needs DNS, which is the pod that has not started yet.
The same shape of loop as the conditional-forwarding one in section 1, one layer
down. An LXC boots from a local rootfs and resolves nothing to come up.

**Argo CD would own it.** Both cluster apps are registered with
`--sync-policy automated --self-heal` (see `argocd-setup.md` and
`n8n/n8n-cloudflare-tunnel-setup.md`). That is correct for `forge` and n8n and
wrong for DNS: a bad manifest commit would roll the flat's resolver, and
self-heal would keep reverting a manual fix while the network is down. The
emergency rollback in section 7 has to be a FritzBox field that anyone can clear,
not a `git revert` that still has to sync.

**It would not gain availability anyway.** Cluster storage is the local-path
provisioner with `ReadWriteOnce` PVCs, which is why both existing apps run
`replicas: 1` pinned to whichever node holds the volume. A Pi-hole pod would be
the same one replica on one node, plus Traefik, a PVC, and Argo CD added to the
failure path. More moving parts for the identical single point of failure.

**The FritzBox needs one fixed address.** There is exactly one Lokaler
DNS-Server field, and the IPv6 side is pinned to an EUI-64 address derived from
this container's MAC (section 4). Pods have neither a stable MAC nor a stable
address. Every other service here is reached through Traefik with a hosts-file
entry, or through the Cloudflare Tunnel. Neither of those is something a DHCP
client can be handed as a resolver on port 53.

### Alternatives rejected

| Option | Why not |
|---|---|
| **VM cloned from 901** | Pi-hole needs no kernel of its own. A VM costs a guest kernel, a full boot cycle, and several times the RAM, on a host already committing 2 GB × 3 to the k3s nodes with a thin pool that is nearly out of room (see open items). It would also drag in the machine-id and `cloud-init clean` ritual from the template doc for no benefit. `pct create` is one command with no template rule to respect. |
| **Pod on the k3s cluster** | See above: bootstrap loop, Argo CD in the failure path, no availability gained, no stable address to hand out. |
| **Docker on an existing VM** | There is no Docker anywhere on the server side of this homelab. The k3s nodes use k3s's own embedded containerd, and images are built on a Windows machine with Docker Desktop (`argocd-setup.md`). Installing Docker here adds a runtime that exists nowhere else, patched by neither `apt` nor Argo CD, to run one daemon. |
| **Raspberry Pi as the primary** | The Pi is the planned *secondary* (section 10) and is also earmarked for the agent harness. Making it primary does not remove the single point of failure, it relocates it onto SD-card storage with no snapshots, no `vzdump`, and no backup job, and spends the only second machine available for failover. The primary belongs where the backup tooling already exists. |
| **FritzBox alone** | It forwards and caches DNS but cannot block. That is the entire reason for this build. |
| **Filtering public resolver (NextDNS, Quad9 blocklists)** | No per-client visibility, no `fritz.box` hostname resolution, and no local DNS records for homelab services (section 9). Quad9 stays as the upstream, not as the whole answer. |

### What this choice costs

- **The host kernel is shared.** An unprivileged LXC has no kernel of its own, so
  patching or rebooting `mypve` takes DNS down with it. That is the same single
  point of failure described under Known gaps, and the reason the backup Pi-hole
  project exists. A VM would not have helped here, it sits on the same host.
- **No live migration.** Proxmox can only restart-migrate containers. Irrelevant
  with one Proxmox host, relevant the day a second one appears.
- **LXC is the case other guides assume away.** The k3s notes record that k3s
  loads `br_netfilter` itself "on a normal VM (non-LXC, non-rootless)", and the
  install obstacles in section 6 are almost all container-specific. Expect any
  guide to assume a VM and to be quietly wrong about the container path.

### Container definition

| | |
|---|---|
| CT ID | 110 |
| Hostname | `pihole` |
| Template | `debian-12-standard_12.12-1_amd64.tar.zst` |
| Type | unprivileged, `nesting=1` |
| Cores / RAM / swap | 1 / 512 MB / 512 MB |
| Rootfs | `local-lvm:4` (4 GB) |
| IPv4 | `192.168.178.9/24`, gw `192.168.178.1` |
| IPv6 | `ip6=auto` (SLAAC) |
| MAC | `bc:24:11:39:8a:96` |
| Start on boot | yes |

Create command:

```bash
pct create 110 local:vztmpl/debian-12-standard_12.12-1_amd64.tar.zst \
  --hostname pihole \
  --cores 1 --memory 512 --swap 512 \
  --rootfs local-lvm:4 \
  --net0 name=eth0,bridge=vmbr0,ip=192.168.178.9/24,gw=192.168.178.1,ip6=auto \
  --onboot 1 \
  --unprivileged 1 \
  --features nesting=1 \
  --password
```

---

## 3. Pi-hole configuration

- Version: Pi-hole v6
- Upstream DNS: Quad9 (filtered, DNSSEC)
- Query logging: on
- Privacy level: 0 (show everything)
- Conditional forwarding: `192.168.178.0/24` / `192.168.178.1` / `fritz.box`

Conditional forwarding is what makes the dashboard show hostnames instead of bare
IPs. The FritzBox owns the DHCP leases, so it is the only thing that knows which
name belongs to which address. Only reverse lookups go there, everything else
goes upstream.

Upstream chosen for jurisdiction (Swiss foundation, non-profit) and because the
filtered variant drops malware and phishing domains as a layer under the
blocklists. ECS variants were avoided: EDNS Client Subnet forwards part of the
client IP to the resolver, which works against the point of self-hosting.

### Blocklists

- Pi-hole default list
- `https://big.oisd.nl`
- planned: HaGeZi (`https://github.com/hagezi/dns-blocklists`)

---

## 4. FritzBox configuration

Switch the UI to **Erweiterte Ansicht** first (bottom left), or the relevant
fields stay hidden.

### IPv4

`Heimnetz > Netzwerk > Netzwerkeinstellungen > IP-Adressen > IPv4-Konfiguration`

- Lokaler DNS-Server: `192.168.178.9`

Clients only pick this up on lease renewal. Reconnect wifi to force it.

### IPv6 (ULA)

`Heimnetz > Netzwerk > Netzwerkeinstellungen > IP-Adressen > IPv6-Adressen`

- Unique Local Addresses: **"Unique Local Addresses (ULA) immer zuweisen"**
  (the other option only assigns ULAs while the internet is down)
- Lokaler DNSv6-Server: `fd92:ebb3:89aa:0:be24:11ff:fe39:8a96`
- "DNSv6-Server auch über Router Advertisement bekanntgeben (RFC 5006)": on

ULA is used because the public IPv6 prefix changes on reconnect, so any static
address from it would break. The ULA prefix `fd92:ebb3:89aa::/48` was generated
randomly by the FritzBox per RFC 4193 and is stable.

Without this, clients would receive the FritzBox as their IPv6 resolver and
bypass Pi-hole for any AAAA-capable lookup.

**Dependency worth knowing:** the interface ID `be24:11ff:fe39:8a96` is EUI-64,
derived deterministically from the container MAC `bc:24:11:39:8a:96` (the `ff:fe`
in the middle is the marker). It survives reboots. It does **not** survive
cloning the container or restoring a backup into a new CT ID, both of which
generate a new MAC. If IPv6 DNS silently stops working after a restore, this is
why: re-read the address and update the FritzBox field.

```bash
pct exec 110 -- ip -6 address | grep "inet6 fd"
```

---

## 5. Verification

From inside the container:

```bash
pihole -v
pihole status
systemctl is-active pihole-FTL
ss -tulpn | grep -E ':53|:80'

dig google.com @127.0.0.1 +short        # real IPs
dig doubleclick.net @127.0.0.1 +short   # 0.0.0.0
```

From a client, after reconnecting to the network:

```bash
nslookup pi.hole
```

Then check three things in the dashboard:

1. individual client hostnames appear (conditional forwarding works)
2. query count climbs (IPv4 path works)
3. AAAA queries appear (IPv6 path works)

If AAAA queries never show up, the ULA path is not being used. Fall back to
disabling IPv6 on the FritzBox.

---

## 6. Install obstacles hit (for next time)

These cost the most time and none of them are documented anywhere obvious.

**Template version.** `pct create` failed with `unsupported debian version '13.6'`
after extracting the rootfs successfully. This PVE version's `pve-container`
package predates Debian 13 (trixie) support, so it has no OS profile to apply the
hostname and network config with. Not a template problem. Debian 12 works.
A newer PVE would accept 13.

**Whiptail needs a real TTY.** The installer exited repeatedly with
`[i] Installer exited at static IP message`, and eventually the real error
`cannot open tty-output`. Two separate causes stacked here:

- `curl -sSL https://install.pi-hole.net | bash` gives bash the script on stdin,
  so the dialogs have no terminal to read from and return cancel instantly.
- `pct enter 110` does not reliably provide a TTY whiptail can draw on.

Fix: SSH into the container (or `pct console 110`, exit with `Ctrl+a` then `q`)
and run the script as a file:

```bash
curl -sSL https://install.pi-hole.net -o install.sh
bash install.sh
```

`curl -sSLO` does **not** work here. `-O` derives the filename from the URL path,
and `https://install.pi-hole.net` has no path, giving `curl: (23) Failed writing
received data to disk`.

Unattended fallback if the terminal will not cooperate: `bash install.sh --unattended`,
then configure everything from the web UI.

**Root SSH is off by default on Debian.** `PermitRootLogin prohibit-password`
produces the same denied prompt as a wrong password, so the two look identical.

```bash
pct exec 110 -- sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
pct exec 110 -- systemctl restart ssh
pct exec 110 -- sshd -T | grep permitrootlogin
```

**German keyboard on the Proxmox console.** The console defaults to US layout.
`y`/`z` swap and most special characters move, so a password typed at
`pct create` can come back different at the login prompt. Reset with
`pct exec 110 -- passwd root` and use letters a-x plus digits until you are in a
proper shell.

---

## 7. Operations

### Updating

```bash
pihole -up            # Pi-hole itself
apt update && apt full-upgrade   # container OS
```

Gravity (blocklist refresh) runs weekly by cron on its own.

### Backup

Two independent layers, both needed:

- **Proxmox**: container-level snapshot. Currently a manual one-off. A scheduled
  job is still to be set up (see open items).
- **Teleporter**: `Settings > Teleporter` exports lists, groups, allowlist, and
  settings as a portable archive. Restoring from this takes two minutes.
  Rebuilding an allowlist from memory after whitelisting forty broken sites takes
  weeks. Re-export after any significant config change.

### Emergency rollback

If Pi-hole is down and the flat has no internet, this is the 30-second fix.
Anyone should be able to do it.

1. FritzBox UI (`http://fritz.box`) > Erweiterte Ansicht
2. `Heimnetz > Netzwerk > Netzwerkeinstellungen > IP-Adressen > IPv4-Konfiguration`
3. Clear the **Lokaler DNS-Server** field, apply
4. Same page, IPv6 section: clear **Lokaler DNSv6-Server**, apply
5. Reconnect the affected device

DNS falls back to the FritzBox. No ad blocking, but the network works.

---

## 8. Known gaps

**Single point of failure.** If `mypve` is down or being patched, the whole flat
loses DNS. This is the main open risk and the reason for the backup Pi-hole
project below.

**Clients that bypass Pi-hole.** Nothing here catches:

- browsers with DoH enabled (Firefox, Chrome secure DNS) resolve over HTTPS to
  their own provider and never touch port 53
- Android "Private DNS" set to automatic or a specific provider
- devices with hardcoded resolvers (Chromecast, some smart TVs)

If blocking looks like it stopped working on one device, check this before
suspecting Pi-hole. The FritzBox can block outbound port 53 for everything except
Pi-hole (`Internet > Filter`), which catches the hardcoded case but not DoH.

**Guest network.** Not in use. If one is ever enabled, note that the FritzBox
always hands out its own IP as DNS for guests, with no setting to change it.
Filtering guest traffic would require making Pi-hole the FritzBox's upstream,
which conflicts with the conditional forwarding setup above.

**Thin pool.** Deferred, see open items.

---

## 9. Open items

| Item | Why | Priority |
|---|---|---|
| Proxmox thin pool: check `data_percent`, enable `thin_pool_autoextend_threshold` | Volumes total 398 GiB against a pool with 16 GiB free in the VG. A full pool wedges and guests go read-only. | high |
| Scheduled Proxmox backup job including CT 110 | Only a manual snapshot exists | high |
| Proxmox host updates (40 packages pending, ~18 months behind) | Needs its own session, k3s and Argo CD live on this host | medium |
| Add HaGeZi list | Chosen, not yet added | low |
| Local DNS records for homelab services | Started. `argocd.internal` -> `192.168.178.134` is live and retired the Argo CD port-forward entirely (`argocd-lan-access.md`). `forge.local` and `nginx-test.local` still need converting. Note `.internal` is used, not `.local` (mDNS) or `.home` (unreserved) | medium |

---

## 10. Follow-up projects

### A. Backup Pi-hole on the Raspberry Pi

Goal: flatmates keep connectivity while `mypve` is down or being patched.

**The problem to solve first:** the FritzBox has exactly one Lokaler DNS-Server
field. Adding a second Pi-hole does not automatically give failover, because
nothing hands its address to clients. Options:

- **keepalived / VRRP with a floating IP.** Both Pi-holes share a virtual IP;
  the FritzBox points at the VIP; the Pi takes it over when the LXC stops.
  This is real failover and the right answer.
- Manual: change the FritzBox field when doing maintenance. Works, but you have
  to remember, which defeats the point.

**Config sync:** `lovelaze/nebula-sync` for Pi-hole v6. Gravity Sync is archived
and Orbital Sync was built for the v5 API (archived March 2025); v6 changed the
API and auth model. Nebula Sync uses the v6 API, runs on a cron, does full
Teleporter export/import from primary to replicas, optionally running gravity
after. Verify current status before building.

Note the Pi is also earmarked for openclaw/hermes. Decide whether they coexist.

### B. Recursive resolution with Unbound

Removes the third party entirely. Instead of asking Quad9, Pi-hole runs Unbound
locally and resolves from the root servers down, doing its own DNSSEC validation.

Trade-off worth understanding before building: this does not make you anonymous.
It changes *who* sees your queries. Right now one resolver sees everything; with
Unbound, each authoritative nameserver sees the queries for its own zone, coming
from your home IP. Cold lookups are slower, cached ones are faster. The real gain
is no single party holding a complete picture, plus no dependence on an upstream
being up.

The distinction to keep straight: DoH/DoT *encrypts* queries to a chosen third
party, it does not remove them. Unbound removes them.

### C. Centralized update overview

Problem: no single view of what needs patching across the homelab. Proxmox host,
LXCs, k3s nodes, container images. Things get forgotten.

Sketch, using what already exists:

- n8n workflow on a weekly cron, SSH to each host, collect `apt list --upgradable`
  counts (and `pihole -v` for Pi-hole, image digests for k3s workloads)
- output a digest to Telegram, and create a single Todoist task when anything
  crosses a threshold
- keep it read-only. Reporting is the value; automated patching on a host running
  a live k3s cluster is a different risk profile.

This fits the same pattern as the Ausbildung pipeline: automate the noticing,
keep the human deciding.
