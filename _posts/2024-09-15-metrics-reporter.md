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
in Strimzi Proposal #64. It offers a way to export Apache Kafka metrics directly to Prometheus without relying solely on
JMX (Java Management Extensions).

This reporter is useful for anyone using Kafka who wants a seamless integration with Prometheus, a powerful monitoring
and alerting toolkit widely used in cloud-native environments.

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

Cruise Control's optimization approach allows it to perform several different operations linked to balancing cluster
load.
As well as rebalancing a whole cluster, Cruise Control also has the ability to do this balancing automatically when
adding and removing brokers or changing topic replica values.
Cruise Control can also supply information on cluster load, can monitor the cluster for anomalies and even perform (
limited) automated self-healing actions.

This is an impressive list of features and to enact them Cruise Control has the ability to manipulate the state and
configuration of the Kafka cluster it is monitoring.
Something else that likes to manipulate the state and configuration of your Kafka cluster is the Strimzi Cluster
Operator.
This means that combining these two systems is not a straight forward task and we are not supporting all the Cruise
Control features in this initial release.
For now, we are supporting the whole-cluster rebalance functionality. The operator will handle installing the metrics
reporters in the brokers and setting up the Cruise Control server for you.
You can then post a Kubernetes Custom Resource (described in the [guide below](#rebalancing-your-cluster)) to handle the
rebalance for you, all in a Kubernetes native, Cluster Operator friendly way.
We will continue to improve the Cruise Control integration and add more functionality over then next few Strimzi
releases.


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

Now Metrics repoter is deployed it will start pulling in metrics from the topics created by the metrics reporters in each
broker.

### Metrics Reporter configuration

The deployment above uses the default Cruise Control configurations.
This will use the upstream project's optimization goals list priorities (with a few exceptions) and their hard/soft goal
configuration.
If you are feeling adventurous, you can configure your own optimization priorities.
You can do this via three main configuration options at the Cruise Control deployment level (there are also rebalance
request level options as well - which are covered in the [sections below](#custom-rebalace-goals).

- `goals`: This configures the _master_ goals list. Basically, this is the list of goals that are allowed to be used in
  any of Cruise Control's functions. You would use this primarily to limit the optimization goals that that users of
  your cluster's Cruise Control server can use. For example if, for some reason, you don't want any cluster user to use
  the preferred leader election goal, you can define a `goals` list that does not include it.
- `hard.goals`: This list of goals will define which goals in the master goals list are considered _hard_ and cannot be
  violated in any of the optimization functions of Cruise Control. The longer this list, the less likely it is that
  Cruise Control will be able to find a viable optimization proposal, so if you do customise this, try to make it as
  short as possible.
- `default.goals`: The default goals list is the most important of these configurations. By default, every 15 mins,
  Cruise Control will use the current state of your Kafka cluster to generate a _cached optimization proposal_ using the
  configured `default.goals` list. This means you should configure the `default.goals` list to be the set of goals that
  you will most often want your cluster to meet. That way, when you ask for a rebalance, a optimization proposal should
  be already prepared.

All these configuration options can be set under `Kafka.spec.cruiseControl.config`.
Please consult
the [Strimzi documentation](https://strimzi.io/docs/operators/latest/using.html#con-optimization-goals-str) for more
detailed information on the various goal configurations.

### Getting an optimization proposal

Now you have Cruise Control deployed you can ask it to generate a optimization proposal.
To do this you will need to apply a `KafkaRebalance` custom resource (the definition for this is installed when you
install Strimzi).
A basic `KafkaRebalance` is shown below:

```yaml
apiVersion: kafka.strimzi.io/v1alpha1
kind: KafkaRebalance
metadata:
  name: my-rebalance
  labels:
    strimzi.io/cluster: my-cluster
spec: { }
```

And you can deploy this like any other resource:

```bash
kubectl apply -f kafka-rebalance.yaml -n myproject
```

This rebalance has an empty `spec` field and so will use the default rebalance settings.
This means it will use the `default.goals` (mentioned above) and will return the most recent _cached optimization
proposal_.
Once you apply this resource to your Kafka cluster the Cluster Operator will issue the relevant requests to Cruise
Control to fetch the optimization proposal.
The Cluster Operator will keep the status of your `KafkaRebalance` updated with the progress from Cruise Control.
You can check this using:

```bash
$ kubectl describe kafkarebalance my-rebalance -n myproject     
```

This will show an output like that shown below:

```yaml
Name: my-rebalance
Namespace: myproject
Labels: strimzi.io/cluster=my-cluster
Annotations:
  API Version: kafka.strimzi.io/v1alpha1
Kind: KafkaRebalance
Metadata:
# ...
Status:
  Conditions:
    Last Transition Time: 2020-06-04T14:36:11.900Z
    Status: ProposalReady
    Type: State
  Observed Generation: 1
  Optimization Result:
    Data To Move MB: 0
    Excluded Brokers For Leadership:
    Excluded Brokers For Replica Move:
    Excluded Topics:
    Intra Broker Data To Move MB: 12
    Monitored Partitions Percentage: 100
    Num Intra Broker Replica Movements: 0
    Num Leader Movements: 24
    Num Replica Movements: 55
    On Demand Balancedness Score After: 82.91290759174306
    On Demand Balancedness Score Before: 78.01176356230222
    Recent Windows: 5
  Session Id: a4f833bd-2055-4213-bfdd-ad21f95bf184
```

The key parts to pay attention to here are the `Status.Conditions.Status` field, which shows the progress of your
rebalance request (if the proposal is not ready yet you may see `ProposalPending` as the `Status`) and
the `Status.Optimization Result` which (once the proposal is ready) will show a summary of the optimization proposal.
The meaning of each of these values is described in
the [Strimzi documentation](https://strimzi.io/docs/operators/latest/using.html#con-optimization-proposals-str),
essentially this summary shows you the scope of the changes that Cruise Control has suggested and from this you can
figure out what kind of impact these changes may have on your cluster whilst they are being applied.

#### Interpreting an optimization proposal

In the case shown above, Cruise Control has suggested 55 partition replicas (totalling 12MB of data) should be move
between separate brokers (_inter_-broker) in the cluster.
This will involve moving data across the network, but 12MB is not a lot so this should not have too much of an impact.
If this number was very high you may want to time this rebalance for a period of less load, as moving large amounts of
data could effect the performance of the cluster as a whole.

There are no _intra_ (between disks on the same broker) movements proposed.
These kind of movements only involve movement between disks so have no network impact, but may effect the performance
the particular brokers whose disks are exchanging data.

Finally, this proposal is suggesting 24 leader replica changes.
This is the cheapest change that can be applied as it only involves a configuration change in ZooKeeper.

### Starting a cluster rebalance

If you are happy with the proposal then you can apply an `approve` annotation to the `KafkaRebalance` resource to tell
Cruise Control to start applying the changes:

```bash
$ kubectl annotate kafkarebalance my-rebalance strimzi.io/rebalance=approve -n myproject
```

It is important to note that if you wait a long time between asking for a proposal and approving it, the underlying
cluster might be in a very different state to the time when you requested the proposal.
In this case it is a good idea to _refresh_ the proposal, just to make sure it is up-to-date. You can do this by
applying the `refresh` annotation to the `KafkaRebalance` resource:

```bash
$ kubectl annotate kafkarebalance my-rebalance strimzi.io/rebalance=refresh -n myproject
```

Once you apply an `approve` annotation Cruise Control will begin enacting the changes from that proposal.
The Cluster Operator will keep the status of the `KafkaRebalance` resource up-to-date with the progress of the rebalance
whilst it is ongoing.
You can check this by running:

```bash
$ kubectl describe kafkarebalance my-rebalance -n myproject
```

Whilst the rebalance is in progress the Status will show `Rebalancing` and once it is finished it will show `Ready`.

```yaml
Name: my-rebalance
Namespace: myproject
Labels: strimzi.io/cluster=my-cluster
Annotations:
  API Version: kafka.strimzi.io/v1alpha1
Kind: KafkaRebalance
Metadata:
# ...
Status:
  Conditions:
    Last Transition Time: 2020-06-04T14:36:11.900Z
    Status: Rebalancing
    Type: State
# ...
```

When you ask for a rebalance with custom goals, Cruise Control will first check if the current cached optimization
proposal already meets the list of custom goals you specified.
If so, you will get your proposal back quickly.
If not, you may have to wait a while for Cruise Control to perform the calculations.

One thing to watch out for here, is that your list of custom goals contains all the configured `hard.goals` (the
default `hard.goals` are listed
in [the docs](https://strimzi.io/docs/operators/latest/using.html#con-optimization-goals-str)).
If it doesn't then Cruise Control will throw an error, after all why would you say these goals are _hard_ (really
important) and then ignore them?
But, if you really do only care about a few goals for this one off rebalance, you can tell Cruise Control to relax by
adding `skipHardGoalCheck: true` to the `KafkaRebalance` resource.

### Stopping a rebalance

Rebalances can take a long time and may impact the performance of your cluster, therefore you may want to stop a
rebalance if that is causing an issue.
It is also possible that you may approve an optimization proposal that you later realise was not what you wanted.
In either case, you need to be able to stop an ongoing rebalance.
To do this you can apply the `stop` annotation to the `KafkaRebalance` resource at any time:

```bash
$ kubectl annotate kafkarebalance my-rebalance strimzi.io/rebalance=stop -n myproject
```

## Conclusion

So, we have shown you what Cruise Control is, why you might want to use, what parts of it Strimzi supports and how to
configure and use them.
We are looking forward to people trying out the Cruise Control support and we are always open to questions and
suggestions, so please [reach out to us](https://strimzi.io/community/).

We are going to be adding more Cruise Control functionality in future so keep an eye on this blog for more updates.

Happy rebalancing!
