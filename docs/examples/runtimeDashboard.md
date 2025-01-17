# Controller Runtime dashboard

The [controller runtime dashboard](https://github.com/grafana/grafonnet/blob/master/examples/runtimeDashboard) example shows how to compose a dashboard from reusable components. There are separate definitions of panels, variables and queries. The queries are combined with the panels and then the panels are grouped into rows. Eventually the panels and rows are rendered into a grid. Let's break it down.

Similarly to the simple dashboard, Grafonnet is imported through `g.libsonnet` and `row` becomes a shortcut for the row panel. Additionally the panels, variables and queries get imported.

```jsonnet
local g = import 'g.libsonnet';

local row = g.panel.row;

local panels = import './panels.libsonnet';
local variables = import './variables.libsonnet';
local queries = import './queries.libsonnet';
```

The dashboard gets initialized with a title and description. The tooltip is configured to share a crosshair and tooltip on all panels.

```jsonnet

g.dashboard.new('Controller Runtime')
+ g.dashboard.withDescription(|||
  Generic dashboard for controller-runtime based processes
  (https://github.com/kubernetes-sigs/controller-runtime)
|||)
+ g.dashboard.graphTooltip.withSharedCrosshair()
```

The variables get added to provide dropdowns for selecting the datasource, cluster, namespace and job values.

```jsonnet
+ g.dashboard.withVariables([
  variables.datasource,
  variables.cluster,
  variables.namespace,
  variables.job,
])
```

And eventually add the panels.

```jsonnet
+ g.dashboard.withPanels( /* ... */ )
```

## Panels

The panels are defined separately from the queries, this turns them into reusable components. A panels can be called with a title and query. Let's take the 'Threads' panel as an example.

```jsonnet
panels.timeSeries.seconds('GC Duration Mean', queries.gcDuration),
```

The 'Threads' panels uses the `base` timeSeries panel. Reusing the same panel definition for this and other panels increases consistency. The shortcuts for `fieldOverride` etc. are intended to make the code more concise.

```jsonnet
// from panels.libsonnet
local g = import 'g.libsonnet';

{
  timeseries: {
    local timeSeries = g.panel.timeSeries,
    local fieldOverride = g.panel.timeSeries.fieldOverride,
    local custom = timeSeries.fieldConfig.defaults.custom,
    local options = timeSeries.options,

    base(title, targets):
      timeSeries.new(title)
      + timeSeries.queryOptions.withTargets(targets)
      + timeSeries.queryOptions.withInterval('1m')
      + options.legend.withDisplayMode('table')
      + options.legend.withCalcs([
        'lastNotNull',
        'max',
      ])
      + custom.withFillOpacity(10)
      + custom.withShowPoints('never'),
  }
}
```

The 'GC Duration Mean' panel extends the `base` panel to display the duration in seconds on a logarithmic scale. The `cpuUsage` panel is a shortcut to `seconds` panel.

```jsonnet
{
  timeseries: {
    seconds(title, targets):
      self.base(title, targets)
      + timeSeries.standardOptions.withUnit('s')
      + custom.scaleDistribution.withType('log')
      + custom.scaleDistribution.withLog(10),

    cpuUsage: self.seconds,
  }
}
```

### Rows and Grid

Rows can be used to group panels and optionally collapse them, however using them within Jsonnet can be quite cumbersome. For example: all panels added after a row in the panel array will inevitably become part of the row, also the order in the array doesn't necessarily apply with how Grafana displays them. The `makeGrid` util function attempts to aid with this.

First the panels are consistently added to the rows so that the intention is clear from a Jsonnet perspective.

```jsonnet
    row.new('Process')
    + row.withPanels([
      panels.timeSeries.cpuUsage('CPU Usage', queries.cpuUsage),
      panels.timeSeries.memoryUsage('Memory Usage', queries.memUsage),
      panels.timeSeries.base('Goroutines', queries.goroutines),
      panels.timeSeries.base('Threads', queries.threads),
      panels.timeSeries.seconds('GC Duration Mean', queries.gcDuration),
    ]),
```

Second the panel array gets processed by `makeGrid` before added it to the dashboard.

```jsonnet
+ g.dashboard.withPanels(
  g.util.grid.makeGrid([
    row.new('Process')
    + row.withPanels([
      panels.timeSeries.cpuUsage('CPU Usage', queries.cpuUsage),
      panels.timeSeries.memoryUsage('Memory Usage', queries.memUsage),
      panels.timeSeries.base('Goroutines', queries.goroutines),
      panels.timeSeries.base('Threads', queries.threads),
      panels.timeSeries.seconds('GC Duration Mean', queries.gcDuration),
    ]),
    row.new('Kubernetes Client')
    + row.withCollapsed(true)
    + row.withPanels([
      panels.heatmap.base('Workqueue Waiting Duration Over Time', queries.wqDurationOverTime),
      panels.timeSeries.durationQuantile('Workqueue Waiting Duration Quantile', queries.wqDurationQuantile),
      panels.timeSeries.short('Workqueue Depth', queries.wqDepth),
      panels.timeSeries.short('Failed Requests', queries.failedRequests),
    ]),
    row.new('Controller Runtime')
    + row.withCollapsed(true)
    + row.withPanels([
      panels.heatmap.base('Reconciling Latency Over Time', queries.reconcilingLatencyOverTime),
      panels.timeSeries.durationQuantile('Reconciling Latency Quantile', queries.reconcilingDurationQuantile),
    ]),
  ], panelWidth=8)
)
```

## Queries

The queries are defined as separate objects. This allows us to swap out the Prometheus queries for Graphite queries while not having to change the dashboard. Additionally this makes it possible to reuse the queries on different panels or even in different dashboards.

Let's take a look at a query definition. The `cpuUsage` is an instance of a `prometheusQuery`. Note that the query definition leans heavily on the variables, making the datasource configurable and using variables in the query expression, setting values for the cluster, namespace and job labels. Finally this configures a `legendFormat`, telling Grafana which values to show in the legend.

```jsonnet
// from queries.libsonnet
local g = import './g.libsonnet';
local prometheusQuery = g.query.prometheus;

local variables = import './variables.libsonnet';

{
  cpuUsage:
    prometheusQuery.new(
      '$' + variables.datasource.name,
      |||
        sum by (cluster, namespace, job) (
            rate(
                process_cpu_seconds_total{
                    cluster=~"$cluster",
                    namespace=~"$namespace",
                    job=~"$job"
                }
            [$__rate_interval])
        )
      |||
    )
    + prometheusQuery.withIntervalFactor(2)
    + prometheusQuery.withLegendFormat(|||
      {{cluster}} - {{namespace}}
    |||),
}
```

## Variables

To make this dashboard dynamic, it uses variables. This allows the user to select and manipulate the data being displayed.

In this case there is a single 'datasource' variable and several 'query' variables. The 'query' variables depend on the datasource variable and then cascadingly depend on each other. This means that the values of for example `namespace` depend on the values from `datasource` and `cluster`.

```jsonnet
// variables.libsonnet
local g = import './g.libsonnet';
local var = g.dashboard.variable;

{
  datasource:
    var.datasource.new('datasource', 'prometheus')
    + var.datasource.withRegex('(ops|dev)-cortex'),

  cluster:
    var.query.new('cluster')
    + var.query.withDatasourceFromVariable(self.datasource)
    + var.query.queryTypes.withLabelValues(
      'cluster',
      'process_cpu_seconds_total',
    )
    + var.query.withRefresh('time')
    + var.query.selectionOptions.withMulti()
    + var.query.selectionOptions.withIncludeAll(),

  namespace:
    var.query.new('namespace')
    + var.query.withDatasourceFromVariable(self.datasource)
    + var.query.queryTypes.withLabelValues(
      'namespace',
      'process_cpu_seconds_total{cluster=~"$%s"}' % self.cluster.name,
    )
    + var.query.withRefresh('time'),

  job:
    var.query.new('job')
    + var.query.withDatasourceFromVariable(self.datasource)
    + var.query.queryTypes.withLabelValues(
      'job',
      'process_cpu_seconds_total{cluster=~"$%s", namespace=~"$%s"}'
      % [
        self.cluster.name,
        self.namespace.name,
      ],
    )
    + var.query.withRefresh('time'),
}

```

The full controller runtime dashboard example can be [found in the repo](https://github.com/grafana/grafonnet/blob/master/examples/runtimeDashboard).
