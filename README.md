# Roeden Lab — Ansible Operations

Ansible plays and roles for running the `roeden` homelab. Three independent
infrastructure tiers, each with its own bootstrap / check-updates / upgrade
lifecycle:

- **Postgres tier** (3 VMs: pg-01/02/03) — HA Postgres via Patroni + etcd
  quorum. Source of truth for any app needing a real database (Authentik,
  Grafana, future apps).
- **k3s cluster** (3 cps + 4 agents) — kubernetes platform. Runs Traefik
  ingress, cert-manager, NFS storage class, ArgoCD, kube-prometheus-stack
  (Prom + Alertmanager + Grafana), Loki + Alloy (logs), Tempo (traces),
  Authentik (SSO), OpenBao (secrets / dynamic Postgres creds), Gitea
  (self-hosted Git + container registry + Helm chart registry).
- **Lab tier** — every host across both above. Shared OS-level concerns:
  apt updates with rolling reboots, internal CA trust distribution.

Use this README when future-you doesn't remember which play to run. Every
section is a working command sequence — copy/paste them in order.

---

## Repo layout cheat sheet

```
inventories/roeden/
  hosts.yml                # node IPs + SSH user (k3s_servers/agents + postgres_servers)
  group_vars/all.yml       # tracked versions + cluster vars
  host_vars/k3s-cp1.yml    # k3s-cp1 specifics (cluster_init: true)

plays/
  postgres/
    bootstrap.yml          # one-time: 3-VM Postgres HA cluster (etcd + pg + patroni)
    check-updates.yml      # READ-ONLY: drift report for etcd/Patroni/Postgres
    upgrade.yml            # rolling upgrade of etcd / Patroni / pg minor versions
  k3s/
    bootstrap.yml          # one-time: stand up the k3s cluster from scratch
    ingress.yml            # idempotent: Traefik + cert-manager + Gateways
    services.yml           # idempotent: NFS + ArgoCD + KPS + Loki + Alloy + Tempo + Authentik + OpenBao + Gitea
    unseal-openbao.yml     # standalone: re-unseal OpenBao pods after restart/eviction
    reconfigure.yml        # rolling restart for k3s node-level config changes
    check-updates.yml      # READ-ONLY: drift report for every tracked component
    upgrade.yml            # weekly: rolling k3s/component upgrades + cert renewal
  lab/
    check-os-updates.yml   # READ-ONLY: apt updates available per host (all VMs)
    rolling-os-updates.yml # rolling apt upgrade + reboot (all VMs, drains k3s nodes)
    trust-internal-cert.yml # push the internal CA cert to every host

roles/
  postgres/setup/          # per-VM: etcd + Postgres + Patroni
  k3s/...                  # k3s install + every cluster-tier role
  lab/...                  # OS update + trust-cert roles (shared across all VMs)
  local/
    ensure_random_secrets  # auto-generate random secrets in artifacts/ if missing
    require_user_secrets   # fail with instructions if user-provided files missing

artifacts/                 # gitignored — holds secrets, certs, and the kubeconfig
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

### Files in `artifacts/` — three categories

`artifacts/` is gitignored. Three different categories of files end up there:

**1. User-provided (you must create these; plays will hard-fail with
instructions if missing).** Two roles enforce this — `require_user_secrets`
on the controller side prints exact "how to make it" instructions if you
forget.

| File | What it is | How to create |
|---|---|---|
| `roeden-ca.crt` | Internal CA cert (trust anchor) | One-time openssl command, see "Initial setup" step 4 |
| `roeden-ca.key` | Internal CA private key | Same openssl command |
| `cloudflare-api-token` | Cloudflare DNS-01 API token | Generate in Cloudflare dashboard (My Profile → API Tokens → Edit zone DNS) |
| `discord-webhook-url` | Discord webhook for Alertmanager → Discord | Channel settings → Integrations → Webhooks → New Webhook → Copy URL |

**2. Auto-generated random secrets (plays create these if missing).** The
`ensure_random_secrets` role checks at play start and generates with
`openssl rand` if absent. You shouldn't ever need to make these by hand.

| File | What it is | Created by |
|---|---|---|
| `k3s-token` | k3s cluster join token | `plays/k3s/bootstrap.yml` |
| `authentik-secret-key` | Authentik session/token signing key | `plays/k3s/services.yml` |
| `authentik-postgres-password` | Authentik's Postgres password | `plays/k3s/services.yml` |
| `grafana-postgres-password` | Grafana's Postgres password | `plays/k3s/services.yml` |
| `valkey-password` | Shared Valkey cache password | `plays/k3s/services.yml` |
| `grafana-oidc-client-secret` | Grafana OIDC client_secret (shared with Authentik) | `plays/k3s/services.yml` |
| `argocd-oidc-client-secret` | ArgoCD OIDC client_secret (shared with Authentik) | `plays/k3s/services.yml` |
| `openbao-oidc-client-secret` | OpenBao OIDC client_secret (shared with Authentik) | `plays/k3s/services.yml` |
| `openbao-vault-admin-postgres-password` | Password for the `vault_admin` Postgres role that OpenBao's database engine uses | `plays/k3s/services.yml` |
| `gitea-postgres-password` | Gitea's Postgres password | `plays/k3s/services.yml` |
| `gitea-oidc-client-secret` | Gitea OIDC client_secret (shared with Authentik) | `plays/k3s/services.yml` |
| `gitea-jwt-secret` | Gitea OAuth2 JWT signing key | `plays/k3s/services.yml` |
| `gitea-secret-key` | Gitea's internal SECRET_KEY — encrypts sensitive fields in the DB | `plays/k3s/services.yml` |
| `gitea-internal-token` | Gitea's internal-API auth token | `plays/k3s/services.yml` |
| `postgres-superuser-password` | Postgres cluster admin password | `plays/postgres/bootstrap.yml` |
| `postgres-replication-password` | Patroni's replication user password | `plays/postgres/bootstrap.yml` |

**Special — OpenBao init keys (`openbao-init.json`).** Generated on the very
first run of the OpenBao role; **NOT regenerable.** Contains the 5 Shamir
unseal keys and the root token. If this file is deleted with no backup,
the OpenBao cluster CANNOT be unsealed and all secrets stored in it are
permanently lost. Treat it like the CA private key: back it up out of band.

**3. Outputs from play runs (just show up; harmless).**

| File | What it is | Created by |
|---|---|---|
| `internal-tls.crt` + `.key` | Server cert for `*.roeden.lab`, signed by the CA | `plays/k3s/ingress.yml` (or `upgrade.yml`) — auto-generated and auto-renewed |
| `kubeconfig-roeden.yaml` | k3s admin kubeconfig | `plays/k3s/bootstrap.yml` fetches it from cp1 |
| `roeden-ca.srl` | openssl serial-number tracker for cert signing | First time the CA signs a cert; harmless |

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

## Initial setup (fresh-from-zero, once)

The order matters: Postgres before k3s, because some k3s services (Authentik,
eventually others) treat Postgres as a precondition. Skip this section if
the cluster already exists.

### 1. Inventory: confirm the host list

`inventories/roeden/hosts.yml` should list three groups:
- `k3s_servers` (cp1–cp3) at 192.168.10.21–23
- `k3s_agents` (w1–w4) at 192.168.10.31–34
- `postgres_servers` (pg-01/02/03) at 192.168.10.41–43

All inherit `ansible_user: infraadmin` + the SSH key from group `all` vars.

### 2. Provision VMs (Proxmox manual — outside ansible's scope)

In Proxmox: clone your cloud-init template for each host. SSH key + user
should match `infraadmin` from the inventory.

- **k3s nodes**: 4 GB RAM, 2 vCPU, 20 GB disk
- **Postgres nodes**: 4 GB RAM, 2 vCPU, 50 GB local-ssd disk (one VM per Proxmox host for true HA — pg-01 on proxmox-1, pg-02 on proxmox-2, etc.)

Sanity check controller → all hosts:
```bash
ansible -i inventories/roeden/hosts.yml all -m ping
# Expect all 10 hosts respond
```

### 3. Drop user-provided secrets into `artifacts/`

```bash
# Cloudflare API token (scope: Zone DNS Edit for your domain only)
# Generate at https://dash.cloudflare.com → My Profile → API Tokens
echo -n 'PASTE-TOKEN-HERE' > artifacts/cloudflare-api-token
chmod 600 artifacts/cloudflare-api-token

