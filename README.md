[![CircleCI](https://dl.circleci.com/status-badge/img/gh/giantswarm/grafana-organization-app/tree/main.svg?style=svg)](https://dl.circleci.com/status-badge/redirect/gh/giantswarm/grafana-organization-app/tree/main)
[![OpenSSF Scorecard](https://api.securityscorecards.dev/projects/github.com/giantswarm/grafana-organization-app/badge)](https://securityscorecards.dev/viewer/?uri=github.com/giantswarm/grafana-organization-app)

# grafana-organization chart

A Helm chart that ships a single [`GrafanaOrganization`](https://docs.giantswarm.io/reference/platform-api/crd/grafanaorganizations.observability.giantswarm.io/) custom resource.
The CR is reconciled by the [observability-operator](https://github.com/giantswarm/observability-operator), which provisions the corresponding Grafana organization along with its tenant-scoped datasources and RBAC bindings.

## Sharing GrafanaOrganization access control

Values are split between Giant Swarm and customer control:

| Path | Owner | Pinned via |
| --- | --- | --- |
| `organizationName`, `displayName`, `rbac.*`, `tenants`, `customer.allowedRoles` | Giant Swarm | `HelmRelease.spec.values` (highest merge priority) |
| `customer.rbac.admins`, `customer.rbac.editors`, `customer.rbac.viewers` | Customer | A customer-editable ConfigMap referenced via `HelmRelease.spec.valuesFrom` |

The chart unions `rbac.<role>` (GS) with `customer.rbac.<role>` (customer) to produce the final `spec.rbac.<role>` on the rendered CR.

The `customer.allowedRoles` gate controls which roles honor customer input:

- `allowedRoles: []` - GS-only mode; all customer contributions are dropped.
- `allowedRoles: ["viewers"]` - customer may contribute viewers; admin/editor contributions are dropped.
- `allowedRoles: ["admins", "editors", "viewers"]` - fully-shared org.

If a customer supplies configuration that is invalid or not permitted, it will be dropped.
Dropped contributions are surfaced via the `grafana-organization-app.giantswarm.io/warnings` annotation on the rendered CR, encoded as a JSON array of human-readable messages.

## Example HelmRelease shapes

To use this chart, create a HelmRelease, and reference the Giant Swarm and customer ConfigMaps.

These live in `management-cluster-bases` / `shared-configs`, not in this repository.

### GS-only organization

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: grafana-organization-sample
  namespace: org-giantswarm
spec:
  interval: 1m
  chart:
    spec:
      chart: grafana-organization
      version: 1.0.0
      sourceRef:
        kind: HelmRepository
        name: giantswarm-operations-platform
        namespace: flux-giantswarm
  install:
    remediation:
      retries: -1
  upgrade:
    remediation:
      retries: -1
      remediateLastFailure: false
  valuesFrom:
    - kind: ConfigMap
      name: grafana-organization-sample-konfiguration
      valuesKey: configmap-values.yaml
  values:
    customer:
      allowedRoles: []
    rbac:
      admins:
        - "giantswarm:sample-admins"
      editors:
        - "giantswarm:sample-editors"
      viewers:
        - "giantswarm:sample-viewers"
```

### Shared-access organization

Extend the HelmRelease to reference an additional ConfigMap in order to allow customers to add their own groups.

The customer ConfigMap must be listed *before* the Giant Swarm konfiguration, otherwise our bound groups may be dropped.

```yaml
# ...
spec:
  valuesFrom:
    # Customer-editable, optional
    - kind: ConfigMap
      name: grafana-organization-sample-viewer-groups
      valuesKey: values
      optional: true
    # GS per-MC config rendered by konfigure-operator.
    - kind: ConfigMap
      name: grafana-organization-sample-konfiguration
      valuesKey: configmap-values.yaml
  values:
    customer:
      allowedRoles: ["viewers"]
    rbac:
      admins:
        - "giantswarm:sample-admins"
      editors:
        - "giantswarm:sample-editors"
      viewers:
        - "giantswarm:sample-viewers"
```

### Customer ConfigMap

The customer uses the referenced ConfigMap to manage their Grafana organization users and groups.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-organization-sample-viewer-groups
  namespace: org-acme
data:
  values: |
    customer:
      rbac:
        viewers:
          - "customer:platform-engineers"
          - "customer:sre-oncall"
```

See the [Creating a Grafana organization](https://docs.giantswarm.io/overview/observability/configuration/multi-tenancy/creating-grafana-organization)
tutorial for background on the fields exposed by the CRD.

## Compatibility

Requires the `GrafanaOrganization` CRD to be installed on the target management cluster .

## See also

- [observability-operator](https://github.com/giantswarm/observability-operator) – reconciles the `GrafanaOrganization` CR.
