# AGENTS.md

## What this repo is

A Zarf/UDS package (not application code). It bundles two UDS Core prerequisite
services — MetalLB and cert-manager — as Helm charts + manifests, published as an OCI
artifact. It ships the **services** (required components) **plus opinionated default
config** as **optional, on-by-default** components (`required: false`, `default: true`):
MetalLB `IPAddressPool`/`L2Advertisement` and cert-manager `ClusterIssuer`s. Consumers
can deselect a config component to bring their own. (This config previously lived in a
separate `uds-prereq-config` package, now superseded.)

## Layout

- `zarf.yaml` — the package definition (source of truth for components, chart versions, images, variables).
- `tasks.yaml` — `uds run` task runner (build/publish/deploy/lint).
- `metallb/` and `cert-manager/` — Helm `values.yaml`, service `manifests/`, and config
  manifests (`metallb-config.yaml`, `clusterissuers.yaml`) per component.
- `.releaserc.json` + `.github/workflows/release.yml` — semantic-release publishing.

## Commands

Requires `uds` (uds-cli) and `zarf` on PATH.

```bash
uds run lint                    # zarf dev lint (do this before committing)
uds run build --set VERSION=0.1.0
uds run publish --set VERSION=0.1.0
uds run deploy --set VERSION=0.1.0   # deploys from OCI, not a local tarball
```

Task vars (`tasks.yaml`): `VERSION` (default `0.0.0-dev`), `REGISTRY`
(`ghcr.io/sam-delap`), `ARCH` (`amd64`). `VERSION` has no default in `zarf.yaml`, so it
must be supplied via `--set VERSION=` on any direct `zarf` invocation.

## Versioning / release (important gotchas)

- **Do not hand-edit `metadata.version` in `zarf.yaml`.** It is the literal template
  string `###ZARF_PKG_TMPL_VERSION###`, filled at build time from `--set VERSION=`.
- Versions come from **Conventional Commits** via semantic-release on push to `main`.
  Commit messages drive the release (`feat:`, `fix:`, etc.) — use them deliberately.
- Publishing happens only in CI (`.github/workflows/release.yml`) via
  `@semantic-release/exec`, which runs `zarf package create`+`publish`. Don't publish
  manually from a branch.

## When editing components

- Chart versions and pinned images both live in `zarf.yaml`. If you bump a chart
  `version`, update the matching image tags in the same component's `images:` list
  (Zarf mirrors those images — they aren't auto-derived from the chart).
- Each component has an `onDeploy.after` `wait` on a specific deployment
  (`metallb-controller`, `cert-manager-webhook`). Keep these in sync if you rename or
  restructure.
- `metallb/manifests/namespace.yaml` sets `privileged` Pod Security Standards — required
  for speaker pods. cert-manager installs CRDs (`crds.enabled: true`) so the config
  components' APIs (`IPAddressPool`, `ClusterIssuer`) exist. Don't remove either.

## Config components (opt-out model)

- `metallb-config` and `cert-manager-config` are `required: false` + `default: true`.
  Under `zarf package deploy --confirm` they deploy automatically. Consumers opt out via
  `--components=-metallb-config`. **UDS bundle caveat:** optional components deploy only
  if listed under the package's `optionalComponents` — omitting them there is the opt-out.
- These components are ordered **after** their services and gate on an `onDeploy.before`
  wait for their CRD (`ipaddresspools.metallb.io`, `clusterissuers.cert-manager.io`) to be
  `established`. Keep that wait if you add config that depends on a new CRD.
- Config values are Zarf variables with the author's home lab defaults:
  `METALLB_IP_RANGE`, `ACME_EMAIL`, `DNS_ZONE` (declared in `zarf.yaml`, templated as
  `###ZARF_VAR_*###` in the manifests). If you rename a variable, update both places.
- The cert-manager ClusterIssuer DNS-01 solver is **hardcoded to Cloudflare** by design;
  only email/zone are variables. Non-Cloudflare users are expected to opt out and BYO —
  don't try to parameterize the solver.

See `README.md` for the full UDS layering model, override flags, and consumption example.
