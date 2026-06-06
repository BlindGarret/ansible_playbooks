# Roeden Lab — Ansible Operations

Ansible plays and roles for running the `roeden` homelab: 3 k3s control planes,
4 k3s agents, supporting infrastructure (NAS, EdgeRouter), and platform
services (Traefik ingress, cert-manager, NFS storage, ArgoCD).

Use this README when future-you doesn't remember which play to run. Every
section is a working command sequence — copy/paste them in order.

---

## Repo layout cheat sheet

```
inventories/roeden/
  hosts.yml                # node IPs + SSH user
  group_vars/all.yml       # tracked versions + cluster vars
  host_vars/k3s-cp1.yml    # k3s-cp1 specifics (cluster_init: true)

plays/
  k3s/
    bootstrap.yml          # one-time: stand up the k3s cluster from scratch
    ingress.yml            # idempotent: Traefik + cert-manager + Gateways
    services.yml           # idempotent: NFS provisioner + ArgoCD
    check-updates.yml      # READ-ONLY: drift report for every tracked component
    upgrade.yml            # weekly: rolling k3s/component upgrades + cert renewal
  lab/
    check-os-updates.yml   # READ-ONLY: apt updates available per host
    rolling-os-updates.yml # rolling apt upgrade + reboot, drains k3s nodes
    trust-internal-cert.yml # push the internal CA cert to every host

roles/                     # see individual roles for details
artifacts/                 # gitignored — holds secrets and the kubeconfig
```

---

## Prerequisites

### On the controller (your laptop)

```bash
# Required CLI tools
helm version
ansible --version          # ansible-core 2.20+ tested
kubectl version --client
openssl version
mise --version             # optional but used to install helm/argocd cli
```

If `helm` is missing: `mise use -g helm@latest` or
`curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash`.

If `argocd` is missing (only needed for the ArgoCD user setup):
`mise use -g argocd@latest`.

Ansible needs the `kubernetes` Python lib for `kubernetes.core.k8s`:
```bash
pipx inject ansible kubernetes pyyaml jsonpatch
```

### Files you must put in `artifacts/` yourself

These are **gitignored** secrets. Never commit them, never copy them to
shared places. Generate them as described below, then they live forever on
the controller (back them up somewhere safe).

| File | What it is | How to create |
|---|---|---|
| `roeden-ca.crt` | Internal CA cert (trust anchor) | One-time openssl command, see "Initial setup" step 3 |
| `roeden-ca.key` | Internal CA private key | Same openssl command |
| `cloudflare-api-token` | Cloudflare DNS-01 API token | Generate in Cloudflare dashboard, paste with `echo -n` |
| `k3s-token` | k3s cluster join token | `openssl rand -hex 32 > artifacts/k3s-token` |

### Files that show up in `artifacts/` automatically

You don't create these — the plays do. Just know what they are when you
see them.

| File | What it is | Created by |
|---|---|---|
| `internal-tls.crt` + `.key` | Server cert for `*.roeden.lab`, signed by the CA | `plays/k3s/ingress.yml` (or `upgrade.yml`) the first time it sees no leaf, then renewed when within 30 days of expiry |
| `kubeconfig-roeden.yaml` | k3s admin kubeconfig | `plays/k3s/bootstrap.yml` fetches it from cp1 |
| `roeden-ca.srl` | openssl serial-number tracker for cert signing | Created the first time the CA signs a cert; harmless |

---

## OS update operations (weekly chore)

### Check what's pending

```bash
ansible-playbook -i inventories/roeden/hosts.yml plays/lab/check-os-updates.yml
```

Output is a per-host count of upgradable packages plus the package names.
Read-only — refreshes apt cache, doesn't install anything.

### Apply OS updates with rolling reboot

```bash
ansible-playbook -i inventories/roeden/hosts.yml plays/lab/rolling-os-updates.yml
```

What this does, per host, one at a time:
1. **k3s nodes**: cordon → drain (workloads relocate) → `apt upgrade` →
   reboot → wait for k3s service active → wait for node Ready → uncordon
2. **Non-k3s lab hosts** (when you add any): just `apt upgrade` → reboot

