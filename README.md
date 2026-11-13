# vortex — GitOps manifests

Part of the [`omicron`](https://github.com/kanatabi/omicron) superproject ·
Frontend: [`earendil`](https://github.com/kanatabi/earendil) ·
Backend contract: [`lumen`](https://github.com/kanatabi/lumen)

GitOps source of truth for the `fzr-blog` K3s cluster. Everything running in the
cluster (except Argo CD itself, which self-hosts from `platform/`) is defined here and
synced by Argo CD via the **app-of-apps** pattern.

Architecture rationale and the full provisioning guide live in
[`omicron/INFRA_ARCHITECTURE.md`](https://github.com/kanatabi/omicron/blob/main/INFRA_ARCHITECTURE.md).
For running the whole stack locally with k3d, see
[`omicron/README.md`](https://github.com/kanatabi/omicron/blob/main/README.md).

## Workflow

- **All changes land on branches forked from `master`** — never commit directly to
  `master` (the initial scaffold commits are the baseline exception). Branch names like
  `feat/<thing>`, `fix/<thing>`, `chore/<thing>`.
- Commit messages follow **Conventional Commits** (`feat|fix|docs|chore(scope): summary`),
  descriptive and granular — one logical change per commit.
- After any manifest edit, validate with `kubectl kustomize <dir>` (no cluster needed).

## Layout

```text
.
├── bootstrap/
│   ├── apps-hub.yaml               # root Application (3-node shape) — apply manually
│   ├── apps-hub-single-node.yaml   # the 1-node variant (local rehearsal + 1-VPS prod)
│   ├── apps-hub-foundation.yaml    # VPS prod entrypoint #1 of 2 (manual) — syncs apps/foundation/
│   └── apps-hub-apps.yaml          # VPS prod entrypoint #2 of 2 (manual) — syncs apps/applications/
├── apps/                      # Application CRs ONLY (one per deployed app) —
│   │                          # apps/ itself = the 3-node hub's flat set; every hub syncs
│   │                          # its own folder NON-recursively, so the shapes never see
│   │                          # each other's CRs — exactly one hub (or hub PAIR) owns a cluster
│   ├── earendil-app.yaml # SvelteKit SSR site   (source repo: earendil)
│   ├── lumen-app.yaml  # Go API               (source repo: lumen — cluster-internal, no ingress)
│   ├── lumen-db-app.yaml       # Patroni PostgreSQL   (selfHeal: false — DB guard)
│   ├── observability-app.yaml # Loki/VictoriaMetrics/Alloy/Grafana (Phase 5)
│   ├── meilisearch-app.yaml   # search backend       (cluster-internal)
│   ├── openbao-app.yaml       # the values store itself (§5)
│   ├── mellon-app.yaml   # admin portal           (console.gofar.id — surface #2)
│   ├── external-secrets-app.yaml # ESO, as an OCI Helm chart — the one non-git Application
│   ├── secrets-plane-app.yaml # ClusterSecretStore `bao` (wave 1: after ESO+OpenBao)
│   ├── single-node/           # twin CR set for 1-node clusters (hub-single syncs this)
│   ├── foundation/            # VPS prod CRs, FOUNDATION hub (openbao, external-secrets,
│   │                          # secrets-plane, lumen-db, meilisearch — waves 0/0/1/1/1)
│   └── applications/          # VPS prod CRs, APPS hub (lumen wave 0, earendil/mellon/
│                              # certificates wave 1, observability wave 2)
├── applications/              # raw manifests, one kustomize dir per app
│   ├── earendil/
│   ├── mellon/
│   ├── lumen/
│   ├── lumen-db/
│   ├── observability/
│   ├── meilisearch/
│   ├── openbao/
│   └── secrets-plane/
├── components/                # placement/cluster-SHAPE toggles, orthogonal to env overlays
│   ├── single-node/           # collapses pools, replica pairs, Patroni instances to 1 node
│   └── selfsigned/            # strips ACME TLS for clusters without cert-manager
├── overlays/
│   ├── dev/                   # ESO feeds reading kv/{app}/dev        (local cluster)
│   ├── staging/               # kv/{app}/staging
│   ├── production/            # same bases + values sourced from OpenBao via ESO (§5.3)
│   │   ├── earendil/          # JSON6902 patch: inline env -> ESO-rendered app-config
│   │   ├── lumen/             # ExternalSecret materializes lumen-secrets
│   │   └── mellon/            # portal config feed (kv/mellon/production)
│   └── local/                 # CHAINS: dev env axis + single-node + selfsigned —
│                              # the minikube/1-VPS rehearsal views, still real GitOps
└── platform/
    └── argocd/                # Argo CD's own install: pinned upstream + overlays
        ├── namespace.yaml     # upstream install.yaml omits it — the overlay owns it now
        ├── pinning-patch.yaml
        └── resources-patches.yaml
```

## How sync works

```
git push ──► Argo hub (apps/) ──► per-app Application ──► applications/<name>/ ──► cluster
```

CI never touches the cluster. Each source repo builds an image to GHCR, then runs
`kustomize edit set image <name>=<ghcr image>:sha-<short>` in its app dir under
`applications/` and pushes. Argo sees the commit and syncs (`prune + selfHeal` by
default).

## Adding a new application

1. Create `applications/<name>/` — copy `applications/lumen/` as the template
   (`kustomization.yaml` with an `images:` stanza, `namespace.yaml`, workloads, ingress).
2. Create `apps/<name>-app.yaml` — copy an existing one; set `path`, `destination.namespace`,
   name. That's it — the hub picks it up automatically (no hub edits).
3. Wire the source repo's deploy workflow to point at `applications/<name>/`.

## Conventions & rules

- **Placement:** `deployment.yaml` files schedule via `node-pool` (labels: `platform`
  on the master, `data`/`edge` on workers) and hostname anti-affinity for 2-replica apps.
  Budget tables in the architecture doc §1 must stay balanced if you change requests.
- **Only one public ingress: the frontend.** The Go API and Meilisearch are
  cluster-internal (ClusterIP service DNS only). The browser's single client-side API
  call rides the SSR same-origin proxy (`/api/...` on `sv-fzr`), so adding a public
  API route is a security decision, not a convenience one — see architecture doc §2.
- **Values live in OpenBao, never in Git.** This repo holds schema/templates only:
  `ExternalSecret` CRs point at `kv/{env}/{app}` paths (version-pinned per overlay);
  ESO renders them into in-cluster ConfigMaps/Secrets at sync time. No plaintext `Secret`
  manifests, and no SOPS blobs for regular config either — see architecture doc §5.
  The only in-Git encryption exception: offline DR seeds in the separate `fzr-dr` store.
  DB credentials are generated by the Zalando operator; the Go app reads them from
  the operator-managed Secret via `envFrom`.
- **Secrets plane is production-overlay territory.** Bases (`applications/*`) stay
  ESO-free so k3d works; per-env values come from `overlays/production/<app>` pointing
  at OpenBao via the `bao` ClusterSecretStore. Going prod for an app = repoint its
  Application CR `source.path` to the overlay — see architecture doc §5.3.
- **Image tags:** immutable `sha-<short>` only. No `latest` in any manifest.
- **Versions pinned everywhere:** Argo overlay and cert-manager/operator URLs carry exact
  versions. Bump deliberately, one upgrade PR at a time.
- **Namespaces:** one per app, declared in the app's own `namespace.yaml`
  (`earendil`, `lumen`, `db`, `search`, `ingress`, …).
- **DB app exception:** `lumen-db-app.yaml` disables `selfHeal` so incident-time
  failovers aren't stomped by Git. Keep it that way.

## First bootstrap (new cluster)

```bash
# 1. Argo CD itself (manual, once)
kubectl apply -k platform/argocd/
# 2. one-time namespace for the ESO chart app (charts don't self-create it)
kubectl create namespace external-secrets
# 3. App-of-apps hub — repoURLs are real (kanatabi/vortex), apply directly
#    (single node? apply bootstrap/apps-hub-single-node.yaml INSTEAD — never both;
#     production VPS? skip both of these — see "Production VPS entrypoint" below)
kubectl apply -f bootstrap/apps-hub.yaml
```

## Production VPS entrypoint — TWO hubs (feat/hub-split)

The live single-VPS estate is bootstrapped by a hub PAIR, both manual applies
(the documented chicken-and-egg exception, same class as every other hub:
GitOps cannot bootstrap the cluster hosting its own root app). Two applies is
the entire manual surface, and their ORDER is the seed-before-apps invariant
made structural:

```bash
# 0. Argo CD itself: kubectl apply -k platform/argocd/
kubectl apply -f bootstrap/apps-hub-foundation.yaml   # #1: values+data plane
# wait for foundation waves (openbao + external-secrets = 0, secrets-plane +
# lumen-db + meilisearch = 1), then run runbook M3 — Bao init/unseal/AppRole
# and the kv/{app}/production seeds. The `bao` ClusterSecretStore stays
# NOT-Ready until M3 lands; do not proceed before it is Ready.
kubectl apply -f bootstrap/apps-hub-apps.yaml         # #2: application plane
```

Sync waves restart per hub (they are only comparable inside one Application
tree): foundation = openbao 0, external-secrets 0, secrets-plane 1, lumen-db 1,
meilisearch 1; applications = lumen 0 (dependency root), earendil 1, mellon 1
(both read lumen), certificates 1, observability 2. The CRs live in
`apps/foundation/` and `apps/applications/` — both synced NON-recursively, so
the 3-node hub (`apps/`) and the single-node hub (`apps/single-node/`) never
see them, and vice versa: exactly one hub (or pair) may own a cluster. The old
combined `bootstrap/apps-hub-production.yaml` + `apps/production/` are retired.
Adding a prod app = a CR in the folder its hub syncs; changing which plane an
app belongs to = `git mv` between the two folders.


## Single-node clusters (local minikube / one VPS)

The blueprint targets 3 nodes (`node-pool` selectors, 2-replica anti-affinity pairs,
2-instance Patroni). One node satisfies none of that — so the collapse lives in
`components/`, and `overlays/local/*` chains the dev env axis through it.
`bootstrap/apps-hub-single-node.yaml` + `apps/single-node/` give the whole estate a
1-node home. Sync waves make the runbook's hard ordering STRUCTURAL: OpenBao (wave 0)
comes up unsealed-but-empty, the `bao` store (wave 1) stays NOT-Ready until the manual
init/AppRole/seed lands, and the apps (wave 2) simply cannot render env before that —
no "pods booted with defaults" failure mode. See the root workspace `RUNBOOK.md` for
the full step-by-step (local rehearsal AND VPS deltas).

Placeholders to replace before going live: `ainurrofiq.com` + `staging.ainurrofiq.com`
(ingress hosts — decide the gofar.id cutover alongside the owner; `ORIGIN` env is gone:
earendil reads no canonical-origin variable), and the `REPLACE_AT_BOOTSTRAP` roleId in
`applications/secrets-plane/`. (`repoURL`s were realigned to `kanatabi/vortex` 2026-11-10
when the manifests repo moved org+name — they are no longer a placeholder.)
