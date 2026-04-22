[![CircleCI](https://dl.circleci.com/status-badge/img/gh/giantswarm/grafana-organization-app/tree/main.svg?style=svg)](https://dl.circleci.com/status-badge/redirect/gh/giantswarm/grafana-organization-app/tree/main)
[![OpenSSF Scorecard](https://api.securityscorecards.dev/projects/github.com/giantswarm/grafana-organization-app/badge)](https://securityscorecards.dev/viewer/?uri=github.com/giantswarm/grafana-organization-app)

# grafana-organization chart

A Helm chart that ships a single [`GrafanaOrganization`](https://docs.giantswarm.io/reference/platform-api/crd/grafanaorganizations.observability.giantswarm.io/) custom resource.
The CR is reconciled by the [observability-operator](https://github.com/giantswarm/observability-operator), which provisions the corresponding Grafana organization along with its tenant-scoped datasources and RBAC bindings.

## Sharing GrafanaOrganization access control

Values are split between Giant Swarm and customer control:

| Source | Owner | Delivered via |
| --- | --- | --- |
| `organizationName`, `displayName`, `rbac.*`, `tenants`, `customer.allowedRoles`, `customer.configMapName` | Giant Swarm | `HelmRelease.spec.values` (highest merge priority) |
| `admins`, `editors`, `viewers` identity groups the customer contributes | Customer | A ConfigMap in the release namespace, looked up by the chart at render time |

The chart unions `rbac.<role>` (GS) with any customer-contributed group list
under `<role>` in the customer ConfigMap to produce the final `spec.rbac.<role>`
on the rendered CR.

The `customer.allowedRoles` gate controls which roles honor customer input:

- `allowedRoles: []` - GS-only mode; all customer contributions are dropped.
- `allowedRoles: ["viewers"]` - customer may contribute viewers; admin/editor contributions are dropped.
- `allowedRoles: ["admins", "editors", "viewers"]` - fully-shared org.

If a customer supplies configuration that is invalid or not permitted, it is
dropped. Dropped contributions are surfaced via the
`grafana-organization-app.giantswarm.io/warnings` annotation on the rendered
CR, encoded as a JSON array of human-readable messages.

## Customer ConfigMap format

The chart reads the customer's contributions at render time via Helm's `lookup` function.

**Name.** By default the chart looks for a ConfigMap in the release namespace
named

```text
<organizationName>-customer-groups
```

For example, if `organizationName` is `sample`, the chart looks for `sample-customer-groups`.

Giant Swarm can override this name per release via `customer.configMapName` in the HelmRelease's `spec.values`.

**Payload.** The customer's ConfigMap carries group mappings in `data.groups`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: sample-customer-groups
  namespace: org-acme
data:
  groups: |
    admins: []
    editors: []
    viewers:
      - "customer:platform-engineers"
      - "customer:sre-oncall"
```

Only roles listed in `customer.allowedRoles` are honored; entries under disallowed roles are dropped with a warning.

**Validation.** Each entry must be a `connector:group` string matching
`^[a-z0-9][a-z0-9-]{0,30}:[a-z0-9][a-z0-9._-]{0,62}$`, ≤ 95 characters, with at
most 50 entries per role. Violations are dropped and exposed as warnings on the CR.

## Shared organization access control

The example values below demonstrate creating an `application-telemetry` GrafanaOrganization.
Three Giant Swarm identity groups are granted `admin`, `editor`, and `viewer` access to the resulting organization.

Additionally, more `viewer` roles will be loaded from the `some-different-viewer-groups` ConfigMap, which is managed externally.

```yaml
values:
  organizationName: "application-telemetry"
  displayName: "Application Telemetry"
  customer:
    allowedRoles: ["viewers"]
    configMapName: "some-different-viewer-groups"
  rbac:
    admins:
      - "giantswarm:sample-admins"
    editors:
      - "giantswarm:sample-editors"
    viewers:
      - "giantswarm:sample-viewers"
```

See the [Creating a Grafana organization](https://docs.giantswarm.io/overview/observability/configuration/multi-tenancy/creating-grafana-organization)
tutorial for background on the fields exposed by the CRD.

## Compatibility

Requires the `GrafanaOrganization` CRD to be installed on the target management cluster.

## See also

- [observability-operator](https://github.com/giantswarm/observability-operator) – reconciles the `GrafanaOrganization` CR.
- [Giant Swarm Observability multi-tenancy](https://docs.giantswarm.io/overview/observability/configuration/multi-tenancy/) documentation.
