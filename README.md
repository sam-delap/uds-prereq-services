# uds-prereq-services

UDS Core **prerequisite services**, packaged as a versioned Zarf/OCI artifact for
home lab (and other bare-metal) clusters.

This package installs the prerequisite _services_ (MetalLB + cert-manager) and ships
**opinionated default configuration** for them as **optional, on-by-default components**.
Consumers get a working home lab setup out of the box, and can opt out of any config
component to bring their own (see [Configuration components](#configuration-components)).

## What's in this package

| Component             | Required | Default | Contents                                                                        |
| --------------------- | -------- | ------- | ------------------------------------------------------------------------------- |
| `metallb-namespace`   | yes      | —       | `metallb-system` namespace with privileged Pod Security Standards               |
| `metallb`             | yes      | —       | MetalLB controller + speakers (chart `0.14.9`, FRR enabled)                     |
| `cert-manager`        | yes      | —       | cert-manager controller, webhook, cainjector, and CRDs (chart `v1.21.0`)        |
| `metallb-config`      | no       | on      | `IPAddressPool` + `L2Advertisement` (IP range is a Zarf variable)               |
| `cert-manager-config` | no       | on      | `letsencrypt-staging` + `letsencrypt-prod` `ClusterIssuer`s (Cloudflare DNS-01) |

## Where this fits in the UDS layering model

```
zarf init (stock)
  → uds-prereq-services   ← THIS package (MetalLB + cert-manager services + default config)
  → UDS Core              ← separate repo/bundle
  → apps
```

MetalLB is a UDS Core prerequisite: it must exist before UDS Core Base so Istio's
ingress gateways (type `LoadBalancer`) can be assigned an address. See the UDS Core
[prerequisites](https://uds.defenseunicorns.com/reference/uds-core/prerequisites/) and
[functional layers](https://uds.defenseunicorns.com/reference/uds-core/functional-layers/) docs.

## Configuration components

The `metallb-config` and `cert-manager-config` components are **optional but deploy by
default** (`required: false`, `default: true`). Under `zarf package deploy --confirm`
they are applied automatically. Because they are ordered after the services that own
their CRDs, and each one waits for its CRD to be `established`, they apply cleanly.

### Overriding the default values

The config is templated with Zarf variables (defaults are the author's home lab values).
Override them at deploy time with `--set`:

| Variable            | Default                       | Purpose                                     |
| ------------------- | ----------------------------- | ------------------------------------------- |
| `METALLB_IP_RANGE`  | `192.168.1.200-192.168.1.210` | MetalLB `IPAddressPool` address range       |
| `ACME_EMAIL`        | `admin@sams-club-it.com`      | Let's Encrypt contact email                 |
| `DNS_ZONE`          | `sams-club-it.com`            | Zone the Cloudflare DNS-01 solver serves    |

```bash
uds run deploy --set VERSION=0.1.0 \
  --set METALLB_IP_RANGE=10.0.0.100-10.0.0.120 \
  --set ACME_EMAIL=you@example.com \
  --set DNS_ZONE=example.com
```

### Bringing your own config (opt out)

To supply your own MetalLB pools or ClusterIssuers instead, deselect the component(s):

```bash
# raw zarf deploy — deselect with a leading dash
zarf package deploy oci://ghcr.io/sam-delap/uds-prereq-services:0.1.0 \
  --components=-metallb-config,-cert-manager-config --confirm
```

In a **UDS bundle**, optional components are *not* deployed unless listed explicitly.
List the config components under `optionalComponents` to keep them, or omit them to opt
out:

```yaml
packages:
  - name: uds-prereq-services
    repository: ghcr.io/sam-delap/uds-prereq-services
    ref: 0.1.0
    optionalComponents:
      - metallb-config        # omit to bring your own MetalLB config
      - cert-manager-config   # omit to bring your own ClusterIssuers
```

### Cloudflare DNS-01 assumption

The `cert-manager-config` ClusterIssuers hardcode a **Cloudflare** DNS-01 solver. Only
`ACME_EMAIL` and `DNS_ZONE` are configurable — the solver block itself is Cloudflare-shaped.
If you use another DNS provider (Route53, Google, Azure, …), opt out of
`cert-manager-config` and supply your own ClusterIssuers.

### Required secret (out-of-band)

The ClusterIssuers reference a Cloudflare API token Secret you must create yourself — it
is never committed to git:

```bash
kubectl create secret generic cloudflare-api-token \
  --namespace cert-manager \
  --from-literal=api-token='<cloudflare-dns-edit-token>'
```

The token needs `Zone:DNS:Edit` permission for your `DNS_ZONE`.

### Let's Encrypt: staging vs. prod

Two `ClusterIssuer`s are created. `Certificate` resources select which one to use via
`issuerRef.name`. Validate issuance against `letsencrypt-staging` first (untrusted, no
rate limits), then flip the `issuerRef` to `letsencrypt-prod` and delete the old secret
to force re-issuance of a trusted cert.

## Building & publishing

Versions are managed by [semantic-release](https://semantic-release.gitbook.io/) from
Conventional Commits. On push to `main`, CI computes the next version, injects it into
`metadata.version` via `--set VERSION=`, and publishes to
`oci://ghcr.io/sam-delap/uds-prereq-services`.

Local build (uses `uds run`):

```bash
uds run build --set VERSION=0.1.0        # single arch (ARCH var, default amd64)
uds run publish --set VERSION=0.1.0
uds run build-all --set VERSION=0.1.0    # both amd64 + arm64 (matches CI)
uds run publish-all --set VERSION=0.1.0
uds run lint
```

Or directly with Zarf (one tarball per architecture):

```bash
zarf package create . --set VERSION=0.1.0 --architecture amd64 --confirm
zarf package create . --set VERSION=0.1.0 --architecture arm64 --confirm
zarf package publish zarf-package-uds-prereq-services-amd64-0.1.0.tar.zst oci://ghcr.io/sam-delap --confirm
zarf package publish zarf-package-uds-prereq-services-arm64-0.1.0.tar.zst oci://ghcr.io/sam-delap --confirm
```

## Notes

- **Supersedes `uds-prereq-config`.** The default config that previously lived in the
  separate `uds-prereq-config` package now ships here as optional components. That
  separate package is no longer needed.
- **Domain:** the wider home lab standardizes on `uds.sams-club-it.com`. The wildcard
  `*.uds.sams-club-it.com` `Certificate` itself is **not** created here — it lives in the
  UDS Core repo where Istio consumes it.
- **Secrets:** the `cloudflare-api-token` Secret is created out-of-band (above). A secrets
  manager (e.g. Infisical) may be introduced later; the ClusterIssuers reference the
  Secret by name, so that swap is non-breaking.