# Discord webhook URL (Alertmanager will pipe alerts here)
# Discord: channel settings → Integrations → Webhooks → New Webhook
echo -n 'PASTE-WEBHOOK-URL-HERE' > artifacts/discord-webhook-url
chmod 600 artifacts/discord-webhook-url

# Gitea PAT for cluster-puller (created after Gitea is up — see step 10.5)
echo -n 'PASTE-PAT-HERE' > artifacts/cluster-puller-token
chmod 600 artifacts/cluster-puller-token
```

Random secrets like the k3s join token and Postgres passwords don't go here
— they're auto-generated on first play run if missing.

### 4. Generate the internal CA

The CA is long-lived (10 years) and only signs the leaf cert. The leaf cert
itself gets auto-generated on first `ingress.yml` run.

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

### 5. Push the internal CA to all hosts (k3s + postgres VMs both)

```bash
ansible-playbook -i inventories/roeden/hosts.yml plays/lab/trust-internal-cert.yml
```

System trust store on every host now trusts `*.roeden.lab`.

### 6. Bootstrap the Postgres HA cluster

```bash
ansible-playbook -i inventories/roeden/hosts.yml plays/postgres/bootstrap.yml
```

What this runs: the localhost play auto-generates Postgres passwords if
missing, then on all 3 VMs in parallel — install etcd 3.5.x from tarball,
bootstrap a 3-node etcd cluster, add PGDG apt repo, install Postgres 16,
disable the systemd PG service (Patroni controls it), pip-install Patroni
in a `/opt/patroni` venv, render Patroni config, start Patroni. Final
verification waits for a Leader to be elected and prints `patronictl list`.

Verify:
```bash
ssh infraadmin@192.168.10.41 -i ~/.ssh/roeden_infra \
  '/opt/patroni/bin/patronictl -c /etc/patroni/patroni.yml list'
