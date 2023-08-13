# Overview

This chart installs the kube-prometheus-stack, which forms the basis of a lot
of our monitoring. The bulk of the heavy-lifting is done by the sub-chart which
is the actual
[kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/blob/main/charts/kube-prometheus-stack/README.md)
chart, and there is a little Kappa-specific configuration on top.

This chart installs, amongst other components:

* prometheus itself - to scrape and store metrics
* alertmanager - to aggregate and route prometheus alerts
* node-exporter - as a DaemonSet on all K8S nodes to monitor host-specific
  metrics
* grafana - to graph data

## Implementation

As usual we implement application-specific variables in the `values.yaml` file,
and also merge in the `environment-values` file from the parent directory to
give us some values that are common across the environment.

## Application

Note from the application-manifest - we use ArgoCD in
[replace](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/#replace-resource-instead-of-applying-changes)
mode for this chart. The reason is that the chart ships several extremely large
CRD files which cannot fit properly in the Kubernetes
`kubectl.kubernetes.io/last-applied-configuration` annotation, and as such this
chart cannot properly apply.

Later versions of ArgoCD (2.5+) have the
[server-side-apply](kubectl.kubernetes.io/last-applied-configuration) option
which may be a better fit for us.

## Chart Configuration

The kube-prometheus-stack subchart is configurable with a vast array of
[values](https://github.com/prometheus-community/helm-charts/blob/main/charts/)
kube-prometheus-stack/values.yaml). Note that in our chart this chart is
defined as a _subchart_, so be sure to indent by two extra spaces if copying
and pasting.

## Permanent Storage

We define a PVC for Prometheus itself in the chart. This is important to
ensure we preserve metrics history. AlertManager and Grafana do not have
permanent storage since they are only a view onto the data supplied by
Prometheus. The implication of this is that dashboards _must_ be defined in
code alongside applications if they are to survice re-starts and
re-deployments.

## Prometheus native vs. Kubernetes Configuration

Prometheus configuration variables are often named using snake_case. This is
not idiomatic in Kubernetes where camelCase is preferred. As a result the names
of configuration key/value pairs when configuring the Prometheus Operator (and
components) are often different.

To avoid pain and frustration, review the
[api spec](https://prometheus-operator.dev/docs/operator/api) when making
configuration changes to Prometheus objects within Kubernetes.

# Overview of Applications

## What is Prometheus?

Prometheus is a database optimised for the storage and query of time-series
data. Using Prometheus we can ingest metrics about our systems and
applications, graph and analyse the data on it, and build alerting rules for
the same data.

### Data Model

In common with other TSDBs such as OpenTSDB and InfluxDB, Prometheus stores a
metric under a _metric name_ and an arbitrary series of key value pairs known
as _metric labels_. The addition of labels to the data model allows for
powerful queries. Consider a metric such as `cpu_time`. We can now add labels
to differentiate instances of that data - such as `node_name`,
`availability_zone`, `application_name` and so forth. Prometheus allows us to
express queries that aggregate across those labels - so gathering a sum of
`cpu_time` for each value of `availability_zone` allows us to report on our
data in interesting ways.

Prometheus supports four types of metric:

* [Counter](https://prometheus.io/docs/concepts/metric_types/#counter) -
monotonically increasing counters, such as how many widgets have I processed
* [Gauge](https://prometheus.io/docs/concepts/metric_types/#gauge) - a metric
that can arbitrarily go up and down such as disk_space_used
* [Histogram](https://prometheus.io/docs/concepts/metric_types/#histogram) -
allowing sampling of a metric into configurable buckets
* [Summary](https://prometheus.io/docs/concepts/metric_types/#summary) -
similar
to histograms with some differences in possible queries

### Data Transfer Model

Prometheus works in a pull fashion. Applications make metrics available over
http using a defined format. A sample scrape from a Prometheus endpoint looks
like this:

```
curl localhost:9100/metrics 2>/dev/null | head -20
# HELP go_gc_duration_seconds A summary of the pause duration of garbage
collection cycles.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 1.5238e-05
go_gc_duration_seconds{quantile="0.25"} 1.9818e-05
go_gc_duration_seconds{quantile="0.5"} 2.4165e-05
go_gc_duration_seconds{quantile="0.75"} 0.00015544
go_gc_duration_seconds{quantile="1"} 0.00015544
go_gc_duration_seconds_sum 0.000214661
go_gc_duration_seconds_count 4
# HELP go_goroutines Number of goroutines that currently exist.
# TYPE go_goroutines gauge
go_goroutines 7
```
Here we can see two metrics - one summary, and one gauge.

Prometheus can be configured to scrape these targets at configurable
frequency. This allows a flexible and scalable system in which applications
make metrics available to any Prometheus' - we can run different Prometheus
instances for different environments, different parts of the application, or
multiple instances for resliency.

### What are Prometheus Rules?

Prometheus allows the definition of [alerting rules](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/).
These are expressed in standard Prometheus query syntax, and will cause
Prometheus to generate alerts when the rule is triggered.

## What is AlertManager?

While Prometheus can generate alerts, it contains no logic to aggregate or
route those alerts. This functionality is left to the
[AlertManager](https://prometheus.io/docs/alerting/latest/alertmanager/)
application. AlertManager will ingest alerts from one or more Prometheus
instances, group and aggregate them according to user-defined criteria, then
route them to one or more alert receivers.

Alert receivers can include Slack or PagerDuty. The configuration is
sufficiently flexible that we can ensure high priority alerts to to Pagerduty,
modify the timing for re-alerting, etc..

We can also silence alerts for configurable time periods with an annotation.

## What is Grafana?

[Grafana](https://grafana.com/docs/grafana/latest/?pg=oss-graf&plcmt=hero-btn-2)
is a stand-alone application which will graph data from TSDBs such as
Prometheus. It is sufficiently common to deploy the two together that the
kube-prometheus-stack chart includes both.

The basis of Grafana is configurable dashboards, allowing us to build groups of
graphs that show our operational status. Graphs can include
[variables](https://grafana.com/docs/grafana/latest/dashboards/variables/?pg=oss-graf&plcmt=hero-btn-2)
which allow us to create interactive dashboards showing values for selectable
label-values.

## Integration with Kubernetes - the Prometheus Operator

The kube-prometheus-stack chart ships with the [Prometheus
Operator](https://github.com/prometheus-operator/prometheus-operator) which
manages the instances of Prometheus and AlertManager. Importantly, the operator
allows us to manage the configuration of Prometheus dynamically - we can define
scrape targets, alert rules, and so forth, as Kubernetes objects.

# Defining configuration for kube-prometheus-stack

## Scrape Targets

We can define
[ServiceMonitors](https://prometheus-operator.dev/docs/operator/design/#servicemonitor)
as targets to scrape. This requires presenting the http endpoints which serve
metric using Kubernetes services (this is a common pattern). Prometheus will
resolve all the endpoints of a given service, and scrape them all. This means
that if we run multiple instances of an applicaiton for resilience or load
balancing purposes, Prometheus will scrape all of the copies of our
application.

An example ServiceMonitor might look like this:

```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: example-app
spec:
  selector:
    matchLabels:
      app: example-app
  endpoints:
  - port: web
  namespaceSelector:
    any: true
```

Notably - we define label selectors on scrape target definitions - this allows
us to use labels on our applications to define scraping.

Additionally, we can define
[PodMonitors](https://prometheus-operator.dev/docs/operator/design/#podmonitor)
 which monitor pods directly, if no service is fronting them. This is a less
common pattern since applications which expose an http endpoint typically have
a service defined.

## Prometheus Rules

Prometheus alerting rules can be defined as
[PrometheusRule](https://prometheus-operator.dev/docs/operator/design/#prometheusrule)
K8S objects. An example syntax, taken from the default K8S rules, is:
```
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  labels:
    app: kube-prometheus-stack
    release: external-kube-prometheus-stack
  name: kappa-prometheus-general.rules
  namespace: prometheus
spec:
  groups:
  - name: general.rules
    rules:
    - alert: TargetDown
      annotations:
        description: '{{ printf "%.4g" $value }}% of the {{ $labels.job }}/{{
$labels.service
          }} targets in {{ $labels.namespace }} namespace are down.'
        runbook_url:
https://runbooks.prometheus-operator.dev/runbooks/general/targetdown
        summary: One or more targets are unreachable.
      expr: 100 * (count(up == 0) BY (job, namespace, service) / count(up) BY
(job,
        namespace, service)) > 10
      for: 10m
      labels:
        severity: warning
```

*Note* that alert definitions can contain a runbook_url. At Kappa we should
make use of this to provide operational runbooks with our applications.

## Grafana Dashboards

Grafana dashboads can be supplied as Kubernetes configMaps. We can use the
following Helm template to include a dashboard with our application. Note that
the definition itself is kept in an adjacent file and included into the
configMap template using a .Files.Get stanza.

```
# This file will create Kubernernetes configMap objects to hold a Grafana
# dashboard. We use one configMap per dashboard.
{{- range .Values.monitoring.grafanaDashboards }}
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    grafana_dashboard: "1"
  name: kappa-grafana-dashboard-{{ . }}
  namespace: prometheus
data:
  grafana-dashboard-{{ . }}.json: |-
    {{- "\n" }}
    {{- $.Files.Get ( printf "files/grafana-dashboard-%s.json" . ) | indent 4
}}
---
{{- end }}
```

This allows us to design a dashboard using the Grafana built in GUI, and when
ready, export the dashboard json directly for inclusion in the ArgoCD
respository.

## Monitoring Configuration with Applications

One pattern which we should enforce at Kappa is to keep monitoring and
alerting configurations alongside the apps they represent. Thus, definitions
of scrape targets, alerting rules, and (application-specific) dashboards
should live in the same chart which deploys the application. This allows us to
operate in a self-service manner.

# Prometheus/Alert Manager at Kappa

Our services are presented using Kubernetes Ingress rules. At the time of
writing, the development environment had the following URLs defined.

* prometheus.dev.kappapay.com
* alertmanager.dev.kappapay.com
* grafana.dev.kappapay.com

Similar services for qa/prod can be accessed by substituting the environment
name as required.

## AlertManager Configuration

By default, AlertManager is set up to receive alerts and... do nothing with
them. In order to become useful, it needs to be configured with
[receivers](https://prometheus.io/docs/alerting/latest/configuration/#receiver) and
[routes](https://prometheus.io/docs/alerting/latest/configuration/#route).

We define two receivers as part of our configuration:

* [PagerDuty](https://prometheus.io/docs/alerting/latest/configuration/#pagerduty_config) is a
  system for managing alerts and on-call rotas
* [Slack](https://prometheus.io/docs/alerting/latest/configuration/#slack_config) is an instant messaging app

AlertManager contains native routing functionality for both. Note that as part
of our configuration we define custom templates to make the alerts a little
more palateable.

Webhook URLs for both Slack and PagerDuty are considered secret material, and
are therefore declared as Kubernetes Secrets. They are kept in the infra vault
of 1Password.
