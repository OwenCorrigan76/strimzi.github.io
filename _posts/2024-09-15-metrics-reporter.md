---
layout: post
title: "Scraping Prometheues Metrics with Strimzi Metrics Reporter"
date: 2024-09-15
author: Owen Corrigan
---

As organizations increasingly rely on Apache Kafka® for building real-time data pipelines and streaming applications,
monitoring becomes crucial to ensure the health and performance of Kafka clusters. While Kafka provides built-in support
for exposing metrics via JMX, using Prometheus as a metrics monitoring solution is becoming more common due to its
popularity in the cloud-native ecosystem.

[Strimzi Metrics Reporter](https://github.com/strimzi/metrics-reporter) is a project aimed at bridging this gap by
providing a direct way to export Kafka metrics to Prometheus. This blog post will introduce the Strimzi Prometheus
Metrics Reporter, discuss its installation and configuration, and guide you on how to use it effectively to monitor your
Kafka clusters.

## Metrics Reporter overview

The Strimzi Metrics Reporter is an early-access project that implements a metrics reporter for Prometheus, as proposed
in this [Strimzi Proposal](https://github.com/strimzi/proposals/blob/main/064-prometheus-metrics-reporter.md). It offers a way to export Apache Kafka metrics directly to Prometheus without relying solely on
JMX (Java Management Extensions).

This reporter is useful for anyone using Kafka who wants a seamless integration with Prometheus, a powerful monitoring
and alerting toolkit widely used in cloud-native environments.
The reporters will also export JVM metrics similar to the ones exported by jmx_exporter. These are provided by the [JVM instrumentation package](https://github.com/prometheus/client_java/tree/main/prometheus-metrics-instrumentation-jvm) from the Prometheus Java client.

## Key Features
* Direct Prometheus Integration: Exposes Kafka metrics directly to Prometheus through an HTTP endpoint.
* Pluggable Reporter: Implements Kafka's pluggable reporter interface to export metrics.
* Configurable Metrics Collection: Allows users to specify which metrics should be collected using a flexible allowlist.

(Maybe take out)
## KafkaPrometheusMetricsReporter
The initialization of the KafkaPrometheusMetricsReporter is handled by Kafka. When Kafka starts up, it looks for any classes that implement the MetricsReporter interface. If it finds any, it creates an instance of each class.
Kafka does not directly communicate with Prometheus. Instead, Kafka exposes its metrics and Prometheus scrapes or pulls this data at regular intervals.
The KafkaPrometheusMetricsReporter class is responsible for exposing Kafka's metrics in a format that Prometheus can understand. This class implements the MetricsReporter interface provided by Kafka, which allows it to collect metrics data from Kafka. KafkaPrometheusMetricsReporter also starts an HTTP server if it's configured to do so. This server exposes the endpoint ( /metrics) that Prometheus can scrape. At runtime as metrics are added or removed in Kafka (for example a topic gets created or deleted),



## Installing Strimzi Prometheus Metrics Reporter

Since the Strimzi Prometheus Metrics Reporter is currently in early access and does not have a formal release yet, you
must build the reporter from the source. Here’s how to install it:

### Here’s how to install it:

Clone the Repository: First, you need to clone the repository and cd into the cloned directory
from [GitHub](https://github.com/strimzi/metrics-reporter):

```bash
git clone https://github.com/strimzi/metrics-reporter
cd strimzi/metrics-reporter
```

Build the Reporter: Use Maven to build the project:

```bash
mvn package
```
After the build is complete, ensure the metrics reporter JAR files located under the target/metrics-reporter-
*/metrics-reporter-*/libs/ directory are in your classpath.



## Strimzi Metrics Reporter support

Metrics Reporter's allows us to display metrics in the form of Kafka Metrics and Yammer metrics.
Metrics Reporter can supply information on cluster by exposing information in the form of labels.

Something else that likes to manipulate the state and configuration of your Kafka cluster is the Strimzi Cluster
Operator.
The Strimzi Cluster Operator is integrated tightly with Metrics reporter and combining these two systems is not a straight forward task. Therefore, we are not supporting all the Metrics Reporter features in this initial release.




### Deploying Metrics Reporter

The first job is to get the metrics reporter installed in your cluster's brokers and get the Metrics Reporter sever
deployment created. To do this add the following to your `Kafka` custom resource:

```yaml
apiVersion: kafka.strimzi.io/v1beta1
kind: Kafka
metadata:
  name: my-cluster
spec:
  # ...
  metricsReporter: { }
```

This will use all the default Metrics reporter settings to deploy the Metrics Reporter server.
If you are adding Metrics Reporter to an existing cluster (instead of starting a fresh cluster) then you will see all your
Kafka broker pods roll so that the metrics reporter can be installed.
Eventually, you should see the Metrics Reporter deployment in your cluster:

```bash
$ kubectl get pods -n myproject   
NAME                                          READY   STATUS    RESTARTS   AGE
my-metrics-reporter-deployment-7f46f588b8-66whz    2/2     Running   0          8m48s
my-cluster-entity-operator-77d8f884d5-jxmht   3/3     Running   0          12m
my-cluster-kafka-0                            2/2     Running   0          9m12s
my-cluster-kafka-1                            2/2     Running   0          10m
my-cluster-kafka-2                            2/2     Running   0          9m46s
my-cluster-zookeeper-0                        1/1     Running   0          13m
my-cluster-zookeeper-1                        1/1     Running   0          13m
my-cluster-zookeeper-2                        1/1     Running   0          13m
strimzi-cluster-operator-54565f8c56-rmdb8     1/1     Running   0          14m
```

Now Metrics Reporter is deployed it will start pulling in metrics from the topics created by the metrics reporters in each broker. .....write this properly

### Metrics Reporter configuration
The deployment above uses the default Metrics configurations.
There are some configurable parts of the Metrics Reporter. These configs can be set in `server.properties`. These include:

-  `prometheus.metrics.reporter.listener`: The listener to expose the metrics in the format http://<HOST>:<PORT>. If the <HOST> part if empty the listener binds to the default interface, if it is set to 0.0.0.0, the listener binds to all interfaces. If the <PORT> part is set to 0, a random port is picked. Default: http://:8080.
- `prometheus.metrics.reporter.listener.enable`: This allows the user to start the listener, or not. The default value is set to 'true'. To disable the listener, set to false.
- `prometheus.metrics.reporter.allowlist`: This is where the user can set the which metrics are allowed to provided. The default setting allows all metrics. Only metrics matching at least one of the patterns in the list will be emitted.
  

All these configuration options can be set under `Kafka.spec.metricsReporter.config`.
Please consult
the [Strimzi documentation](https://strimzi.io/docs/operators/latest/configuring.html) for more
detailed information configurations.



### Starting a cluster with Metrics Reporter

To use Metrics Reporter in your Strimzi Kafka cluster, the following yaml can be deployed:

```bash
$ kubectl apply -f metrics-reporter.yaml -n myproject
```

## Conclusion
So, we have shown you what Metrics Reporter is, why you might want to use, what parts of it Strimzi supports and how to
configure and use them.
We are looking forward to people trying out the Cruise Control support, and we are always open to questions and
suggestions, so please [reach out to us](https://strimzi.io/community/).

Keep an eye on this blog for future updates.

Happy metrics scraping!
