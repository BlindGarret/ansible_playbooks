# Roeden Lab — VM inventory and provisioning specs

Reference for provisioning the underlying VMs before any Ansible runs.
All 10 VMs sit on the `192.168.10.0/24` lab subnet, served by the NAS at
`192.168.0.29` for both DNS (`*.roeden.lab` zone) and NFS storage.

## Network map

| IP | Role | Notes |
|---|---|---|
| `192.168.10.1` | Gateway | Lab router |
| `192.168.10.20` | k3s API VIP | kube-vip floats this across the 3 control planes |
| `192.168.10.21` | `k3s-cp1` | k3s control plane #1 |
| `192.168.10.22` | `k3s-cp2` | k3s control plane #2 |
| `192.168.10.23` | `k3s-cp3` | k3s control plane #3 |
| `192.168.10.31` | `k3s-w1` | k3s worker #1 |
| `192.168.10.32` | `k3s-w2` | k3s worker #2 |
| `192.168.10.33` | `k3s-w3` | k3s worker #3 |
| `192.168.10.34` | `k3s-w4` | k3s worker #4 |
| `192.168.10.41` | `pg-01` | Postgres + Patroni + etcd node #1 |
| `192.168.10.42` | `pg-02` | Postgres + Patroni + etcd node #2 |
| `192.168.10.43` | `pg-03` | Postgres + Patroni + etcd node #3 |
| `192.168.10.50–70` | MetalLB pool | L2 LoadBalancer IPs handed out to k8s Services |
| `192.168.10.51` | Traefik internal | LoadBalancer for `traefik-internal` GatewayClass |
| `192.168.10.52` | Traefik external | LoadBalancer for `traefik-external` GatewayClass |
| `192.168.0.29` | NAS | External: DNS server for `*.roeden.lab`, NFS server, Synology |

Address allocation convention: `.20` for the k3s API VIP, `.21–29` for control
planes, `.31–39` for k3s workers, `.41–49` for Postgres-tier VMs, `.50–70`
reserved for MetalLB. Leave room to grow each band.

## VM specs

All VMs share the same baseline: cloud-init-enabled Debian 12 (Bookworm)
or Ubuntu 24.04 LTS, single-NIC on the lab VLAN, static IP, SSH user
`infraadmin` with the `~/.ssh/roeden_infra` public key pre-installed.

### k3s control planes — `k3s-cp1`, `k3s-cp2`, `k3s-cp3`

Three identical VMs.

| Resource | Min | Recommended |
|---|---|---|
| vCPU | 2 | 2 |
| RAM | 2 GiB | 4 GiB |
| Disk (root) | 20 GiB | 30 GiB |

Footprint: k3s binary (~500 MB) + etcd data dir (grows with cluster state,
plan ~5 GiB) + containerd images for k3s system pods (~3 GiB) + room
for logs. The control plane is **tainted** (`k3s_taint_control_plane:
true` in `group_vars/all.yml`), so no app pods schedule here — disk
usage is bounded by system components.

### k3s workers — `k3s-w1`, `k3s-w2`, `k3s-w3`, `k3s-w4`

Four identical VMs.

| Resource | Min | Recommended |
|---|---|---|
| vCPU | 2 | 4 |
| RAM | 4 GiB | 8 GiB |
| Disk (root) | 40 GiB | 60 GiB |

Footprint: containerd image cache scales with the number of app images
(observability stack alone pulls ~15 images), pod ephemeral storage,
emptyDir volumes. Prometheus + Grafana + Loki + Authentik are the
RAM-hungry workloads — 4 GiB per node is the floor before nodes start
evicting pods under pressure. 8 GiB gives comfortable headroom.

App data lives on NFS (via the `nfs-client` StorageClass) — workers do
**not** need additional data disks. The root disk just holds runtime
state.

### Postgres tier — `pg-01`, `pg-02`, `pg-03`

Three identical VMs, fully independent of the k3s tier.

| Resource | Min | Recommended |
|---|---|---|
| vCPU | 2 | 2 |
| RAM | 4 GiB | 4 GiB |
| Disk (root) | 30 GiB | 50 GiB |

Footprint: Postgres 18 binary + per-cluster data dir + WAL archive +
etcd 3.6 data dir + Patroni venv. Tuning in `group_vars/all.yml`
assumes 4 GiB RAM (`postgres_shared_buffers: 1GB`,
`postgres_effective_cache_size: 3GB`) — changing the VM RAM means
re-tuning those vars.

