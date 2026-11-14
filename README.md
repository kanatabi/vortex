# vortex — GitOps manifests

Part of the [`omicron`](https://github.com/kanatabi/omicron) superproject ·
Frontend: [`earendil`](https://github.com/kanatabi/earendil) ·
Portal: [`mellon`](https://github.com/kanatabi/mellon) ·
Backend: [`lumen`](https://github.com/kanatabi/lumen) ·
Release tooling: [`anvil`](https://github.com/kanatabi/anvil)

GitOps source of truth for the `fzr-blog` K3s cluster. Everything running in the
cluster (except Argo CD itself, which self-hosts from `platform/`) is defined here
and synced by Argo CD via the **app-of-apps** pattern.

**Where the depth lives:**

- **[AGENTS.md](AGENTS.md)** — the full working guide: environments, naming contract,
  bootstrap order, conventions, traps. Read it before changing anything.
- [`omicron/INFRA_ARCHITECTURE.md`](https://github.com/kanatabi/omicron/blob/main/INFRA_ARCHITECTURE.md) —
  architecture rationale and the provisioning guide (the decision record; this repo
  no longer carries its own copy).
- [`omicron/runbook/`](https://github.com/kanatabi/omicron/tree/main/runbook) —
  step-by-step operations (new cluster, seeds, drills).

## What's in the box

```text
platform/argocd/    Argo CD's own install (manual apply, once)
bootstrap/          the four hub entrypoints — applied in order: foundation,
                    observability (manual gate), production, staging
apps/<hub>/         Application CRs ONLY — each hub syncs its folder non-recursively
bases/<app>/        raw manifests: suffix-free, env-free — and ESO-free
components/         cluster-shape toggles (single-node, config-gate)
environments/       the env axis: foundation (shared) · staging · production
```

Three environments share one repo: **foundation** (openbao, external-secrets,
secrets-plane, one Patroni cluster with three logical DBs, one Meilisearch with
per-env indexes, observability, the `lumen-admin` control plane) plus the
**staging** and **production** app trees. One cluster can run any combination of
hub sets; moving an environment between clusters is a hub-CR destination edit —
zero manifest changes.

Public doors: `gofar.id` + `www.gofar.id` (earendil), `console.gofar.id` (mellon),
`anor.gofar.id` (grafana); staging twins at `staging.gofar.id` and
`staging-console.gofar.id`. Never a public API.

## Working in this repo

- Branches forked from `main`, Conventional Commits — never direct commits to
  `main` (the scaffold commits are the baseline exception).
- After **any** manifest edit: `kubectl kustomize <dir>` (no cluster needed;
  `bases/external-secrets` and its chains need `--enable-helm`).
- Deployments don't hand-edit manifests: source repos bump images via
  `anvil`'s release pipeline, and config changes ride the `omicron/config.epoch`
  TRIO (byte-equal, one commit). Details in AGENTS.md §6–§7.

## Status

Production doors are LIVE (cutover 2026-11-11). Staging hosts are contractual —
DNS records and vault seeds land when staging goes live on a cluster. Current
details in AGENTS.md §3.