# Expect: 1 Leader (state: running) + 2 Replicas (state: streaming)
```

### 7. Bootstrap k3s

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

### 8. Install the ingress tier

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

### 9. Set DNS

- **Internal DNS** (the NAS at 192.168.0.29 serves it): wildcard
  `*.roeden.lab → 192.168.10.51`. Also serves as upstream for pods —
  CoreDNS forwards there for both `*.roeden.lab` and external lookups.
- **Public DNS** (Cloudflare): A record for whatever public hostname you'll
  use (e.g. `home.roedev.com → <your public IP>`, **DNS-only**, grey cloud).
- **Router NAT**: TDS forwards `:80`/`:443` → `192.168.0.22`, EdgeRouter
  forwards same ports → `192.168.10.52` *only when destination is
  `192.168.0.22`* (so internal `.51` traffic isn't hijacked).

### 10. Install platform services

```bash
ansible-playbook -i inventories/roeden/hosts.yml plays/k3s/services.yml
```

Everything that lands in this run:
1. **NFS provisioner** — `StorageClass nfs-client` marked default
2. **ArgoCD** — Ingress at `argocd.roeden.lab`
3. **kube-prometheus-stack** — Prometheus + Alertmanager + Grafana,
   Ingress at `grafana.roeden.lab`, Alertmanager pointed at Discord webhook
4. **Loki** — single-binary log aggregator on NFS, 14d retention
5. **Tempo** — distributed tracing on NFS, 7d retention
6. **Grafana Alloy** — DaemonSet: collects every pod's stdout to Loki AND
   receives OTLP traces from apps (forwards to Tempo)
7. **Authentik** — Ingress at `auth.roeden.lab`, SSO identity provider
8. **OpenBao** — Ingress at `vault.roeden.lab`, secrets management. 3-pod
   raft cluster on NFS PVCs. First run auto-initializes and saves Shamir
   keys to `artifacts/openbao-init.json`, then unseals all pods. Subsequent
   runs are idempotent (init skipped if keys file present, unseal skipped
   if pods already unsealed).
9. **Gitea** — Git + built-in container registry + built-in Helm chart
   registry. Internal Ingress at `git.roeden.lab`, external HTTPRoute at
   `git.roedev.com`. OIDC against `auth.roedev.com` (works for both internal
   and external users — internal logins hairpin-NAT through CF on auth).
   Built-in registries replace standalone Docker + ChartMuseum.

Verify:
```bash
kubectl get storageclass
# nfs-client (default)

kubectl -n argocd get pods
# All Running

kubectl -n argocd get ingress
# argocd-server   traefik-internal   argocd.roeden.lab   192.168.10.51

kubectl -n observability get pods
# kps-grafana, prometheus-kps-..., alertmanager-kps-...,
# kps-prometheus-node-exporter-* (one per node), kube-state-metrics,
# loki-0 (2/2 Running), alloy-* (one per node) — all Running
```

### 10.5 Install cluster registry pull credentials

After Gitea is up (it gets installed by `services.yml` in step 10), the
cluster needs credentials to pull private container images. This step
does it **once at the node level** so every pod everywhere can pull from
Gitea with no per-app `imagePullSecrets`.

#### The policy (one decision, no per-friend recurrence)

All deployable container images live in the **`roeden` Gitea org**:

```
git.roedev.com/roeden/<image>:<tag>
```

That's it. Personal namespaces are for source repos and the occasional
experimental package; anything ArgoCD ever pulls comes from `roeden`.
Repos under `roeden` stay **private** (org-private packages — outsiders
can't pull). Everyone with push access to the platform's deployable
images is an `roeden` org member; new friend joins = invite them to
`roeden` once.

#### Set up cluster-puller (one-time)

The cluster authenticates to Gitea as a service-account user that's
**also** a member of `roeden` (so it can read the org's private packages).

1. Sign in to https://git.roedev.com as your admin user
2. Site Administration → Users → Create User
   - Username: `cluster-puller`
3. `roeden` org → People → Invite → add `cluster-puller`
4. Sign in as `cluster-puller` (or impersonate via admin)
5. Settings → Applications → Generate New Token
   - Name: `k3s registries.yaml`
   - Scope: **`read:package`**
6. Copy the token into `artifacts/cluster-puller-token` on the controller
   (already done in step 3 if you knew the value at that point)

#### Apply

```bash
ansible-playbook -i inventories/roeden/hosts.yml plays/k3s/registry-auth.yml
```

Templates `/etc/rancher/k3s/registries.yaml` on every k3s node, then
restarts each node's k3s (or k3s-agent) serially. Existing workloads stay
up during the rolling restart — only the k3s control plane briefly
bounces on the node being updated.

#### Verify

```bash
# Check the file landed on a node
ssh infraadmin@k3s-w1 sudo cat /etc/rancher/k3s/registries.yaml

# Force a fresh pull of a private Gitea-hosted image
kubectl run pull-test --rm -i --image=git.roedev.com/roeden/<some-image>:<tag> \
  --restart=Never -- echo "pull worked"
