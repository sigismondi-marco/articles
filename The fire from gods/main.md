# The fire from gods. Searching for the real form of the software through monitoring and observability

## Warning

The following are notes I've taken during an analisys I made upon tools and metodologies for monitor and observe distributed systems. After the work I reviewed them to make them a little bit more discursive.

My primary goal was to evaluate the use of Prometheus for observe systems enforcing the exposure of contextual metrics of system's specific business. For sure my goal was not not became an expert about configure and deploy Prometheus so don't be disappointed.

Some - if not most - of the concepts are well known but I found useful to write them to give the writing a self-consistent form.

## Discover the fire

As everyone knows Prometheus is an open-source systems monitoring originally built at SoundCloud and then entered in the Cloud Native Computing Foundation family.

Even if it carries with it the reputation of a time series database is a lot more:

1. it collects timestamped metrics optionally labeled

1. it provides alerting rules and a ready to use integration with AlertMaganager to easily implement alerts and notifications

but more important

1. it uses a pull rather than push approach for collecting

1. it defined a metric format which became the de facto standard for monitoring system (about this take a look at the [OpenMetrics project](https://openmetrics.io/)

1. it defined query language, PromQl, that even it became the de facto standard for monitoring system

### Scraping

Prometheus scrapes targets: it means that as long as a system expose metrics in Prometheus format the only thing needed to collect data is to give Prometehus that endpoint.

Now it seems not so important and not so different from a push approach but actually it makes a lot of difference:

1. in the containers world adding things (think about sidecars) withouth change the application is daily work

1. even without containers is not so difficult (actually widely used) dinamically add things. Coming from Java I'm thinking about java agents

1. it is easier detect loss of signals: getting errors is easier to detect than absence of something that is what would happen in push scenario

By the way there are a couple of drawbacks:

1. every metrics must be stored somewhere by the monitored system (even placing data in RAM is somehow storing data) so the availabilty of metrics depends on both monitored and monitoring system storage

1. metrics are available after some time the events happened in the monitored system. This is caused by tha scrape interval

1. scraping is possible with systems that can act as servers (are able to respond to HTTP request). Thinking about IoT devices it became slightly difficult


### The Prometheus exposition format

Metrics are collected via HTTP request in text format as a sequence of lines and composed by:
1. the name
1. one or more labels thanks to which PromQL is able to filter or group
1. the value in float64 format

Timestamp are added by Prometheus when receives the data: that's why there is a discrepancy between metrics timestamp and the realt timing of events and the time stored.

This seems pretty obvious or slightly relevant for aggregated metrics (see histograms and summary) but could cause some troubles if you expect to gain a specific metric value for an exact time (maybe relying on on other input like logs).

Metrics follow the pattern

```
<metric name>{<label name>=<label value>, ...} value
```

For example 

```
# HELP grafana_page_response_status_total page http response status
# TYPE grafana_page_response_status_total counter
grafana_page_response_status_total{code="200"} 210
grafana_page_response_status_total{code="404"} 0
grafana_page_response_status_total{code="500"} 0
grafana_page_response_status_total{code="unknown"} 4

```

where:
- everything after `#` and until the line feed is a comment
- `grafana_page_response_status_total` is the metric name
- everything inside the brackets are labels with keys and values. The example shows nicely the use of labels for aggragating: in fact are reported 210 responses with `code="200"`
- after the closing curly bracket, separated by a white space, there is the value

From the HTTP point of view nothing fancy: `text/plain; version=0.0.4; charset=utf-8` as `Content-Type` and the list of metrics in the body

This an example of a request made with curl to a Grafana pod running locally (yes, Grafana natively expose metrics about itself in Prometheus format)

```

*   Trying 10.100.224.149:3000...
* Connected to grafana.observe.svc.cluster.local (10.100.224.149) port 3000 (#0)
> GET /metrics HTTP/1.1
> Host: grafana.observe.svc.cluster.local:3000
> User-Agent: curl/7.84.0-DEV
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Cache-Control: no-cache
< Content-Type: text/plain; version=0.0.4; charset=utf-8
< Expires: -1
< Pragma: no-cache
< X-Content-Type-Options: nosniff
< X-Frame-Options: deny
< X-Xss-Protection: 1; mode=block
< Date: Mon, 22 Aug 2022 10:35:05 GMT
< Transfer-Encoding: chunked
< 

# HELP access_evaluation_duration Histogram for the runtime of evaluation function.
# TYPE access_evaluation_duration histogram
access_evaluation_duration_bucket{le="1e-05"} 0
access_evaluation_duration_bucket{le="4e-05"} 0
access_evaluation_duration_bucket{le="0.00016"} 0
access_evaluation_duration_bucket{le="0.00064"} 0
access_evaluation_duration_bucket{le="0.00256"} 0
access_evaluation_duration_bucket{le="0.01024"} 0
access_evaluation_duration_bucket{le="0.04096"} 0
access_evaluation_duration_bucket{le="0.16384"} 0
access_evaluation_duration_bucket{le="0.65536"} 0
access_evaluation_duration_bucket{le="2.62144"} 0
access_evaluation_duration_bucket{le="+Inf"} 0
access_evaluation_duration_sum 0
access_evaluation_duration_count 0


```

### The four metrics types

One thing I didn't believe at first but that then I realized is that four metrics types can fit basically all the needs, even when defining custom metrics. 

Here they are:

#### Counter
Counter is something that always grows and never decreases unless restarted. Usefull for representing for example number of errors or requests served

```
# HELP prometheus_http_requests_total Counter of HTTP requests.
# TYPE prometheus_http_requests_total counter
prometheus_http_requests_total{code="200",handler="/-/healthy"} 46
prometheus_http_requests_total{code="200",handler="/-/ready"} 48
prometheus_http_requests_total{code="200",handler="/-/reload"} 1
prometheus_http_requests_total{code="200",handler="/metrics"} 16
```

Most of the time counters are used with some PromQL aggregator function like `rate` to calculate averages. 

For example

```
rate(prometheus_http_requests_total{code="200"}[5m])
```

calculates the numebr of request per second in the last 5 minutes.

By the way sometimes can be useful as absolute value. For example having a job that executes tasks, exposing the number of successfully completed tasks the number of total task it is possible to monitor the progress of the job.

#### Gauge

When measures go up and down then it's time for gauges. They fit pretty well, for example, when measuring resource usage but sometimes is usefull for metrics that looks like counters:
- concurrent requests
- concurrent active users
- server connection pool committed resources


```
# TYPE promhttp_metric_handler_requests_in_flight gauge
promhttp_metric_handler_requests_in_flight 1
```

#### Histogram

With histogram things became interesting: instead of return a single value during a scrape they return multiple items:
- the number of measurements expressed as counter and identified by the name of the metrics followed by the suffix `_count`
- the sum of all values of all measurements expressed as counter and identified by the name of the metrics followed by the suffix `_sum`
- a list of buckets distingued by a  label named `le` (the bucket upper bound) that counts the number of measurements falling under that specific bound. These are identified by the name of the metrics followed by the suffix `_bucket`

> **_NOTE:_** One thing I found nice is that histograms are basically a set of counters that share the same name prefix and distinguish values with labels and name suffix

```
# HELP prometheus_http_request_duration_seconds Histogram of latencies for HTTP requests.
# TYPE prometheus_http_request_duration_seconds histogram
prometheus_http_request_duration_seconds_bucket{handler="/metrics",le="0.1"} 16
prometheus_http_request_duration_seconds_bucket{handler="/metrics",le="0.2"} 16
prometheus_http_request_duration_seconds_bucket{handler="/metrics",le="0.4"} 16
prometheus_http_request_duration_seconds_bucket{handler="/metrics",le="1"} 16
prometheus_http_request_duration_seconds_bucket{handler="/metrics",le="3"} 16
prometheus_http_request_duration_seconds_bucket{handler="/metrics",le="8"} 16
prometheus_http_request_duration_seconds_bucket{handler="/metrics",le="20"} 16
prometheus_http_request_duration_seconds_bucket{handler="/metrics",le="60"} 16
prometheus_http_request_duration_seconds_bucket{handler="/metrics",le="120"} 16
prometheus_http_request_duration_seconds_bucket{handler="/metrics",le="+Inf"} 16
prometheus_http_request_duration_seconds_sum{handler="/metrics"} 0.1199457
prometheus_http_request_duration_seconds_count{handler="/metrics"} 16
```

So using request duration as metric how could look like an implementatio logic for the exposure?

Given that the upper bounds must be pre-defined and all values for `le` being known:
1. every time a reqeust is served its duration evaluated to check under what bounds it falls
1. after the bounds is found the respective counters is encreased
1. the `_count` is increased
1. the duration added to the `_sum` counter

>**_NOTE:_** One consequence of using inclusive upper bound as aggregation is that a measure can fall into more than a group: in the example above 16 requests are being served and all within 0.1 seconds (the value of `_sum` confirms this since it is 0.1199457) so all buckets contain 16 items and all are euqal to `_count` (tipically at least one bucket is equal to `_count` and is `+Inf`)

####  Summary

While histograms define upper bounds, summeries define quantiles so measurements are distributed into buckets basing upon distribution. As summeries returns `_sum` and `_count` following the same rules but the items
- do not provie specific suffix (like `_bucket` for histograms)
- measure the value of the specific quantile

```
# HELP go_gc_duration_seconds A summary of the pause duration of garbage collection cycles.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 9.66e-05
go_gc_duration_seconds{quantile="0.25"} 0.0001877
go_gc_duration_seconds{quantile="0.5"} 0.0001912
go_gc_duration_seconds{quantile="0.75"} 0.0002081
go_gc_duration_seconds{quantile="1"} 0.0015068
go_gc_duration_seconds_sum 0.0041706
go_gc_duration_seconds_count 13
```

Given the example above we can say that
- the minimum value is 9.66e-05: `quantile="0"`
- the max value is 0.0015068: `quantile="1"`
- the median is 0.0001912: `quantile="0.5"`
- `quantile="0.25"` and `quantile="0.275` represent respectively 25th and 75th percentile

>**_NOTE:_** One consequence of using quantile is that distribution can be calculate only after receiving all the mesaurements for the given time period (1 second in the example above). Furthermore calculation must be done by the monitored system so is more expensive

## Turn on the fire

Now that Prometheus seems more familiar what is missing is to make it run so let's check its architecture:

[![Prometheus architecture](https://prometheus.io/assets/architecture.png)]

_The Prometheus architecture taken directly from [Prometheus.io](https://prometheus.io/docs/introduction/overview/)_

Putting aside *Pushgateway* (that allows systems to push metrics to somewhere Promethues can pull inverting the approach) and *Alertmanager* (that handle alerts based on Prometheus rules integrating with notification services) the Prometheus server is standalone.

Quoting the FREQUENTLY ASKED QUESTIONS page 

> _The main Prometheus server runs standalone and has no external dependencies._

So no additional databases or other components are nedded (at first).

> **_NOTE:_** My goal is not to analyze how to fully and securely deploy Prometheus stack in a production enviroment so obviously this section cannot be considered exhaustive

The documentation propose different installation options

- Pre-compiled binaries
- From source
- Using Docker
- Using configuration management systems (Ansible, Chef, Puppet and SaltStack)

Since my study is related to a Kubernetes environment I picked Docker. 

Thare are different options when evaluating the way to handle kubernetes installation: 
- Helm
- Kustomize
- Operator
- just rely on standard kubernetes resources

I decided to use the [Prometheus Operator project](https://prometheus-operator.dev/).

It is beyond the scope of this writing a comparison between the different options but the main reason about my choice is that I was searching for something as responsive as possible and the Operator seems to me the most suitable

### Spin off

When talking about Operator in Kubernetes we talk about a pattern and not a tool. That's beacuse everything's needed already exists in vanilla Kubernetes: Custom Resources and Controllers

Let's start understanding the Resource: a Resource is an API that stores a collection of objects of a specific kind. 

For example executing an HTTP GET to `https://{k8s-api}/apis/apps/v1/namespaces/kube-system/deployments` will results in a list of deployments in the `kube-system` namespace while executing an HTTP DELETE at `https://{k8s-api}/api/v1/namespaces/default/pods/nginx` will delete the specific `nginx` object.

> **_TIPS:_** A quick way that I love to play with kubernetes API is the joint use of `--v=8` flag and  `proxy` command when using `kubectl`. The first change the log level so that the endpoints called by `kubectl` are shown and the second allow accessing kubernetes API on localhost bypassing authentication. 
>
> That's an exemple
> ```
> $ kubectl get pods -n kube-system --v=8
> I0823 14:30:38.117371   70216 loader.go:372] Config loaded from file:  C:\Users\[USER]\.kube\config
> I0823 14:30:38.302372   70216 cert_rotation.go:137] Starting client certificate rotation controller
> I0823 14:30:38.378981   70216 round_trippers.go:463] GET https://127.0.0.1:55790/api/v1/namespaces/kube-system/pods?limit=500
> ```
> 
> then in another terminal
>
> ```
> $ kubectl proxy --port=8080
> Starting to serve on 127.0.0.1:8080
> ```
>
> and back on the first terminal
>
> ```
> $ curl -X "GET" http://localhost:8080/api/v1/namespaces/kube-system/pods?limit=500
> {
>  "kind": "PodList",
>  "apiVersion": "v1",
>  "metadata": {
>    "resourceVersion": "43260"
>  },
>  "items": [...]
>    
> ```

Not surprisingly a Custom Resource is an API that stores a collection of API objects not provided by default but defined via [Custom Resource Definition](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/). Once the definition is submitted to kubernetes the API Server expose a new HTTP API that can be used to manage the custom objects.

Of course just the ability to handle objects that define some kind of state is pretty useless until some logic is provided: that's the job of Controllers that constantly watch at resources to ensure that the current state match the desired state.

This is the Operator Pattern and one possible use of is the deployment of clusters in Kubernetes.

>**_NOTE:_** For further reading about the Operato Pattern of course there is the [Kubernetes documentation](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/) but I found really clear the original article from CoreOS that proposed the pattern: [Introducing Operators: Putting Operational Knowledge into Software](https://web.archive.org/web/20170129131616/https://coreos.com/blog/introducing-operators.html).

### The Prometheus operator

> **_DISCLAIMER_**: For a detailed documentation and comprehensive list of the resource objects please refer to the project [webpage](https://prometheus-operator.dev) 

As written in the home page of the project [website](https://prometheus-operator.dev/): _The Prometheus Operator manages Prometheus clusters atop Kubernetes._

To do that it defines and manage custom resources which
- deploy pods (**Prometheus**) and rules (**PrometheusRule**) (rules that define alerts and recording rules that are precomputed expressions whose result is saved as time series)
- define monitoring target for pods (**PodMonitor**), static targets (**Probe**) or services (**ServiceMonitor**)
- deploy AlertManager pods (**Alertmanager**) and its configuration (**AlertmanagerConfig**)

> **_NOTE:_** Actually the Operator manage another Custom resource: **ThanosRuler**. Going deep into the details of Thanos is beyond my strength but just to give an overiview - combining a sidecar container in Prometheus pods and standalone components - it extends Prometheus for high availabilty. 
>
>It provides, for example, the ability to execute query and evaluate rules across different Promethues pods and extends the system storage to use Block Storage solutions instead of just the node's local storage (that is the Prometheus deafult).
The Prometheus Operator takes care just about the sidecar container for the Prometheus pods and the component that evaluate Prometheus rules (**ThanosRuler**). The choice - or the burden - of install other component is left to the user.

To give just an high level ovierview about how it works, the Proemetheus resource defines for example number of pods replicas, interval between scrapes and how many time wait target to respond before erroring.

**PodMonitor** and **ServiceMonitor** work almost similar: they select pods and services to monitor via label matching. **Probe** is quite different: since it is intended to monitor static targets it actually scrapes a `prober` that is some kind of service which privdes metrics.

So adding a target is just about to definine **PodMonitor**, **ServiceMonitor** or **Probe** custom resource.

## Epilogue

I made myself a question some times ago: *can I make a system evolves if I'm unable to properly inspect it?*

Until now it could seems that this writing is all about monitoring but it is not entirely true: my primary goal is to inspect the way in which a system should be deisgned to well be monitored and observed.

That's why I'm so interested in metrics data model, metrics format and push versus pull approach.

Implementing _monitoring by design_ not only means to provide in the system topology components that  retrieve metrics and are able to inspect other component, but even implements the code that can be easily and efficiently monitored and ispected from the outside.

In kubernetes cluster is pretty common to provide every microservice of an HTTP rest API used by kubernetes to check health and this imply that 
- code must be implemented
- tasks must be created
- tests must be written
and all to let kubernetes monitor hour microservices.

Another example in microservices architecture is the need to correlate lines of log with a unique ID dedicated to a reqeust chain so that it is possible to follow what happened through the various services and instances. One way to accomplish this is to add an HTTP header when receiving the request at first and then just propagate it through the chain until the end (the completion of the first request): this also imply that non business implementation must be done.

Last one: RESTful.

Even if maybe is not the main reason why peolple use REST - one thing that I found beautiful - is how it is natually suitable to be monitored using *params in the path* and *HTTP method* for specify the action.

With this couple of things it is possible to monitor an API just identifying method and path of the HTTP request.

So thinking about a REST api that handle for example user login we can answer questions like:
- how many people access my website?
- how many attempts are needed to create a new user?

But even more (and here some business comes in play)
- since most of the time people get a *Bad Request* creating an user should we improve our UI to validate the fields better?
- since the average request per minute of the login API is higher in the evening than at lunchtime it still makes sense to do maintenance in the evening? 

Could (or should) all these informations drive business and architectural choices? 

I think that a deep understanding of the mechanisms *effectively* in place in our systems is the basis to make the right choices.

It allows designers to maintain their knowledge about the system even after changes made beyond their control (hotfix).
Furthermore it allows to recursively redesign the system to make it keep it alive and efficient.

Inspection of a system makes it appear as it really is: something alive.

It reveals its real nature and the real form that the initial design finally has taken.

## Appendix I. Semantic dilemma

I tend to avoid the terms architect and architecture and I know that a lot of people would disagree but I have my reasons:
1. In high school, I studied as a surveyor and one thing I know for sure: a mason could very hardly become an architect while a programmer could easily do
1. architecture transmits to me the idea of something static while software has (or should have) the ability to evolve over the time;
1. it seems to me that architecture is being called into question when dealing with components that interact with each other but a lot of problems I face do not deal with that. Is defining the right log pattern an architectural choice? Is the plan for a cloud provider migration an architectural job? 

P.S.: I love how Robert Martin (Uncle Bob) defines itself on twitter: **Software craftman**


## Appendix II. Monitoring and observability are diffrent things

DORA - DevOps Research and Assessment define monitoring and observability

>**Monitoring** is tooling or a technical solution that allows teams to watch and understand the state of their systems. Monitoring is based on gathering predefined sets of metrics or logs
>
>**Observability** is tooling or a technical solution that allows teams to actively debug their system. Observability is based on exploring properties and patterns not defined in advance

The difference is made by the concept of predefined and not defined in advance.

Let's try with an exemple.
Let's say we have a log file updated in real time
1. check every 5 minutes for the number of errors to evaluate system health is **monitoring**
1. read the logs to identify the root cause of an increase of errors is **observability**

It the previous example the log file is just a tool that offer both capabilities if and only if lines wirtten report the right information so is alway again design (does the log pattern deserve to be a design choice? yes, it is)