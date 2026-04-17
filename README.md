[![CircleCI](https://dl.circleci.com/status-badge/img/gh/giantswarm/grafana-organization-app/tree/main.svg?style=svg)](https://dl.circleci.com/status-badge/redirect/gh/giantswarm/grafana-organization-app/tree/main)
[![OpenSSF Scorecard](https://api.securityscorecards.dev/projects/github.com/giantswarm/grafana-organization-app/badge)](https://securityscorecards.dev/viewer/?uri=github.com/giantswarm/grafana-organization-app)

# grafana-organization chart

A lightweight Helm chart that ships a single [`GrafanaOrganization`](https://docs.giantswarm.io/reference/platform-api/crd/grafanaorganizations.observability.giantswarm.io/)
custom resource (group `observability.giantswarm.io`) to be installed on a Giant
Swarm management cluster. The CR is reconciled by the
[observability-operator](https://github.com/giantswarm/observability-operator),
which provisions the corresponding Grafana organization along with its
tenant-scoped datasources (Mimir, Loki, Tempo, Alertmanager) and RBAC bindings.

This app is intended for Giant Swarm internal use, deployed via the
`giantswarm-operations-platform-catalog`.

## Installing

This chart is published to the `giantswarm-operations-platform-catalog` and is
installed on management clusters via an `App` CR, typically managed through
GitOps in the `workload-clusters-fleet` / `management-cluster-bases`
repositories.

See the [Giant Swarm app developer guide](https://handbook.giantswarm.io/docs/dev-and-releng/app-developer-processes/adding_app_to_appcatalog/)
for details on how managed apps are packaged and rolled out.

## Configuring

Once the `GrafanaOrganization` template is added, the chart will expose values
for `displayName`, `rbac` (admins/editors/viewers) and `tenants`. Until then,
the chart has no configurable values.

See the [Creating a Grafana organization](https://docs.giantswarm.io/overview/observability/configuration/multi-tenancy/creating-grafana-organization)
tutorial for background on the fields exposed by the CRD.

## Compatibility

Requires the `GrafanaOrganization` CRD to be installed on the target
management cluster (provided by `observability-operator`).

## See also

- [observability-operator](https://github.com/giantswarm/observability-operator) – reconciles the `GrafanaOrganization` CR.