```

#### Adding a new friend

1. Gitea admin → invite them to the `roeden` org with write access
2. They build + push to `git.roedev.com/roeden/<their-app>:<tag>`
3. Their image is immediately pullable by the cluster, no platform-side
   change. (Re-running `registry-auth.yml` is **not** required.)

#### Rotating the cluster-puller PAT

Replace `artifacts/cluster-puller-token`, re-run
`plays/k3s/registry-auth.yml`. The serial restart applies the new value
across nodes without a workload outage.

### 11. Trust the CA on your laptop

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

### 12. Authentik first-login + SSO wiring

See the "Authentik SSO & blueprints" section below for the akadmin
recovery → lroe-as-admin → SSO-into-apps walkthrough.

---

## Authentik SSO & blueprints (the identity story)

All app login goes through Authentik. ArgoCD, Grafana, and any future
internal app trust Authentik as their OIDC identity provider — no per-app
local accounts to manage, no separate passwords. The single human step
on a fresh bootstrap is "set your Authentik password the first time."

### Two channels for declarative Authentik config

Authentik watches `/blueprints/` and applies any YAML it finds. SSO wiring
for each app — provider, application, group, scope mappings — is expressed
as a blueprint. No clicking through the UI, fully reproducible from scratch.

**Platform channel — Ansible-managed.** Files at
`roles/k3s/authentik/templates/blueprints/`. The Authentik role renders
each template into a single ConfigMap; each blueprint is mounted at
`/blueprints/platform/<name>.yaml` via a subPath mount (subPath is
required because plain ConfigMap mounts produce a `..data` symlink dir
that Authentik's discovery treats as hidden and silently skips). Currently
in this channel: `grafana-oidc.yaml.j2`, `argocd-oidc.yaml.j2`.

**Apps channel — app-managed, no Ansible touching.** NFS PVC
`authentik-apps-blueprints` in the `authentik` namespace, mounted at
`/blueprints/apps/`. Custom apps drop their own `*.yaml` files in here
at deploy time. Authentik picks them up the same as platform blueprints.

### To add a new platform-tier app to SSO (Ansible side)

1. Create `roles/k3s/authentik/templates/blueprints/<app>-oidc.yaml.j2`.
   Copy `grafana-oidc.yaml.j2` as a template; change names, scopes, the
   redirect_uri, and the `!Env <APP>_OIDC_CLIENT_SECRET` reference.
2. In `roles/k3s/authentik/tasks/main.yml`:
   - Add a new key under the platform-blueprints ConfigMap data
     (`<app>-oidc.yaml: "{{ lookup('template', ...) }}"`)
   - Add a Secret creation task for `<app>-oidc-secret`
3. In `roles/k3s/authentik/templates/values.yaml.j2`:
   - Append the new template to the `blueprints_hash` calculation
   - Add a new subPath mount under `global.volumeMounts`
   - Add a new env var injection under `global.env` referencing the Secret
4. In `inventories/roeden/group_vars/all.yml`:
   - Add `<app>_oidc_client_secret_path` and `<app>_admins_group_name`
5. In `plays/k3s/services.yml`:
   - Add the client-secret artifact to the `ensure_random_secrets` list
6. Wire the app's own chart to use OIDC against Authentik (see how
   `roles/k3s/argocd/` and `roles/k3s/kube_prometheus_stack/` do it for
   their respective charts — both create a matching Secret in the app's
   namespace and reference it from the chart values).
7. Run `services.yml`. The Authentik role's final task explicitly fires
   blueprint discovery — when the playbook exits, the new SSO is live.

### To wire a custom app you deploy via ArgoCD (apps channel)

For apps you write later that don't have an Ansible role, the goal is
"deploy the app → Authentik knows about it → users can SSO into it." Steps:

1. Generate an OIDC client_secret in your app's deployment manifests
   (random, stored in a Secret in your app's namespace).
2. Mirror that Secret into the `authentik` namespace (so the blueprint can
   read it as an env var via `!Env`). Either replicate the Secret via
   external-secrets/Reflector, or have your app's deploy include a small
   resource in the authentik namespace.
3. **Use a Job in the authentik namespace** (or a pre-sync hook in your
   ArgoCD Application) that:
   - Mounts the `authentik-apps-blueprints` PVC
   - Writes `/blueprints/apps/<your-app>.yaml` with the blueprint content
   - **Triggers Authentik blueprint discovery via `ak shell`** (see below)
4. Configure your app's chart for OIDC against
   `https://auth.roeden.lab/application/o/<your-app>/`, using the same
   client_secret you generated in step 1.

