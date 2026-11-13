# vortex — GitOps manifests

Part of the [`omicron`](https://github.com/kanatabi/omicron) superproject ·
Frontend: [`earendil`](https://github.com/kanatabi/earendil) ·
Backend contract: [`lumen`](https://github.com/kanatabi/lumen)

GitOps source of truth for the `fzr-blog` K3s cluster. Everything running in
the cluster (except Argo CD itself, which self-hosts from `platform/`) is defined here and
synced by Argo CD via the **app-of-apps** pattern.

Architecture rationale and the full provisioning guide live in
[`omicron/INFRA_ARCHITECTURE.md`](https://github.com/kanatabi/omicron/blob/main/INFRA_ARCHITECTURE.md).
For the step-by-step operations (new cluster, seeds, drills), see
`omicron/runbook/` — the archived minikube/single-node rehearsal history
lives in the workspace `RUNBOOK.md` (the `overlays/{dev,local}` trees and
`components/selfsigned` it walked were retired with feat/environments;
git history keeps them).

## Workflow

- **All changes land on branches forked from `main`** — never commit directly
  to `main` (the initial scaffold commits are the baseline exception). Branch
  names like `feat/<thing>`, `fix/<thing>`, `chore/<thing>`.
- Commit messages follow **Conventional Commits** (`feat|fix|docs|chore(scope): summary`),
  descriptive and granular — one logical change per commit.
- After any manifest edit, validate with `kubectl kustomize <dir>` (no
  cluster needed; `applications/external-secrets` and anything chaining it
  need `--enable-helm` with the helm binary on PATH).

## The three environments

`environments/` is the env axis: **foundation** (shared plumbing, no
suffix, `omicron/env=shared`), **staging** and **production** (per-env app
trees, suffixed and labeled). The trees are ENVIRONMENT files, not cluster
files — one cluster can run foundation+production, foundation+staging, or
all three co-tenanted; moving an environment to another cluster is
repainting the destination in its hub CR (and seeding a vault there), with
**zero manifest edits**.

| tree | nameSuffix | label omicron/env | namespace stance |
| :-- | :-- | :-- | :-- |
| `environments/foundation/*` | — (explicit plain names) | `shared` | owns its platform ns + the co-tenant app ns (lumen-admin) |
| `environments/staging/*` | `-staging` | `staging` | co-tenants the app ns (selectors env-restricted) |
| `environments/production/*` | `-production` | `production` | co-tenants the app ns (selectors env-restricted) |

**Shared-by-design services** (all in the foundation tree):

- `lumen-db` — ONE Patroni cluster (ns `db`) declaring THREE logical
  databases + owner users, selected per consumer by the `LUMEN_DB_URL`
  behind each environment's Bao path:

  | database | owner user | consumer | vault path |
  | :-- | :-- | :-- | :-- |
  | `lumen_admin` | `db_user_admin` | foundation `lumen-admin` (Control Plane) | `kv/lumen/shared` |
  | `lumen_staging` | `db_user_staging` | `environments/staging/lumen` | `kv/lumen/staging` |
  | `lumen_production` | `db_user_production` | `environments/production/lumen` | `kv/lumen/production` |

  The Zalando operator generates credentials into its dotted-naming
  Secrets (`lumen-pg.db_user_*`).
- `meilisearch` — ONE shared instance (ns `search`); environments coexist as
  **per-env index names** (`LUMEN_MEILI_INDEX_*` — values behind each cms
  overlay's Bao path, keys via ExternalSecret data entries).
- `openbao`, `external-secrets`, `secrets-plane`, `observability` — one
  instance each; the Grafana door (`anor.gofar.id`) is production-owned DNS
  hosted by the foundation plane.

**lumen-admin** (foundation; overlay dir `environments/foundation/lumen` —
dir name == app name so the bump tooling needs no name map, while every
resource keeps its explicit `lumen-admin-*` name): the estate's single
Control Plane — Deployment `lumen-admin-http-server` (role=admin; no worker
fleet by owner ruling), Service `lumen-admin` (unsuffixed — the two cms
trees reach it by SEEDED address, not hardcoded manifest: `kv/lumen/shared`
property `LUMEN_ADMIN_URL` rendered through their ES, the naming contract
guaranteeing the Service name), no ingress
ever (§2: reached only via mellon's relay in-cluster), fed from
`kv/lumen/shared`.

**App identities for vault paths:** `kv/lumen/shared` (admin),
`kv/lumen/staging|production` (cms), `kv/{app}/{env}` for earendil/mellon
as before, `kv/grafana/production` + `kv/cert-manager/production` stay
foundation-side singletons.

**ES = the render contract.** Every ExternalSecret `data:` list mirrors
what the deployed role actually binds — for lumen that truth is machine-
readable (`lumen config keys --role admin|cms`, lumen ec16db5): the admin
ES carries only `LUMEN_DB_URL` + `LUMEN_SESSION_JWT_SECRET` (the signing
key has exactly ONE home), the cms ESes carry the content keys plus
`LUMEN_ADMIN_URL` (source `lumen/shared` — a service URL is a VALUE;
the `LUMEN_MEILI_URL` precedent) and never the JWT secret. `LUMEN_ROLE`
stays inline on the pods: deployment IDENTITY, not configuration — vault
must never be able to silently re-role a fleet. **Seeding order**: ESO
fails the whole render on a missing property, so every key a contract
names must exist in Bao BEFORE its hub syncs (the runbook orders this).

### Naming contract (the kustomize trap, handled per overlay)

`nameSuffix` rewrites `metadata.name` **only**. Every name-bearing spec
reference is therefore made explicit — either AUTHORED final in files the
overlay owns (ES `spec.target.name` = `lumen-secrets-staging`,
`app-config-production`, …) or JSON6902-patched (base-file references:
deployments' `envFrom` secretRef/configMapRef, migrate Job envFrom, Ingress
backends stated final). The `bao` store, issuerRef and the
`lumen-bao-credentials` bootstrap family stay literal everywhere — they are
single-instance or Git-unmanaged. **Vault values must name the suffixed
Services** (seeding contract): production earendil/mellon `API_BASE` →
`http://lumen-http-server-production.lumen.svc.cluster.local`; staging →
the `-staging` twin. The overlays carry these strings in comments for the
runbook to mirror.

### Hosts

| environment | host | served by | TLS cert (applications/certificates) |
| :-- | :-- | :-- | :-- |
| production | `gofar.id` + `www.gofar.id` | earendil (surface #1) | `gofar-id` → `tls-gofar` |
| production | `console.gofar.id` | mellon (surface #2) | `console-gofar-id` → `tls-console` |
| production | `anor.gofar.id` | grafana (dashboard door) | `anor-gofar-id` → `tls-anor` |
| production | `img.gofar.id` | R2 bucket (custom domain — GATE #9, runbook/06; no manifest) | bucket-side, outside ACME |
| staging | `staging.gofar.id` | earendil-staging | `staging-gofar-id` → `tls-gofar-staging` |
| staging | `staging-console.gofar.id` | mellon-staging | `staging-console-gofar-id` → `tls-console-staging` |

www is production-only (owner ruling); staging never wears it. DNS-01 works
on any cluster, so staging rides the same ClusterIssuer — the self-signed
component died with the rehearsal trees.

## Layout

```text
.
├── bootstrap/                     # manual-applied hub entrypoints
│   ├── apps-hub-foundation.yaml   # #1: syncs apps/foundation/ (values+data plane)
│   ├── apps-hub-production.yaml   # #2: syncs apps/production/ (application plane)
│   ├── apps-hub-staging.yaml      # #3: syncs apps/staging/ (staging plane, same
│   │                              #     or another cluster — destination-only)
│   └── apps-hub.yaml              # 3-node BLUEPRINT hub (future scale-out shape)
├── apps/                      # Application CRs ONLY (one per deployed app) —
│   ├── earendil-app.yaml # SvelteKit SSR site (3-node blueprint: the prod bases)
│   ├── lumen-app.yaml  # Go API (cluster-internal, no ingress)
│   ├── lumen-db-app.yaml       # Patroni PostgreSQL (selfHeal: false — DB guard)
│   ├── observability-app.yaml # Loki/VictoriaMetrics/Alloy/Grafana (Phase 5)
│   ├── meilisearch-app.yaml   # search backend       (cluster-internal)
│   ├── openbao-app.yaml       # the values store itself (§5)
│   ├── mellon-app.yaml   # admin portal           (console.gofar.id — surface #2)
│   ├── external-secrets-app.yaml # ESO, as an OCI Helm chart — the one non-git Application
│   ├── secrets-plane-app.yaml # ClusterSecretStore `bao` (wave 1: after ESO+OpenBao)
│   ├── foundation/            # FOUNDATION hub CRs (openbao 0, external-secrets 0,
│   │                          # secrets-plane 1, lumen-db 1, meilisearch 1,
│   │                          # observability 2, lumen-admin 2)
│   ├── production/            # PRODUCTION hub CRs (lumen 0, earendil 1, mellon 1,
│   │                          # certificates 1) → environments/production/*
│   └── staging/               # STAGING hub CRs (lumen-staging 0, earendil-staging 1,
│                              # mellon-staging 1) → environments/staging/*
├── applications/              # raw manifests (BASES), one kustomize dir per app —
│   │                          # suffix-free, env-free, cluster-shape-free
│   ├── earendil/  mellon/  lumen/  lumen-db/  observability/  meilisearch/
│   ├── openbao/   secrets-plane/  certificates/  external-secrets/
├── components/                # placement/cluster-SHAPE toggles, orthogonal to env trees
│   ├── single-node/           # collapses pools, replica pairs, Patroni instances to 1 node
│   └── config-gate/           # PreSync epoch gate (Job + RBAC, ns-agnostic)
├── environments/              # THE ENV AXIS (nameSuffix + omicron/env per the contract)
│   ├── foundation/            # shared plumbing, no suffix: openbao, external-secrets,
│   │   │                      # secrets-plane, lumen-db, meilisearch, observability,
│   │   └── lumen-admin/       # the Control Plane (+ the co-tenant app namespaces)
│   ├── staging/               # lumen, earendil, mellon — suffix -staging
│   └── production/            # lumen, earendil, mellon — suffix -production
└── platform/
    └── argocd/                # Argo CD's own install: pinned upstream + overlays
```

## Entrypoints — bootstrap order (new cluster)

Three hubs, in this order — the ordering IS the seed-before-apps invariant
(the single-VPS estate runs foundation+production; add staging wherever the
staging tree should land — same cluster or another):

```bash
# 0. Argo CD itself (manual, once): kubectl apply -k platform/argocd/
kubectl apply -f bootstrap/apps-hub-foundation.yaml   # #1: values+data plane
# wait for foundation waves (openbao + external-secrets = 0, secrets-plane +
# lumen-db + meilisearch = 1), then run runbook M3: Bao init/unseal/AppRole
# and the kv seeds — kv/{app}/{env} for EVERY environment you plan to run
# (production, staging) plus kv/lumen/shared (admin) and the
# cert-manager/grafana feeds. The `bao` ClusterSecretStore stays NOT-Ready
# until M3 lands; foundation wave 2 (observability, lumen-admin) waits for
# it structurally — do not proceed before it is Ready.
kubectl apply -f bootstrap/apps-hub-production.yaml   # #2: production plane
kubectl apply -f bootstrap/apps-hub-staging.yaml      # #3 (optional): staging
```

Sync waves restart per hub. Foundation = waves 0/0/1/1/1/2/2 above;
production = lumen 0 (dependency root), earendil + mellon + certificates 1;
staging = lumen-staging 0, earendil-staging + mellon-staging 1. Exactly one
hub SET (foundation + at least one app hub) may own a cluster; every hub
syncs its own folder **non-recursively**, so the trees never see each
other's CRs. Adding an app = a CR in the folder its hub syncs + an
`applications/<name>` base (+ env overlays as the app needs envs) — never a
hub edit.

`bootstrap/apps-hub.yaml` (flat `apps/`) is the 3-node BLUEPRINT entrypoint
for the future scale-out: it syncs the raw bases (no ESO render chain). The
old single-node rehearsal hubs (`apps-hub-single-node.yaml`,
`apps/single-node/`) are retired — their walk history is preserved in the
archived workspace `RUNBOOK.md` and in git log.

## How sync works

```
git push ──► hub (bootstrap/apps-hub-*) ──► per-app Application ──► environments|applications/<path> ──► cluster
```

CI never touches the cluster. Each source repo builds an image to GHCR, then
runs `kustomize edit set image <name>=<ghcr image>:<tag>` in its app dir —
the bases AND the lockstep `lumen` gate stanzas across
`environments/*/*/kustomization.yaml` move in ONE commit (the config-gate
must never decode epochs with an image older than the app's). Argo sees the
commit and syncs (`prune + selfHeal` by default).

## Conventions & rules

- **The naming contract is load-bearing** (sibling repos, seeding docs and
  dashboards quote its exact strings): staging `-staging`/`omicron/env=
  staging`, production `-production`/`omicron/env=production`, foundation
  no-suffix/`omicron/env=shared`; env-restricted selectors via
  `includeSelectors: true` make namespace co-tenancy safe.
- **Epoch TRIO everywhere config-gate runs**: the pod-template annotation
  `omicron/config.epoch`, the gate's `--declared` arg, AND the
  ExternalSecret annotation must stay byte-equal — the bump flow edits all
  three lines in one commit, CI asserts the pair. Values today:
  production lumen 7, earendil/mellon 0; staging all 0; shared (lumen-admin) 0.
- **Placement:** `deployment.yaml` files schedule via `node-pool` (labels:
  `platform` on the master, `data`/`edge` on workers) and hostname
  anti-affinity for 2-replica apps; `components/single-node` collapses that
  for the one-VPS reality. Budget tables in the architecture doc §1 must
  stay balanced if you change requests.
- **Two sanctioned public surfaces + the dashboard door, never a public
  API.** The Go API, Meilisearch and lumen-admin are cluster-internal
  (service DNS only). Browser→API traffic rides the SSR/portal same-origin
  proxies — see architecture doc §2.
- **Values live in OpenBao, never in Git.** This repo holds
  schema/templates only: `ExternalSecret` CRs point at `kv/{app}/{env}` (and
  `kv/lumen/shared` for the admin identity; per-env index names are VALUES,
  e.g. `LUMEN_MEILI_INDEX_POSTS`); ESO renders them in-cluster at sync time.
  DB credentials are generated by the Zalando operator (dotted-naming
  Secrets); the BAO relay trio is the §5.6 k8s-manual bootstrap family.
- **The secrets plane is foundation territory.** Bases (`applications/*`)
  stay ESO-free; per-env values ride `environments/{staging,production}/<app>`
  pointing at OpenBao via the `bao` ClusterSecretStore. (Observability is
  the one declared exception in the bases: store consumers only, never
  blocking.)
- **Image tags:** released semver `vX.Y.Z` / immutable `sha-<short>` only —
  no `latest`, no branch SHAs in manifests.
- **Versions pinned everywhere:** Argo overlay, cert-manager/operator URLs,
  ESO chart: exact versions. Bump deliberately, one upgrade PR at a time.
- **Namespaces:** platform services declare their own (`secrets`, `db`,
  `search`, `observability`, `external-secrets`); the APP namespaces
  (lumen/earendil/mellon) are co-tenant objects rendered ONCE by the
  foundation lumen-admin tree — env trees `$patch: delete` their copies so
  no two hubs ever fight over one Namespace.
- **DB app exception:** `apps/lumen-db-app.yaml` (blueprint, 2-instance
  shape) disables `selfHeal` so incident-time failovers aren't stomped by
  Git; the foundation overlay (one instance on the VPS) keeps selfHeal on —
  there is no hand-executed failover to stomp at that shape.

## Adding a new application

1. Create `applications/<name>/` — copy `applications/lumen/` as the template
   (`kustomization.yaml` with an `images:` stanza, `namespace.yaml` [if the
   ns is single-owner], workloads, ingress).
2. Create `environments/<env>/<name>/` overlays per env you want (copy the
   staging/production twin — remember the spec-field patch list: envFrom,
   ingress backends, ES target names) and an `apps/<env>/<name>-app.yaml`
   CR with the right suffix/wave/`destination.namespace`.
3. Wire the source repo's deploy workflow to point at the base AND the env
   overlay stanzas it must keep in lockstep.

## Placeholders / status

`gofar.id` production doors are LIVE (cutover 2026-11-11). Staging hosts are
contractual (`staging.gofar.id`, `staging-console.gofar.id`) — DNS records +
vault seeds (`kv/{app}/staging`, `kv/lumen/shared`, suffixed `API_BASE`
values!) land with the runbook when staging goes live on a cluster. `ORIGIN`
is dead (no consumer); re-add only WITH its reader.
