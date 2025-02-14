---
title: "Monitoring"
linkTitle: "Monitoring"
weight: 30
description: >
  Each Spinnaker microservice is instrumented with numerous metrics exposed via a built in endpoint.
---


> The Spinnaker Observability plugin replaces the Spinnaker monitoring daemon
 which was deprecated as of Spinnaker release 1.20.

Each Spinnaker microservice is instrumented with numerous metrics exposed
via a built in endpoint. Monitoring spinnaker typically involves the
Spinnaker Observability plugin, which collects metrics reported by each
microservice instance and reports them to a third-party monitoring system
which you then use to view overview dashboards, receive alerts, and
informally browse depending on your needs.

The plugin currently supports two specific third-party systems:
[Prometheus](https://prometheus.io/),
and [New Relic](https://newrelic.com/). The plugin is
extensible so that it should be straightforward to add other systems as well.

Spinnaker publishes internal metrics using a multi-dimensional data model
based on "tags". The metrics, data-model, and usage are discussed further
in the sections [Consuming Metrics](#consuming-metrics) and in the
[Monitoring Reference document](/docs/reference/monitoring/).

You can also use the microservice HTTP endpoint `/spectator/metrics`
directly to scrape metrics yourself. The JSON document structure is
further documented in the Monitoring reference section.

The plugin can be configured to control which collected metrics are forwarded
to the persistent metrics store. This can alleviate costs and pressure on the
underlying metric stores depending on your situation.


To read more about the spinnaker monitoring daemon deprecation, check out the
[announcement](https://blog.spinnaker.io/announcing-the-new-spinnaker-observability-plugin-d7fbb17e1e07).


## Configuring the Spinnaker Observability Plugin

The instructions on how to install and configure the plugin can be found on
the [armory website](https://docs.armory.io/docs/armory-admin/prometheus-monitoring/#configure-monitoring-using-the-observability-plugin).

Additional information on how to configure the plugin can be found below.

* [Prometheus](https://github.com/armory-plugins/armory-observability-plugin#condensed-prometheus-example)
* [New Relic](https://github.com/armory-plugins/armory-observability-plugin#condensed-nr-example)

Once this is complete, you can optionally use the
[spinnaker-mixin](https://github.com/uneeq-oss/spinnaker-mixin) package to deploy pre-configured [Spinnaker
dashboards](#supplied-dashboards) for Grafana.

## Consuming metrics

Spinnaker publishes internal metrics using a multi-dimensional data model
based on "tags". Each "metric" has a name and type. Each data point is a
numeric value that is time-stamped at the time of reporting and tagged with
a set of one or more "label"="value" tags. These tag values are strings,
though some may have numeric-looking values. Taken together, the set of
tags convey the context for the reported measurement. Each of these
contexts forms a distinct time-series data stream.

For example a metric counting we requests may be tagged with a "status" label
and values indicating whether the call was successful or not. So rather
than having two metrics, one for successful calls and the other for
unsuccessful calls, there is a single metric, where the underlying
monitoring system can filter the successful from unsuccessful as you want
depending on how you wish to abstract and interpret the data. In practice
the metrics have many tags providing a lot of granularity and ways in
which you can aggregate and interpret them. The data model is described
further in [the Monitoring reference section](/docs/reference/monitoring/).

In practice there are relatively few distinct metric names (hundreds).
However when considering all the distinct time-series streams from the
different label values there are thousands of distinct streams. Some
metrics are tagged with the application or account they were used on
behalf of, so the number of streams may grow as the scope of your
deployment grows. Typically you will be aggregating these dimensions
together while breaking out along others. The granularity can come in
handy when it comes time to diagnose problems or investigate for deeper
understanding of runtime behaviors but you can aggregate across dimensions
(or parts of dimensions) when you dont care about that level of refinement.

### Types of metrics

There are two basic types of metrics currently supported,
*counters* and *gauges*.

  * __Counters__ are monotonically increasing values over the lifetime of
    the process. The process starts out with them at 0, then increments
    them as appropriate. Some counters may increase by 1 each time, such
    as the number of calls. Other counters may increase by an arbitrary
    (but non-negative) amount, such as number of bytes.

    Counters are scoped to the process they are in. If you have a counter
    in each of two different microservice replicas (including a restart),
    those counters will be independent of one another. Each process only
    knows about itself. The plugin adds a tag to each data point that
    identifies which instance it came from so that you can drill down
    into individual instances if you need. However, typically you will
    use your monitoring system to aggregate counters across all replicas.

    Counters are useful to determine rates. Given two points in time,
    the counter differences will be the measurement delta and the
    delta divided by the time difference will be the rate.
    (divide by another 1000000 to convert nanoseconds to milliseconds,
     such as for latency-oriented metrics or by another 100000000 for seconds,
     such as for call-rate metrics).

    * Spinnaker also has a special type of counter called a *Timer*.

      __Timers__ are used to measure timing information. These are
      always in nanoseconds. When consuming metrics straight from
      Spinnaker, a Timer will have two complementary time series.
      One will have a tag "statistic" with the value "count" and
      the other a tag with a "statistic" with the value "totalTime".

      The "count" represents the number of measurements taken.
      The "totalTime" represents the number of nanoseconds measured
      across all the calls. Dividing the "totalTime" by the "count"
      over some time window gives the latency over that time window.

      For example given a series of measurements for the pair of
      metrics example__count and example__totalTime, where the
      sum of the __count values was 5 and of the __totalTime values
      was 50000000, then dividing the time by count gives
      10000000 as an average time per count. Since this is in nanoseconds,
      we can divide by another 1000000000 to get 0.1 seconds per call.
      (or we could divide by 1000000 to get 100 milliseconds per call)

      Note that in order to do this, the tag bindings for the two measurements
      should be the same. Dividing measurements whose count has a success=true
      tag by times that have success=false tags wont give you the average time
      of the success calls (but would give you the average cost in total time
      spent for each successful call outcome if that is what you wanted.)


  * __Gauges__ are instantaneous value readings at a given point in time.
    Like counters, individual gauges are scoped to individual microservice
    instances. The daemon adds an instance tag to each data point so
    that you  can identify the particular instance if you want to, but
    typically you will use your monitoring system to aggregate across
    instances.

    Since gauges are instantaneous, the values between samples is
    unknown. Gauges are useful to determine current state, such as the
    size of queues. Sometimes answers to questions provided by gauges
    (e.g. active requests) might be answered by taking the difference
    in counters (e.g. completed requests - started requests).


### Example

Each microservice has a `controller.invocations` metric used to
instrument API calls into it. Since this is a timer, in practice
this is broken out into two 'controller.invocations\_\_count' and
'controller.invocations\_\_totalTime'.

These typically have the labels "controller", "method", "status",
"statusCode", and "success". Some microservices may add an additional
label such as "account" or "application" depending on the nature of
the microservices API.

These metrics will have several time series, such as those with the
following tag bindings:

| account    | controller             | method                      | status | statusCode | success |
|------------|------------------------|-----------------------------|--------|------------|---------|
| my-account | ClusterController      | getForAccountAndNameAndType | 2xx    | 200        | true    |
| my-account | ClusterController      | getForAccountAndNameAndType | 4xx    | 404        | false   |
| my-account | ClusterController      | getForAccountAndNameAndType | 4xx    | 400        | false   |
| None       | ApplicationsController | list                        | 2xx    | 200        | true    |
| None       | ApplicationsController | get                         | 2xx    | 200        | true    |
| None       | ApplicationsController | get                         | 4xx    | 404        | false   |

You can aggregate over the success tag to count successful calls vs failures,
perhaps breaking out by controller and/or method to see where the failures
were. You can break out by statusCode to see which controller and/or
method the errors are coming from and so forth.

Different metrics have different tags depending on their concept and
semantics. Some of these tags may be of more interest than others. In
the case above, some of the tags are at different levels of abstraction
and not actually independent. For example a 2xx status will always be
success=true and a non-2xx status code will always be success=false.
Which to use is a matter of convenience but given the status tag (which
can distinguish 4xx from 5xx errors) the success tag does not add any
additional time-series permutations since its value is not actually
independent.

### Supplied dashboards

Each of the supplied monitoring solutions provides a set of dashboards
tailored for that system. These are likely to evolve at different rates
so are not completely analogous or consistent across systems and might
not be completely consistent with the document. However, the gist and
intent described here should still hold since the monitoring intent is
the same across all the concrete systems.

As a rule of thumb, the dashboards currently prefer showing value differences
(over rates) for 1-minute sliding windows. This might change in the future.
Some of the caveats here are due to the choice to show values over rates, but
at this time the values seem more meaningful than rates, particularly where
there arent continuous streams of activity. Where latencies are shown, they
are computed using the counters from the past minute.

Depending on the chart and underlying monitoring system, some charts show
instantaneous value differences (between samples) while others show
differences over a sliding window. The accuracy of the timeline may vary
depending on the dashboard, but the underlying trends and relative signals
over time will still be valid.


#### Types of dashboards

There are several different dashboards. Each monitoring system has its own
implementation of the dashboards. See the corresponding documentation for
that system for more details or caveats.

*__Note__: Some systems might have an earlier prototype "KitchenSinkDashboard"
that has not yet been broken out into the individual dashboards. Most of the
information is still there, just all in the one dashboard.*

   * __*&lt;Microservice&gt;* Microservice__

     These dashboards are tailored for an individual microservice. As a rule
     of thumb they provide a system wide view of all replicas of a given
     microservice while also letting you isolate a particular instance. They
     show success/error counts and latencies for the different APIs the
     microservice offers as well as special metrics that are fundamental to
     the operation or responsibilities of that particular service.

   * __Spinnaker *&lt;Provider&gt;* API__

     These dashboards are tailored for a particular cloud provider. They
     show a system level perspective of Spinnaker's interaction with that
     provider. Depending on the provider, the dashboard details may vary.
     In general they offer a system wide view while also letting you isolate
     a particular instance and region, showing success/error counts and
     latencies for different resource interactions or individual operations.
     This provides visibility into what your deployment is doing and where
     any problems might be coming from.

   * __Minimal Spinnaker__

     The intent of this dashboard is provide the most essential or useful
     metrics to quickly suggest whether there are any issues and confirm
     Spinnaker is behaving normally. Your needs may vary so consult each of
     the other dashboards and consider refining your own. If you do, also
     consider sharing that back!
