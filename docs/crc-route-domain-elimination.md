# CRC route domain elimination findings

This repo exposes CRC/OpenShift Local through per-cluster custom domains such as
`crc02.testing` and `apps-crc02.testing`.

## Goal tested

Remove every remaining OpenShift Route using CRC's original domains:

- `*.apps-crc.testing`
- `*.crc.testing`

and keep only per-cluster custom route domains, for example:

- `*.apps-crc02.testing`
- `api.crc02.testing`

## Finding

Full in-place elimination of the old generated route domains is not safely
supported on an already-created CRC cluster.

The root cluster ingress domain is immutable:

```text
oc patch ingress.config.openshift.io cluster --type=merge \
  -p '{"spec":{"domain":"apps-crc02.testing"}}'

The Ingress "cluster" is invalid: spec.domain: Invalid value:
"apps-crc02.testing": domain is immutable once set
```

Because `spec.domain` remains `apps-crc.testing`, some OpenShift operators keep
owning/reconciling default routes on that domain.

## Operator behavior observed on crc02

### Console / downloads

Deleting these old routes:

- `openshift-console/console`
- `openshift-console/downloads`

while the Console operator is `Managed` causes them to be recreated on
`apps-crc.testing`.

Setting the Console operator to `Unmanaged` prevents recreation, but the
`console` clusteroperator becomes `Unknown`, so this is not acceptable for a
healthy deployment.

A ValidatingAdmissionPolicy denying non-`apps-crc02.testing` Route hosts blocks
normal users, but causes Console operator degradation:

```text
ConsoleDefaultRouteSyncDegraded: routes.route.openshift.io "console" is forbidden
DownloadsDefaultRouteSyncDegraded: routes.route.openshift.io "downloads" is forbidden
```

### Ingress canary

The default ingress canary route is required by the Ingress operator:

- `openshift-ingress-canary/canary`

Deleting or blocking it causes ingress degradation:

```text
CanaryChecksSucceeding=Unknown (CanaryRouteDoesNotExist: Canary route does not exist)
```

Creating a parallel custom-domain canary route works externally, but does not
replace the operator-required default route.

### Image registry

The image registry has a supported operator setting and is safe to move:

```yaml
spec:
  defaultRoute: false
  routes:
  - name: registry-crc02
    hostname: default-route-openshift-image-registry.apps-crc02.testing
```

## Current supported approach

Keep operator-required old defaults where OpenShift insists on them, and expose
all externally used routes via custom-domain routes:

- `openshift-authentication/oauth-openshift` -> `oauth-openshift.apps-crc02.testing`
- `openshift-console/console-custom` -> `console-openshift-console.apps-crc02.testing`
- `openshift-console/downloads-crc02` -> `downloads-openshift-console.apps-crc02.testing`
- `openshift-image-registry/registry-crc02` -> `default-route-openshift-image-registry.apps-crc02.testing`
- `openshift-ingress-canary/canary-crc02` -> `canary-openshift-ingress-canary.apps-crc02.testing`

This keeps operators healthy and all externally relevant endpoints reachable from
Hermes/VPN.

## If strict zero old-domain routes is required

Recreate the CRC/OpenShift cluster with the desired ingress domain from initial
bootstrap, if CRC/OpenShift Local supports that for the target version. It cannot
be changed in-place via `ingress.config.openshift.io/cluster.spec.domain` after
cluster creation.