Disk grows with two things: (a) actual app data (Authentik, Grafana,
etc.), (b) WAL retention for the lagging replicas. 30 GiB is the floor;
50 GiB is comfortable for normal homelab use. If you start hosting an
app with sustained write load, revisit.

**Do NOT use NFS for Postgres data dirs** — Postgres needs synchronous
fsync semantics that NFS doesn't guarantee. The whole reason this tier
runs on VMs (not in k3s with NFS-backed PVCs) is this constraint.

## OS + base configuration

All VMs should be installed with:

- **OS**: Debian 12 (Bookworm) or Ubuntu 24.04 LTS, server install, no
  desktop environment. Both have been tested.
- **Network**: Static IP per the map above, `192.168.10.1` gateway,
  `192.168.0.29` as DNS server. The `search roeden.lab` search domain
  comes from DHCP normally — fine to leave; the k3s `kubelet-arg
  resolv-conf=/etc/k3s-resolv.conf` setup strips it from pods.
- **NTP**: any reasonable source (`pool.ntp.org` is fine). Patroni etcd
  is mildly clock-sensitive; ±2s drift is fine, more than that and the
  Raft cluster can flap.
- **SSH**: Enabled with key-based auth. User `infraadmin` (sudoer, NOPASSWD).
  Public key from `~/.ssh/roeden_infra` (on your Ansible controller) goes
  in `/home/infraadmin/.ssh/authorized_keys`.
- **Cloud-init**: Recommended — most cloud-init templates handle the IP
  + user + key in one shot. Past experience: corrupted cloud-init dpkg
  state has caused playbook failures; if you hit this, `dpkg
  --configure -a` is the fix.

No firewall preconfiguration needed beyond "allow inbound from the lab
subnet." K3s/kube-vip/MetalLB/Patroni all manage their own ports.

## External infrastructure expected to exist

These are NOT provisioned by these Ansible playbooks — they exist before
any rebuild starts.

### NAS at `192.168.0.29` (Synology or equivalent)

- **DNS server** authoritative for `roeden.lab`. Must answer queries for:
  - `*.roeden.lab` → `192.168.10.51` (Traefik internal IP — set this
    wildcard before issuing the internal cert)
  - Reverse PTR records for all 10 VMs (optional but nice for logs)
- **NFS export** at `/volume1/appdata`, accessible from the
  `192.168.10.0/24` subnet, no-root-squash. This is the backing store
  for the `nfs-client` StorageClass — every PVC in k3s lands as a
  subdir under this path.
- DHCP scope arrangement is up to you, but the VM IPs in the map above
  should be reserved/static.

### Wildcard DNS for the external domain (Cloudflare)

- Cloudflare-managed zone for `roedev.com`
- API token with `Zone:DNS:Edit` scope (used by cert-manager for the
  DNS-01 challenge — stored in `artifacts/cloudflare-api-token`)

## Pre-rebuild checklist

Before running `plays/postgres/bootstrap.yml`:

- [ ] All 10 VMs provisioned with the IPs above, infraadmin user + key
- [ ] All 10 VMs reachable from the Ansible controller via SSH
- [ ] NAS DNS resolves `*.roeden.lab` to `192.168.10.51` (do this BEFORE
      running ingress.yml so cert-manager challenges succeed)
- [ ] NAS NFS export `/volume1/appdata` mountable from the worker subnet
- [ ] `artifacts/cloudflare-api-token` exists and is current
- [ ] `artifacts/discord-webhook-url` exists (for Alertmanager)
- [ ] `artifacts/roeden-ca.crt` + `artifacts/roeden-ca.key` exist
      (internal CA — generate once, keep forever; lose this and every
      internal cert needs reissuing AND every laptop needs re-trusting
      the new CA)
- [ ] If restoring an existing OpenBao cluster (NFS data dirs preserved
      across the rebuild): `artifacts/openbao-init.json` exists — without
      it the existing OpenBao state is unrecoverable. For a true clean
      slate (delete OpenBao PVCs on the NAS too), this file is
      regenerated on first run.

The rest of `artifacts/` (random passwords, OIDC secrets, k3s join token)
auto-generates on the first playbook run.
