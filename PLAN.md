# Context: rke2-gitops repo

This is the GitOps repo for Ryan's homelab RKE2 cluster. Argo CD (running on
the cluster) syncs everything in this repo. A separate repo, `rke2`
(https://github.com/RyanMillerC/rke2 once pushed), is the Ansible/Hyper-V
bootstrap codebase that provisions the VMs, OS, RKE2, kube-vip, Cilium, and
Argo CD itself — that repo's job ends once Argo CD exists; from there, this
repo is the only thing that should change the cluster's platform/app state.
Full design doc lives in that repo's `PLAN.md` — ask Ryan for it if you need
the complete picture; the essentials are below.

## Current state (as of 2026-11-23)

- This repo is **public, empty, default branch `main`**.
- The `rke2` repo already has an `argocd` Ansible role that writes 3
  manifests into RKE2's server-manifests dir: a `HelmChart` CR (installs
  Argo CD 10.18.3 into namespace `argocd`), two `AppProject`s (`platform`,
  `apps`), and one root `Application` (see exact spec below). This is
  committed in the `rke2` repo but **`make configure` has not been run
  yet** — Argo CD is not actually installed on the live cluster as of this
  writing. Don't assume it's running; ask Ryan or have him confirm before
  assuming any live-cluster state.
- The 3-node RKE2 cluster (`pico-1/2/3.taco.moe`) is up and `Ready`, with
  kube-vip (API VIP `192.168.0.98` / `api.pico.taco.moe`) and Cilium
  already running — both installed by Ansible, **not** by Argo, and Argo
  must never manage either.

## The root Application you need to satisfy

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
spec:
  project: platform
  source:
    repoURL: "https://github.com/RyanMillerC/rke2-gitops.git"
    targetRevision: HEAD
    path: "bootstrap"
    directory:
      recurse: true
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [CreateNamespace=true]
```

Two `AppProject`s already exist (created by Ansible, don't recreate them
here): `platform` (cluster components) and `apps` (workloads). Both have
`sourceRepos: [this repo]`, `destinations: namespace "*"`, wide-open
`clusterResourceWhitelist`.

## Required layout (app-of-apps)

`bootstrap/` is the root Application's sync path with `directory.recurse:
true` — **only put child `Application` manifests there**, never raw
platform/app manifests, or Argo will try to apply them directly instead of
through their own child Application. Real manifests live in sibling
directories that the bootstrap Applications point to:

```
bootstrap/                  # one Application per PLAN.md sync wave below
  01-cilium-lb.yaml          # project: platform
  02-storage.yaml
  03-cert-manager.yaml
  04-vault.yaml
  05-eso.yaml
  06-traefik.yaml
  07-apps.yaml               # project: apps
platform/<component>/       # actual manifests/kustomize/helm values
apps/<app>/
```

Each bootstrap `Application` needs an
`argocd.argoproj.io/sync-wave: "<N>"` annotation matching its number below,
and its own `automated: {prune: true, selfHeal: true}` + `CreateNamespace=true`.

## Sync waves (PLAN.md Build Sequence step 7, lowest first)

1. Cilium LB IP pool + L2 announcement policy
2. Storage drivers and StorageClasses
3. cert-manager + issuers
4. Vault
5. External Secrets Operator + ClusterSecretStore
6. Traefik configuration
7. Apps

## Key architecture decisions this content must follow

- **Cilium**: Argo only owns `CiliumLoadBalancerIPPool` (single IP
  `192.168.0.99/32`) and `CiliumL2AnnouncementPolicy` — Cilium's own
  install/config is Ansible's job, not Argo's. Pin the shared apps IP
  `192.168.0.99` via Cilium's LB-IPAM sharing-key annotation so Traefik and
  any non-HTTP LoadBalancer Services can share it.
- **Ingress**: RKE2's bundled Traefik, configured via `HelmChartConfig`,
  exposed as a LoadBalancer Service on `192.168.0.99`, host ports disabled.
- **cert-manager**: DNS-01 solver, wildcard `*.apps.taco.moe`. Ryan has an
  existing working config from the old K3s cluster to reuse — ask him for
  it rather than designing from scratch. Issuer differs per environment:
  Let's Encrypt **staging** for `pico` (sandbox, current), **production**
  for `guac` (lab hardware, not yet built). Verify `ingressClassName`
  matches whatever class RKE2's Traefik registers.
- **Storage**: `csi-driver-nfs` as the default StorageClass, reclaim policy
  `Retain`, subdirectories named by namespace+PVC name (so rebuilds
  reattach to existing data). Synology CSI (iSCSI) StorageClass for
  databases, Prometheus, and Vault's Raft storage — **never put databases
  or Prometheus on NFS**.
- **Vault + ESO**: Vault in-cluster, integrated Raft storage on iSCSI,
  **AWS KMS auto-unseal**. ESO authenticates via Vault's Kubernetes auth
  method (in-cluster, no static credential). Sync order Vault → ESO → apps
  (waves 4→5→7 already encode this).
- **Bootstrap secrets** (AWS creds for KMS unseal, DNS provider API token
  for cert-manager, Synology CSI credentials, Vault seed secrets) live in
  the `rke2` repo's Ansible Vault — **never put real credentials in this
  repo**, even encrypted. How those bootstrap secrets actually get from
  Ansible Vault into the cluster as k8s Secrets (before ESO exists to do
  it) is an **open question nobody has solved yet** — don't invent a
  mechanism silently; flag it to Ryan.
- Network: domain `taco.moe`, apps wildcard `*.apps.taco.moe` →
  `192.168.0.99`, API `api.pico.taco.moe` → `192.168.0.98` (not this
  repo's concern, just for context).

## Acceptance tests this repo's content is ultimately judged against

- Wildcard cert present and served by Traefik on `*.apps.taco.moe`
- A non-HTTP LoadBalancer Service gets `192.168.0.99`, reachable from the
  admin network
- Vault auto-unseals and ESO refreshes secrets after a node reboot
- After a full cluster rebuild, an NFS-backed app reattaches to its old PVC
  data
- Full rebuild (destroy all 3 nodes, rerun Ansible) converges with zero
  manual steps, including Vault seeding

## Out of scope for now (Day 2, PLAN.md)

Cilium host firewall, kube-prometheus-stack, Hubble metrics, Loki, RKE2
upgrade automation, SSO, VLAN segmentation.

## Working with Ryan

He's experienced across this whole stack (Ansible, Kubernetes/CNI
internals, Vault/ESO/cert-manager patterns) — not a beginner project, don't
over-explain basics. He prefers a clear recommendation with one concrete
tradeoff over a neutral menu of options, and prefers moving fast /
iterating in code over long upfront planning. He generally commits his own
work between sessions, so re-check `git log`/`git status` rather than
assuming prior state. Never use em-dashes in any file or output.