Skips nodes with zero pending updates (won't drain/reboot for nothing).
Rolling order is control planes → agents → others, `serial: 1`, so the
cluster stays up.

Run cadence: **weekly** for security patches.

---

## Initial Kubernetes setup (fresh cluster, once)

This is the bootstrap-from-zero path. Skip this section if the cluster
already exists.

### 1. Inventory: confirm the node list

`inventories/roeden/hosts.yml` should list `k3s_servers` (cp1–cp3) and
`k3s_agents` (w1–w4) with their static IPs in the `192.168.10.0/24` range.

### 2. Drop required secrets into `artifacts/`

```bash
# k3s join token — any 32-byte hex string, just keep it consistent for the cluster
openssl rand -hex 32 > artifacts/k3s-token
chmod 600 artifacts/k3s-token

# Cloudflare API token (scope: Zone DNS Edit for roedev.com only)
# Generate at https://dash.cloudflare.com → My Profile → API Tokens
echo -n 'PASTE-TOKEN-HERE' > artifacts/cloudflare-api-token
chmod 600 artifacts/cloudflare-api-token
```

### 3. Generate the internal CA

The CA is long-lived (10 years) and only signs the leaf cert. The leaf cert
itself gets auto-generated on first `upgrade.yml` or `ingress.yml` run.

```bash
openssl req -x509 -nodes -newkey rsa:2048 \
  -keyout artifacts/roeden-ca.key \
  -out artifacts/roeden-ca.crt \
  -days 3650 \
  -subj "/CN=Roeden Lab Internal CA" \
  -addext "basicConstraints=critical,CA:TRUE" \
  -addext "keyUsage=critical,keyCertSign,cRLSign"
chmod 600 artifacts/roeden-ca.key
```

### 4. Bootstrap k3s

```bash
ansible-playbook -i inventories/roeden/hosts.yml plays/k3s/bootstrap.yml
```

This runs in phases: prereqs on every node → cp1 init → kube-vip up →
cp2/cp3 join → agents join → MetalLB → fetch kubeconfig to
`artifacts/kubeconfig-roeden.yaml`.

Verify:
```bash
export KUBECONFIG=$PWD/artifacts/kubeconfig-roeden.yaml
kubectl get nodes
# 7 nodes Ready: 3 cps + 4 workers
```

### 5. Push the internal CA to all hosts

So every node trusts `*.roeden.lab` without `--insecure`:

```bash
ansible-playbook -i inventories/roeden/hosts.yml plays/lab/trust-internal-cert.yml
```

### 6. Install the ingress tier

Traefik (two isolated instances on `.51`/`.52`), cert-manager, the two
Gateways (internal + external), and the wildcard Let's Encrypt cert.

```bash
ansible-playbook -i inventories/roeden/hosts.yml plays/k3s/ingress.yml
```

This run also auto-generates `artifacts/internal-tls.crt` + `.key` from the
CA the first time (no leaf cert exists yet).

Verify:
```bash
kubectl get gatewayclass
# traefik-internal   traefik.io/gateway-controller   True
# traefik-external   traefik.io/gateway-controller   True

kubectl -n traefik get gateway
# internal   traefik-internal   192.168.10.51   Programmed=True
# external   traefik-external   192.168.10.52   Programmed=True

kubectl -n traefik get certificate
# external-tls   True   90d
```

### 7. Set DNS

- **Internal DNS** (Pi-hole / dnsmasq / EdgeRouter): wildcard
  `*.roeden.lab → 192.168.10.51`.
- **Public DNS** (Cloudflare): A record for whatever public hostname you'll
  use (e.g. `home.roedev.com → <your public IP>`, **DNS-only**, grey cloud).
- **Router NAT**: TDS forwards `:80`/`:443` → `192.168.0.22`, EdgeRouter
  forwards same ports → `192.168.10.52` *only when destination is
  `192.168.0.22`* (so internal `.51` traffic isn't hijacked).

### 8. Install platform services

```bash
ansible-playbook -i inventories/roeden/hosts.yml plays/k3s/services.yml
```

This installs the NFS provisioner (StorageClass `nfs-client`, marked
default) and ArgoCD (Ingress on `argocd.roeden.lab`).

Verify:
```bash
kubectl get storageclass
# nfs-client (default)

kubectl -n argocd get pods
# All Running

kubectl -n argocd get ingress
# argocd-server   traefik-internal   argocd.roeden.lab   192.168.10.51
```

### 9. Trust the CA on your laptop

System:
```bash
sudo cp artifacts/roeden-ca.crt /usr/local/share/ca-certificates/roeden-ca.crt
sudo update-ca-certificates
```

Chrome/Chromium NSS DB:
```bash
sudo apt install libnss3-tools           # if needed
certutil -d sql:$HOME/.pki/nssdb -A -t "C,," -n "roeden-ca" -i artifacts/roeden-ca.crt
```

Firefox (no clean CLI): Settings → Privacy & Security → Certificates →
View Certificates → Authorities → Import → pick `artifacts/roeden-ca.crt` →
tick "Trust this CA to identify websites" → OK.

Restart browsers fully.

### 10. Set up your ArgoCD login

See the next section.

---

## Setting up ArgoCD with your own login (one-time, after services.yml)

ArgoCD ships with a built-in `admin` user. We want the `lroe` user (declared
in `roles/k3s/argocd/templates/values.yaml.j2`) to be the only usable
identity, with admin disabled. ArgoCD doesn't let you create accounts via
CLI — config has to declare them, then CLI sets the password.

### Phase 1: temporarily leave `admin` enabled so you can set lroe's password

Open `roles/k3s/argocd/templates/values.yaml.j2` and make sure
`admin.enabled: "false"` is **commented out** (or absent). If you flipped
it already, comment it back:

```yaml
configs:
  cm:
    accounts.lroe: "apiKey, login"
    # admin.enabled: "false"     # leave commented during initial bootstrap
```

Apply:
```bash
ansible-playbook -i inventories/roeden/hosts.yml plays/k3s/services.yml
```

### Phase 2: login as admin, set lroe's password, verify

```bash
# Grab the auto-generated admin password
INITIAL_PW=$(kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d)

# Login as admin via argocd CLI
argocd login argocd.roeden.lab --username admin --password "$INITIAL_PW"

# Set lroe's password — pick something strong, store in your password manager
argocd account update-password \
  --account lroe \
  --current-password "$INITIAL_PW" \
  --new-password 'PASTE-NEW-PASSWORD-HERE'

# Verify lroe can log in AND has admin RBAC
argocd logout argocd.roeden.lab
argocd login argocd.roeden.lab --username lroe --password 'PASTE-NEW-PASSWORD-HERE'
argocd account list
# Should list admin AND lroe; lroe with capabilities "apiKey, login"
argocd app list
# Empty list (no apps yet) — but no permission error means RBAC worked.
```

### Phase 3: disable admin permanently

Edit `roles/k3s/argocd/templates/values.yaml.j2` — **uncomment** the
`admin.enabled: "false"` line:

```yaml
configs:
  cm:
    accounts.lroe: "apiKey, login"
    admin.enabled: "false"
```

Apply:
```bash
ansible-playbook -i inventories/roeden/hosts.yml plays/k3s/services.yml
```

Delete the now-useless initial admin secret:
```bash
kubectl -n argocd delete secret argocd-initial-admin-secret
```

Confirm admin is locked out:
```bash
argocd login argocd.roeden.lab --username admin --password "$INITIAL_PW"
# Expect: "permission denied" / "account disabled"
```

From this point forward only `lroe` can log in. Password is stored as a
bcrypt hash in the `argocd-secret` Kubernetes Secret, replicated across all
three control planes via etcd. It survives pod restarts, chart upgrades,
node reboots — anything short of `kubectl delete ns argocd`.

### Re-enabling admin (if you ever need to)

Edit `values.yaml.j2`, comment the `admin.enabled: "false"` line back out,
re-run `services.yml`. Then generate a new initial admin secret:
```bash
kubectl -n argocd patch secret argocd-secret \
  -p '{"stringData": {"admin.password": "$2a$10$NEWBCRYPTHASH...", "admin.passwordMtime": "'$(date +%FT%T%Z)'"}}'
```
(Or just bcrypt-hash a known password and patch it in.)

### Rotating lroe's password later

```bash
argocd login argocd.roeden.lab --username lroe --password 'OLD-PW'
argocd account update-password \
  --account lroe \
  --current-password 'OLD-PW' \
  --new-password 'NEW-PW'
```

No ansible run needed — the password lives in `argocd-secret` (in etcd).

---

## Kubernetes update operations

### Check what's behind (weekly read-only)

```bash
ansible-playbook -i inventories/roeden/hosts.yml plays/k3s/check-updates.yml
```

Output: latest released version vs. configured version for every tracked
component (k3s itself per node, kube-vip, Traefik chart, cert-manager,
Gateway API CRDs, NFS provisioner, ArgoCD). Each line marked
`(up to date)` or `(behind, latest X.Y.Z)`.

### Apply Kubernetes upgrades (weekly, even if no version bumps)

The upgrade play has two purposes:
1. Roll out any version bumps you made in `inventories/roeden/group_vars/all.yml`
2. Renew the internal leaf cert if it's within 30 days of expiry

So you run it weekly even when no versions changed — the leaf cert
renewal phase still does useful work.

**To bump versions**: edit `inventories/roeden/group_vars/all.yml`:
```yaml
k3s_version: "v1.36.1+k3s1"
kubevip_image: "ghcr.io/kube-vip/kube-vip:v1.2.0"
traefik_chart_version: "v40.2.0"
cert_manager_chart_version: "v1.19.5"
gateway_api_version: v1.5.1
nfs_provisioner_chart_version: "4.0.18"
argocd_chart_version: "7.7.0"
```

(Use `check-updates.yml` first to see what to bump.)

**Run the upgrade**:
```bash
ansible-playbook -i inventories/roeden/hosts.yml plays/k3s/upgrade.yml
```

Phases, in order:
1. **kube-vip** — applied first, API VIP verified before touching k3s
2. **k3s control planes** — `serial: 1`, drained + reinstalled per node
3. **k3s agents** — `serial: 1`, same
4. **Gateway API CRDs** — re-applied with `server-side apply`
5. **cert-manager** — helm upgrade if behind target
6. **Traefik** (both instances) — helm upgrade if behind target
7. **NFS provisioner** — helm upgrade if behind target
8. **ArgoCD** — helm upgrade if behind target
9. **Internal leaf cert** — renew if expiring within 30 days, then push
   the fresh Secret to both `traefik` and `argocd` namespaces

Each component's phase fast-skips when the configured version matches
deployed. Safe to run repeatedly.

### k3s version policy

Follow `n-1`: when a new minor releases upstream (e.g., 1.37 drops),
upgrade to the *previous* minor (1.36). Never skip more than one minor at
a time — kubelet/control-plane skew is supported across n-2, k3s upgrade
expects sequential minors.

---

## Adding a new app (Gateway API, internal)

Concept-level checklist for a typical internal app:

1. App's Helm chart produces:
   - Deployment + Service (standard)
   - `HTTPRoute` referencing the **internal** Gateway:
     ```yaml
     parentRefs:
       - name: internal
         namespace: traefik
     hostnames:
       - myapp.roeden.lab
     ```
2. PVCs use no `storageClassName` (NFS is the default).
3. DNS already covers `*.roeden.lab → 192.168.10.51`, so the hostname
   resolves automatically.
4. Browser already trusts `*.roeden.lab` because we installed the CA.

Nothing in this repo needs to change. The app's chart is fully
self-contained. For an *external* app: same pattern, `parentRefs.name:
external` and a public hostname under `*.roedev.com`.

---

## Glossary of "what runs where"

- **Controller (your laptop)**: where you run `ansible-playbook`. Reads
  `artifacts/` directly (cert, kubeconfig, tokens). Does NOT need helm or
  kubernetes Python lib (those run on cp1).
- **`k3s-cp1`**: the Ansible target for all `kubernetes.core.helm` and
  `kubernetes.core.k8s` tasks. Has `/etc/rancher/k3s/k3s.yaml`. Files
  needed from the controller are read via Jinja `lookup` and shipped over.
- **All k3s nodes**: targets for OS update plays + trust-cert push.

---

## Recovery / "I forgot the password"

### Lost lroe's password but admin is still enabled

Use the admin path from "Phase 2" above.

### Lost lroe's password AND admin is disabled

Patch the bcrypt hash directly in `argocd-secret`:
```bash
HASH=$(htpasswd -bnBC 10 "" 'new-password' | tr -d ':\n' | sed 's/^\$2y/$2a/')
kubectl -n argocd patch secret argocd-secret \
  -p "{\"stringData\":{\"accounts.lroe.password\":\"$HASH\",\"accounts.lroe.passwordMtime\":\"$(date +%FT%T%Z)\"}}"
```
ArgoCD reads the new hash on its next reconcile (within seconds).
