# Stackdriver Exporter

Prometheus exporter for Stackdriver, allowing for Google Cloud metrics.
You must have appropriate IAM permissions for this exporter to work.
If you are passing in an IAM key then you must have:

* monitoring.metricDescriptors.list
* monitoring.timeSeries.list

These are contained within `roles/monitoring.viewer`.
If you're using legacy access scopes, then you must have `https://www.googleapis.com/auth/monitoring.read`.

Learn more: <https://github.com/prometheus-community/stackdriver_exporter>

This chart creates a Stackdriver-Exporter deployment on a
[Kubernetes](http://kubernetes.io) cluster using the [Helm](https://helm.sh)
package manager.

## Prerequisites

- Kubernetes 1.8+ with Beta APIs enabled

## Get Repo Info

```console
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

_See [helm repo](https://helm.sh/docs/helm/helm_repo/) for command documentation._

## Install Chart

```console
# Helm 3
$ helm install [RELEASE_NAME] prometheus-community/prometheus-stackdriver-exporter --set stackdriver.projectId=google-project-name

# Helm 2
$ helm install --name [RELEASE_NAME] prometheus-community/prometheus-stackdriver-exporter --set stackdriver.projectId=google-project-name
```

The command deploys Stackdriver-Exporter on the Kubernetes cluster using the default configuration.

_See [configuration](#configuration) below._

_See [helm install](https://helm.sh/docs/helm/helm_install/) for command documentation._

## Uninstall Chart

```console
# Helm 3
$ helm uninstall [RELEASE_NAME]

# Helm 2
# helm delete --purge [RELEASE_NAME]
```

This removes all the Kubernetes components associated with the chart and deletes the release.

_See [helm uninstall](https://helm.sh/docs/helm/helm_uninstall/) for command documentation._

## Upgrading Chart

```console
# Helm 3 or 2
$ helm upgrade [RELEASE_NAME] [CHART] --install
```

_See [helm upgrade](https://helm.sh/docs/helm/helm_upgrade/) for command documentation._

## Configuration

See [Customizing the Chart Before Installing](https://helm.sh/docs/intro/using_helm/#customizing-the-chart-before-installing).
To see all configurable options with detailed comments, visit the chart's [values.yaml](./values.yaml), or run these configuration commands:

```console
# Helm 2
$ helm inspect values prometheus-community/prometheus-stackdriver-exporter

# Helm 3
$ helm show values prometheus-community/prometheus-stackdriver-exporter
```

> **Tip**: You can use the default [values.yaml](values.yaml), as long as you provide a value for stackdriver.projectId

## Google Storage Metrics

In order to get metrics for GCS you need to ensure the metrics interval is >
24h.  You can read more information about this in [this bug
report](https://github.com/frodenas/stackdriver_exporter/issues/14).

The easiest way to do this is to create two separate exporters with different
prefixes and intervals, to ensure you gather all appropriate metrics.