The Job-in-authentik-ns approach sidesteps cross-namespace PVC mounting
(which is a pain — the PVC lives in `authentik` and your app probably
doesn't). You're effectively saying "to register with the identity
provider, run a small init step in the IdP's namespace." Clean separation.

### The discovery trigger — don't forget this

Authentik scans `/blueprints/` on a schedule (every ~10–15 minutes by
default) AND when the worker pod starts. The pod-startup scan has an
observed race where freshly-mounted blueprints can be silently skipped on
the first scan; we hit it during initial setup.

For platform blueprints, this is handled — the Authentik role's final
task in `services.yml` explicitly fires discovery, so the playbook is
deterministic: when it exits, blueprints are applied.

For apps-channel blueprints written by your custom apps, **your app
deployment must trigger discovery itself** or accept the 10-minute wait
for Authentik's scheduled scan. The trigger is a one-liner:

```bash
kubectl -n authentik exec deploy/authentik-worker -- \
  ak shell -c "from authentik.blueprints.v1.tasks import blueprints_discovery; blueprints_discovery.send()"
```

Bake this into your blueprint-writing initContainer/Job. Without it,
"deploy app → log in" works "eventually." With it, it works immediately.

### First login, on a fresh bootstrap

The Authentik chart bootstraps a single break-glass user `akadmin` whose
password lives in a k8s Secret. Steps once Authentik is up:

```bash
# Generate a one-shot recovery URL for akadmin
kubectl -n authentik exec deploy/authentik-worker -- \
  ak create_recovery_key 1 akadmin
```

Browse the URL, set akadmin's password, log into `auth.roeden.lab`, then:

- *Directory → Users → Create* → username `lroe`, your email, type Internal
- *Users → lroe → Set password*
- *Directory → Groups* — `grafana-admins` and `argocd-admins` already exist
  (created by the blueprints). Add `lroe` to whichever groups you want.
- Log out as akadmin, log in as lroe, confirm Admin Interface access works.
- *Users → akadmin → Edit → uncheck "Is active" → Save*. Break-glass account
  neutralized; you (lroe) are the only meaningful identity.

### Disabling app-level admin accounts

Once SSO works, the per-app local admin accounts (ArgoCD's `admin`,
Grafana's `admin`) are unnecessary. Both are already disabled in this
repo's chart values (`admin.enabled: "false"` for ArgoCD; Grafana's login
form is hidden via `disable_login_form: true`). The chart's auto-generated
admin password Secrets still exist in each namespace as break-glass — if
SSO breaks, you can re-enable admin and log in with that password.

### Internal vs external SSO (which Authentik URL to point at)

The same Authentik instance is reachable at **two** hostnames:

- `auth.roeden.lab` — served by the chart's built-in Ingress through
  `traefik-internal`, internal CA cert. Use for apps on `*.roeden.lab`.
- `auth.roedev.com` — served by an HTTPRoute attached to the external
  Gateway (`traefik-external`), Let's Encrypt cert via the wildcard
  `*.roedev.com` cert that lives on that Gateway. Use for apps on
  `*.roedev.com`.

Same identity store, same blueprints, same users. **Sessions don't span
the two hostnames** — cookies are domain-scoped, so logging in via
`auth.roeden.lab` doesn't carry to `auth.roedev.com` and vice versa. For
a user who hits both internal and external apps, that's two logins per
work session. Acceptable trade-off for the resilience win (internal SSO
survives WAN/Cloudflare outages).

Rule of thumb for picking the issuer when adding a new app:

- App's Ingress is `traefik-internal` → app's OIDC issuer is `auth.roeden.lab`
- App's HTTPRoute is `traefik-external` → app's OIDC issuer is `auth.roedev.com`

Always match: the issuer URL the app's OIDC client discovers MUST be
reachable from the same network/internet position the app's users are on.

Prerequisites for the external path (one-time):

- **Public DNS**: `auth.roedev.com` resolves to your home public IP. The
  existing wildcard `*.roedev.com` A record (at your registrar) already
  handles this — no per-host record needed.
- **No new cert needed** — the wildcard `*.roedev.com` cert on the external
  Gateway already covers `auth.roedev.com`.
- **No internal DNS override** — by design. Internal users who want LAN-
  direct access use `auth.roeden.lab`. Internal users who hit `auth.roedev.com`
  hairpin out through the WAN and back in via the public path. Keeping the
  two paths actually distinct (no split-horizon shortcut) means "internal"
  and "external" stay honest categories rather than blurring into a middle
  case. Requires hairpin NAT on the edge router (EdgeOS: on by default for
  destination-NAT rules).

### Adding an external-facing app (preview — codified once the helm chart lands)

For an app you're exposing on `*.roedev.com` that wants SSO:

1. App's `HTTPRoute` attaches to the **external** Gateway with hostname
   `<app>.roedev.com`.
2. App's OIDC blueprint sets `oidc_discovery_url=https://auth.roedev.com/application/o/<app>/`
   and `redirect_uris` pointing at `https://<app>.roedev.com/...`.
3. Everything else (group binding, scope mapping, blueprint application
   via the apps channel) is identical to the internal pattern.

The forthcoming app helm chart bakes this in as a single `exposure:
internal|external` value — chart picks the right hostname, Gateway, and
issuer URL based on that one knob.

---

## OpenBao — secrets, dynamic Postgres creds, unseal ops

OpenBao (Apache-2.0 fork of Vault) is the platform's secrets manager.
Apps use it via:
- The **database secrets engine** for short-lived per-app Postgres
  credentials. Apps register their own DB role at deploy time and request
  fresh creds on startup — no static passwords in app config.
- The **Kubernetes auth method**: apps authenticate via their
  ServiceAccount token; no static OpenBao credentials in app config either.
- The **OIDC auth method** (humans): log into the UI at
  `https://vault.roeden.lab/` via Authentik. Members of the
  `openbao-admins` Authentik group get the built-in `admin` policy.

3-pod raft cluster on NFS-backed PVCs. The Ansible role's `tasks/main.yml`
handles install, conditional init, unseal, and configure — all idempotent.

### First-run init: what happens, what to back up

The first time `plays/k3s/services.yml` runs the openbao role, openbao-0
hits `bao operator init`, gets back JSON with 5 unseal keys + the root
token, and we save the whole blob to `artifacts/openbao-init.json` on the
controller (mode 0600).

That file is **the one and only way to unseal the cluster** after a pod
restart. Subsequent role runs detect it (`stat`) and skip the init step —
they just read it back for unseal.

If you lose `artifacts/openbao-init.json` AND all pods are sealed, the
cluster is unrecoverable. Back this file up out-of-band, same as you
back up `artifacts/roeden-ca.key`.

### Unseal play (catastrophic restart)

When something restarts an OpenBao pod outside an upgrade window — node
power-cycle, eviction, manual `kubectl delete pod` — that pod comes back
sealed. Sealed pods serve no API requests until unsealed.

To re-unseal without re-running the full services playbook:

```bash
ansible-playbook -i inventories/roeden/hosts.yml plays/k3s/unseal-openbao.yml
```

Reads `artifacts/openbao-init.json`, checks each pod's seal status, submits
the first 3 unseal keys to any sealed pod. Idempotent — already-unsealed
pods are no-ops. Takes <30 seconds.

`plays/k3s/upgrade.yml` also sweeps for sealed pods every run, so weekly
upgrade runs catch this automatically. The standalone play exists for
the cases where you don't want a full upgrade.

### Lost the init keys file

If `artifacts/openbao-init.json` is gone AND some (or all) pods are
sealed, those pods stay sealed forever. **There is no other recovery
path** — Shamir keys are not derivable from anything else. Any secret
material stored only in OpenBao is lost.

Reset path: delete the OpenBao PVCs, `helm uninstall openbao`, re-run the
role. Fresh cluster, no historical secrets, no continuity.

---

## Gitea — Git, container images, helm charts

Gitea is the self-hosted home for everything that used to require three
separate services: source control, container registry, Helm chart
registry. Single binary, one Postgres DB, OIDC via Authentik.

**Reachable only at `https://git.roedev.com/`** (external, via traefik-external
+ the *.roedev.com Let's Encrypt cert). No internal hostname.

### Why external-only — the SSO single-side rule

Any service that uses Authentik SSO sits on exactly ONE network position
(internal or external), never both. This isn't a stylistic choice — it's a
hard architectural constraint:

- OAuth-style flows have a single canonical redirect URI tied to the app's
  configured ROOT_URL.
- The OIDC provider bounces the browser back to that one URI regardless of
  where the user started the flow.
- So a "dual URL" service ends up canonicalizing on whichever URL is
  declared as ROOT_URL — the other URL just becomes a confusing entry point
  that always dumps you on the canonical one anyway.
- Trying to make both sides work cleanly would require host-aware redirect
  rewriting, per-host session cookies, and a fragile reverse-proxy layer.
  Not worth it.

So the rule: **SSO-using app picks one side**. Gitea picks external because
we want push/pull access from anywhere (CI, remote work, sharing with
external collaborators). LAN users pay a single Cloudflare roundtrip on
auth, and a hairpin-NAT cost on subsequent Git operations — small price
for the architectural consistency.

(Authentik itself is the exception: it gets dual URLs because it IS the
OIDC provider — it has no callbacks of its own to canonicalize.)

The `roeden-app` chart bakes this rule in at the template level: if you
set `sso.enabled: true` without enabling `expose`, helm install fails
loudly. There's no way to accidentally end up with the dual-URL anti-pattern
for an SSO service via the chart.

### Authentication

Gitea uses Authentik OIDC against `auth.roedev.com`. Login flow: click
"Sign in with Authentik" → bounce to Authentik → consent → back to Gitea
→ first-time users auto-register (no signup form), subsequent users just
log in. Group `gitea-admins` membership in Authentik grants Gitea Admin.

Members of the Authentik group `gitea-admins` get the Gitea Admin role
automatically (via the `groupClaimName` + `adminGroup` config in
`roles/k3s/gitea/templates/values.yaml.j2`). To grant yourself admin:

1. Authentik admin UI → `gitea-admins` group → Add `lroe`
2. Log in to Gitea via the SSO button — admin role applied on first login

### Push a container image

```bash
# Log in. Username is your Gitea username (auto-created on first SSO login).
# Password is a personal access token — generate one in Gitea: Settings → Applications.
docker login git.roedev.com -u lroe

# Tag + push. The registry path is git.roedev.com/<owner>/<image-name>:<tag>.
docker tag myapp:latest git.roedev.com/lroe/myapp:1.0.0
docker push git.roedev.com/lroe/myapp:1.0.0
```

Pulls in-cluster use the same URL — k3s containerd has internet access and
can reach `git.roedev.com` via the wildcard external A record / hairpin NAT.
If you want to skip the hairpin for in-cluster pulls, retag with
`git.roeden.lab` and push there too (same registry, both URLs work).

### Push a Helm chart

Gitea supports Helm charts via the OCI distribution spec. From the
roeden-app chart repo:

```bash
cd charts/roeden-app
helm package .                  # produces roeden-app-0.1.0.tgz
helm registry login git.roedev.com -u lroe
helm push roeden-app-0.1.0.tgz oci://git.roedev.com/lroe/charts
```

And to install from it:

```bash
helm install myapp oci://git.roedev.com/lroe/charts/roeden-app --version 0.1.0 ...
```

ArgoCD also supports OCI registries; just point an Application's source
at `repoURL: git.roedev.com/lroe/charts` with `chart: roeden-app`.

### Storage

Gitea's data dir lives on NFS (`gitea_storage_size` in `group_vars/all.yml`,
default 50 GiB). Contains: Git repos, container layer blobs, helm chart
artifacts, attachments, avatars. Backups = back up that NFS subdir + a
`pg_dump gitea` from the Postgres tier. Restoring needs both.

The pre-generated `gitea-secret-key` and `gitea-internal-token` in
`artifacts/` are required to decrypt sensitive DB fields after a restore.
Lose them AND restore the DB = unreadable. They live with the same
protection posture as `roeden-ca.key`.

### Adding a new app that wants dynamic Postgres creds

The platform-side groundwork is done (database engine configured against
`postgres-rw.postgres.svc:5432` via the `vault_admin` PG role). For an
app to opt in:

1. App's Postgres database exists. For now, append to `postgres_databases`
   in `group_vars/all.yml` and re-run `services.yml`; longer-term we'll
   move this to an app-side init container so app additions don't
   require Ansible changes (TBD with the helm chart work).
2. App's deploy registers a Vault DB role:
   `bao write database/roles/<app> ...` with the creation SQL template.
3. App's deploy creates an OpenBao role bound to the app's ServiceAccount:
   `bao write auth/kubernetes/role/<app> ...`
4. App reads dynamic creds at startup:
   `bao read database/creds/<app>` → short-lived username/password.

This whole flow becomes a helm chart helper template once the chart work
lands.

---

## Grafana — sidecar password gotcha (rare but important)

Login itself is SSO via Authentik — see the SSO section above. The
chart's auto-generated `admin` user still exists as break-glass; password
is in the `kps-grafana` Secret if you ever need to bypass SSO:

```bash
kubectl -n observability get secret kps-grafana \
  -o jsonpath='{.data.admin-password}' | base64 -d ; echo
```

### The sidecar password gotcha

Loki, Tempo, and any future datasources are auto-discovered via the
sidecar that watches ConfigMaps labeled `grafana_datasource: "1"`. The
sidecar writes the datasource file to disk AND calls Grafana's reload API
using the admin password from the `kps-grafana` Secret.

If you've **manually rotated** the admin password via Grafana's API or by
patching the Secret, the next sidecar reload call **fails with 401**.
The new datasource lands on disk but Grafana won't load it until it
restarts.

So: if you ever change the admin password manually, follow up with:

```bash
kubectl -n observability rollout restart deploy/kps-grafana
```

In the normal "SSO + don't touch the admin password" flow this never
fires — the sidecar and the Secret stay in sync. Documenting it for the
day you ever rotate that break-glass credential.

---

## Discord alerting — what gets sent and what's silenced

Alertmanager is configured (via the `kube_prometheus_stack` role) to
route every alert to the Discord webhook in `artifacts/discord-webhook-url`.
Default routing:

- `alertname = "Watchdog"` → **silenced** (it's a 5-minute heartbeat
  alert that proves the pipeline works; we drop it to a `null` receiver
  so it doesn't spam Discord)
- everything else → Discord channel, formatted with severity + description

The Discord channel notification setting matters. If you have the
channel set to "@mentions only", you won't get phone pushes — change
to "All Messages" for a dedicated alerts channel.

### Smoke-testing the path

```bash
# Terminal 1
kubectl -n observability port-forward svc/kps-kube-prometheus-stack-alertmanager 9093:9093

# Terminal 2
curl -X POST http://localhost:9093/api/v2/alerts \
  -H 'Content-Type: application/json' \
  -d '[{"labels":{"alertname":"TestAlert","severity":"info"},"annotations":{"description":"manual smoke test"}}]'

# Wait ~30s (Alertmanager group_wait) — check Discord
```

### Rotating the webhook (if leaked or just paranoid)

1. Discord → channel settings → Integrations → delete the old webhook
2. Create a new one, copy URL
3. `echo -n 'NEW-URL' > artifacts/discord-webhook-url`
4. `ansible-playbook -i inventories/roeden/hosts.yml plays/k3s/services.yml`

Alertmanager picks up the new URL via helm upgrade; no pod restart.

---

## Pod DNS: the search-domain trap (already mitigated)

This bit you in initial setup; documenting so you don't trip again. Your
nodes' `/etc/resolv.conf` has `search roeden.lab` from DHCP. K3s pods
normally inherit that search list with `ndots:5`, which means any external
hostname with fewer than 5 dots (like `discord.com`) gets `.roeden.lab`
appended and hits the wildcard `*.roeden.lab → 192.168.10.51`. Result:
every external pod request silently lands at Traefik internal, TLS handshake
returns Traefik's default cert, calls fail with cert errors.

The fix is already in this repo: `roles/k3s/common_prereqs` writes
`/etc/k3s-resolv.conf` (host nameservers minus loopbacks + `search` line),
and `roles/k3s/setup`'s config template tells kubelet to use that file
via `kubelet-arg: [resolv-conf=/etc/k3s-resolv.conf]`. Pods get a clean
resolv.conf with no `roeden.lab` search.

If you ever change `k3s_kubelet_fallback_dns` in `group_vars/all.yml`
(currently the NAS at `192.168.0.29` — which knows `*.roeden.lab` AND
forwards external lookups; using two upstreams causes ~50% NXDOMAIN
flapping because CoreDNS's `forward` policy is `random`), apply with:

```bash
ansible-playbook -i inventories/roeden/hosts.yml plays/k3s/reconfigure.yml
```

That play re-renders `/etc/k3s-resolv.conf` on every node then rolling-restarts
k3s with cordon/drain per node. Same pattern as the upgrade play, no
workload disruption.

---

## Update operations

There are two independent update lifecycles. Run check-updates on both;
upgrade either when its check shows drift.

### Check what's behind (weekly read-only)

```bash
# kubernetes tier — every chart, k3s itself, Gateway API CRDs, kube-vip
ansible-playbook -i inventories/roeden/hosts.yml plays/k3s/check-updates.yml

# postgres tier — etcd, Patroni, Postgres (running versions per host + latest released)
ansible-playbook -i inventories/roeden/hosts.yml plays/postgres/check-updates.yml
```

Both are read-only — query upstream, report drift, print `(up to date)` or
`(behind, latest X.Y.Z)`. K3s side covers: k3s per node, kube-vip, Traefik,
cert-manager, Gateway API CRDs, NFS provisioner, ArgoCD,
kube-prometheus-stack, Loki, Alloy, Tempo, Authentik, OpenBao, Gitea. Postgres
side covers etcd + Patroni + Postgres minor.

### Apply Kubernetes upgrades (weekly, even if no version bumps)

The upgrade play has two purposes:
1. Roll out any version bumps you made in `inventories/roeden/group_vars/all.yml`
2. Renew the internal leaf cert if it's within 30 days of expiry

So you run it weekly even when no versions changed — the leaf cert
renewal phase still does useful work.

**To bump versions**: edit `inventories/roeden/group_vars/all.yml`. Every
tracked component is a single var here; check-updates tells you which are
behind. (Use `plays/k3s/check-updates.yml` and `plays/postgres/check-updates.yml`
first to see what to bump.)

**Run the k3s upgrade**:
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
9. **kube-prometheus-stack** — helm upgrade if behind target
10. **Loki** — helm upgrade if behind target
11. **Tempo** — helm upgrade if behind target
12. **Alloy** — helm upgrade if behind target
13. **Authentik** — helm upgrade if behind target
14. **OpenBao** — helm upgrade if behind target. Whether or not the chart
    moves, this phase also sweeps for sealed pods and unseals them
    (covers pod rescheduling between upgrade windows). Idempotent.
15. **Gitea** — helm upgrade if behind target
16. **Internal leaf cert** — renew if expiring within 30 days, push fresh
    Secret to `traefik`, `argocd`, `observability`, `authentik`, `openbao`,
    `gitea` namespaces

Each phase fast-skips when configured matches deployed. Safe to re-run.

**Run the Postgres upgrade** (only when check shows drift):
```bash
ansible-playbook -i inventories/roeden/hosts.yml plays/postgres/upgrade.yml
```

Phases, all `serial: 1` for HA:
1. **etcd** — rolling tarball replace on each pg-XX, waits for cluster quorum
2. **Patroni** — `pip install --upgrade` in venv on each, rolling Patroni restart
3. **Postgres minor** (e.g. 16.4 → 16.5) — `apt upgrade postgresql-16`, rolling Patroni restart so Postgres is restarted under Patroni's control. Leader fails over to a replica while its node restarts.

Major Postgres upgrades (16 → 17) **aren't** in this play — they need a
manual dump-restore dance with Proxmox snapshot backstop. Document that
procedure here when the day comes.

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

### Adding observability to the same app

Free with zero config:
- **Logs**: write to stdout — Alloy's DaemonSet collects every container log
  cluster-wide. Filter in Grafana → Explore → Loki by `namespace`, `pod`,
  `container`, or `app` labels.

Opt in with a tiny chart addition:
- **Metrics**: app exposes `/metrics` (Prometheus client lib), chart ships a
  `ServiceMonitor`:
  ```yaml
  apiVersion: monitoring.coreos.com/v1
  kind: ServiceMonitor
  metadata:
    name: myapp
  spec:
    selector:
      matchLabels:
        app: myapp
    endpoints:
      - port: http
        path: /metrics
  ```
  Prometheus auto-discovers and scrapes; metrics show up in Grafana → Explore
  → Prometheus.

Alerts on your app: ship a `PrometheusRule` resource alongside, same pattern.
Alertmanager routes any firing rule to Discord automatically.

---

## Glossary of "what runs where"

- **Controller (your laptop)**: where you run `ansible-playbook`. Reads
  `artifacts/` directly (cert, kubeconfig, tokens, passwords). Does NOT
  need helm or kubernetes Python lib (those run on cp1).
- **`k3s-cp1`**: the Ansible target for all `kubernetes.core.helm` and
  `kubernetes.core.k8s` tasks. Has `/etc/rancher/k3s/k3s.yaml`. Files
  needed from the controller are read via Jinja `lookup` and shipped over.
- **`pg-01` and friends**: Postgres VMs (`postgres_servers` group). Run
  Postgres + Patroni + etcd. Ansible targets them directly for the
  `plays/postgres/*` plays. Apps in k3s reach them via TCP at
  192.168.10.41–43:5432 (eventually behind a HAProxy in front when we add
  it).
- **All hosts (k3s nodes + postgres VMs)**: targets for the `plays/lab/*`
  plays — OS updates and internal-CA trust distribution.

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
