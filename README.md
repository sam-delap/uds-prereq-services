# uds-prereq-services

UDS Core **prerequisite services**, packaged as a versioned Zarf/OCI artifact for
home lab (and other bare-metal) clusters.

This package installs the _services only_ — no environment-specific configuration.
Downstream consumers inherit these services and supply their own configuration via a
separate package (see [`uds-prereq-config`](https://github.com/sam-delap/uds-prereq-config)).

## What's in this package

| Component            | Contents                                                             |
| -------------------- | -------------------------------------------------------------------- |
| `metallb-namespace`  | `metallb-system` namespace with privileged Pod Security Standards    |
| `metallb`            | MetalLB controller + speakers (chart `0.14.9`, FRR enabled)          |
| `cert-manager`       | cert-manager controller, webhook, cainjector, and CRDs (chart `v1.21.0`) |

Explicitly **not** included (these belong in the config package):

- MetalLB `IPAddressPool` / `L2Advertisement`
- cert-manager `ClusterIssuer` / `Certificate` resources

## Where this fits in the UDS layering model

```
zarf init (stock)
  → uds-prereq-services   ← THIS package (MetalLB + cert-manager services)
  → uds-prereq-config     ← IPAddressPool, L2Advertisement, ClusterIssuers
  → UDS Core              ← separate repo/bundle
  → apps
```

MetalLB is a UDS Core prerequisite: it must exist before UDS Core Base so Istio's
ingress gateways (type `LoadBalancer`) can be assigned an address. See the UDS Core
[prerequisites](https://uds.defenseunicorns.com/reference/uds-core/prerequisites/) and
[functional layers](https://uds.defenseunicorns.com/reference/uds-core/functional-layers/) docs.

## Building & publishing

Versions are managed by [semantic-release](https://semantic-release.gitbook.io/) from
Conventional Commits. On push to `main`, CI computes the next version, injects it into
`metadata.version` via `--set VERSION=`, and publishes to
`oci://ghcr.io/sam-delap/uds-prereq-services`.

Local build (uses `uds run`):

```bash
uds run build --set VERSION=0.1.0
uds run publish --set VERSION=0.1.0
uds run lint
```

Or directly with Zarf:

```bash
zarf package create . --set VERSION=0.1.0 --confirm
zarf package publish zarf-package-uds-prereq-services-amd64-0.1.0.tar.zst oci://ghcr.io/sam-delap --confirm
```

## Consuming this package

Compose it with a config package in a UDS bundle. Deploy order matters — the base
package installs the CRDs (`metallb.io`, `cert-manager.io`) that the config package's
resources depend on:

```yaml
kind: UDSBundle
metadata:
  name: homelab-prerequisites
  version: "0.1.0"

packages:
  - name: uds-prereq-services
    repository: ghcr.io/sam-delap/uds-prereq-services
    ref: 0.1.0
  - name: uds-prereq-config
    repository: ghcr.io/sam-delap/uds-prereq-config
    ref: 0.1.0
```

## Notes

- **Domain:** the wider home lab standardizes on `uds.sams-club-it.com`. The wildcard
  `*.uds.sams-club-it.com` `Certificate` itself is **not** created here — it lives in the
  UDS Core repo where Istio consumes it.
- **Secrets (Phase 2):** the cert-manager Cloudflare DNS-01 `ClusterIssuer` (in the config
  package) references a user-created `cloudflare-api-token` Secret in the `cert-manager`
  namespace. A secrets manager (e.g. Infisical Cloud + Operator) may be introduced after
  UDS Core is up; the `ClusterIssuer` references a Secret by name, so that swap is
  non-breaking.
