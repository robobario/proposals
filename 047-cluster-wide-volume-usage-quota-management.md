# Cluster Wide Volume Usage Quota Management

Extend the static quota mechanism to throttle message production in a broker if any active broker in the cluster is running out of disk.

- [Current situation](#current-situation)
- [Motivation](#motivation)
- [Proposal](#proposal)
  - [Caveat - Kraft Disk Usage](#caveat---kraft-disk-usage)
  - [High Level Changes](#high-level-changes)
    - [Volume](#volume)
    - [Quota Source](#quota-source)
    - [Throttle Factor](#throttle-factor)
    - [Throttle Factor Source](#throttle-factor-source)
    - [Cluster Volume Source](#cluster-volume-source)
      - [Admin Client Configuration](#admin-client-configuration)
      - [Limit Type Configuration](#limit-type-configuration)
    - [Throttle Factor Fallback](#throttle-factor-fallback)
      - [Throttle Factor Validity Duration](#throttle-factor-validity-duration)
- [Configuration Summary](#configuration-summary)
  - [Limit Configuration](#limit-configuration)
- [Metrics](#metrics)
- [Rejected Alternatives](#rejected-alternatives)
- [Affected Projects](#affected-projects)
- [Compatibility](#compatibility)

## Current situation

The [kafka-static-quota-plugin](https://github.com/strimzi/kafka-quotas-plugin) applies byte-rate limits on client
connections to individual brokers (thresholds derived from storage limits), slowing producers down when used storage is
over the soft limit (but below the hard limit) and effectively pausing publication when reaching or exceeding the hard
limit. This largely prevents out of disk scenarios when partition replicas are replicated to all brokers thus there is largely
similar disk usage between all brokers. Assuming all the brokers have similar disk usage levels, they will all apply
rate limits at similar times and levels, effectively giving cluster-wide out-of-storage protection.

This however provides limited protection to clusters with un-even distribution of partition replicas and thus storage
usage. Additionally, it ties the plug-in directly to storage usage, there are other factors which users may wish to
respond to.

As clusters scale up the likelihood of even partition replica distribution drops. When partition replica are not evenly
distributed it is possible for replication from broker `1` to broker `2` will cause broker `2` to consume all available
storage without triggering throttling of clients, as broker `1` disk usage remains acceptable.

Addressing the effects of uneven partition replicas distribution sounds like it should come
under [KIP-73](https://cwiki.apache.org/confluence/display/KAFKA/KIP-73+Replication+Quotas). Unfortunately replication
quotas are designed to manage the additional network load of migrating replicas between brokers, which does not address
client publication leading to out of disk conditions through replication. As replication throttling is configured in
terms of `TopicPartitions` the quota plug-in would need a mechanism to translate a logDir into the set of
TopicPartitions replicated to that directory. Assuming it generated the appropriate throttle configuration this could
lead to unpredictable latency spikes for producers configured with `acks = 1` or `acks = all` as the replication from
partition leader to follower may be delayed due to the throttle.

Currently, the kafka-quotas-plugin considers the total quantity of storage attached to a single broker and how much of
that is used ( see [issue#2](https://github.com/strimzi/kafka-quotas-plugin/issues/2)) when considering whether to apply
throttling to clients. This is problematic with respect to handling disk failure for Just a Bunch Of Disks (JBOD)
deployments [(KIP-112)](https://cwiki.apache.org/confluence/display/KAFKA/KIP-112%3A+Handle+disk+failure+for+JBOD) as
the broker will take partitions offline when the volume they are stored on runs out of space. Which in the case of
unbalanced usage between volumes can lead to a volume running out of storage without throttling being applied.

Throttling down message production on all broker nodes will protect the cluster from running out of disk due to replication.

## Motivation

Users need better protection against running out of disk on any of their nodes as it can degrade the service
in unpredictable ways (corrupted segment logs) which could impede the interventions required to recover.

## Proposal

1. Extend the kafka-quotas-plugin so that we observe the disk usage of all brokers in the cluster.
2. [Remove](#compatibility) the existing local disk observations, configuration and metrics.
3. Add new [limit types](#limit-type-configuration), so that it is explicit what is being limited and better support heterogeneous disks
   1. throttle if available bytes less than threshold on any volume
   2. throttle if available ratio less than threshold on any volume
4. Introduce extension points in the quotas-plugin to support pluggable sources of quotas and throttle factors
5. Introduce a [throttle factor fallback](#throttle-factor-fallback) and [throttle factor validity duration](#throttle-factor-validity-duration) in case we cannot retrieve the cluster state
6. Strimzi Cluster Operator becomes responsible for configuring the admin client connection properties of the plugin

Every broker will make its own independent throttling decision based on knowledge of the volumes on all active broker nodes.
The brokers should all operate on a similar view of the cluster state and make a deterministic decision. If a broker detects that
any volume in the cluster is becoming too full it will throttle production of messages. The Kafka quota API isn't rich 
enough to do anything smarter about only throttling writes to the brokers running out of space, so we fence the whole cluster.

### Caveat - KRaft Disk Usage

This proposal will only help prevent running out-of-disk caused by topic data. It will not prevent disks being exhausted
due to writes to the upcoming KRaft metadata log.

Currently, when using separate controller-only nodes*, those nodes are not described by `Admin#describeCluster` and we cannot use 
`Admin#describeLogDirs` against the controllers. So the disk usage of the volume could be invisible. We should not
enable this plugin on a controller-only node.

When using controller+broker mode*, by default the metadata log is kept in the first `log.dirs` directory but could be 
configured to a custom location, potentially on its own volume. If it was put on a separate volume it would not be
described in the describeLogDirs response, nor contribute to the volume usage of an existing log dir.

So in some cases with controller+broker mode it would afford some protection, as growth of the metadata log dir could
cause throttling of topic writes (because they are on the same volume as another log dir).

Even if we had reliable insight into the disk usage (by extending the Admin APIs for example), of the volume the metadata dir 
resides on, we cannot take effective measures from the quota plugin. Potentially you could use an Authorization
plugin instead and block metadata-generating operations.

*tested with Kafka 3.3.1

### High level changes

To better support alternative sources for managing quotas and how they interact with volume usage this proposal
separates some existing concepts and separates their responsibilities. We envisage the architecture looking like
this ![Component interactions](images/047-quota-plugin-interactions.png)

#### Quota Source

We propose adding a Quota Source concept to the plugin to provide an abstraction through which where we could provide
external sources of quotas, like pull them from an external system.

#### Throttle Factor

We introduce the concept of throttle factor. This is a factor in the range (0.0, 1.0) that is applied to a quota to
calculate the final byte rate limit. 

This is a concept that was already implicit in the current calculations, but we want to name it so that we can use it in 
metric names and provide an extension point in case we want to externalise the Throttle Factor.

#### Throttle Factor Source

We propose adding a Throttle Factor Source concept to the plugin to provide an abstraction through which we could
provide alternative sources of throttle factors, such as pull the factor from a single decision-making broker.

#### Volume

The storage quotas operate on observations about a **Volume** with these characteristics:
1. `logDir`: the path of the logdir
2. `brokerId`
3. `totalBytes`
4. `availableBytes`
5. `usedBytes` (totalBytes - availableBytes)
6. `availableRatio` (availableBytes/totalBytes)

#### Cluster Volume Source

With the introduction of [KIP-827](https://cwiki.apache.org/confluence/display/KAFKA/KIP-827%3A+Expose+log+dirs+total+and+usable+space+via+Kafka+API) in
Kafka 3.3 we can now obtain the total and usable (available) bytes per log dir as part of the DescribeLogDirsResponse.

Note: if a single disk contains multiple log dirs, it will be described multiple times through the Kafka APIs. This
repetition is acceptable as our new limit types will be applied per-volume, so redundant volume descriptions don't
impact the outcome.

The Cluster Volume Source will use this API to discover volume information for the whole cluster. We intend to continue
using `client.quota.callback.static.storage.check.interval` to configure the interval between polling the
cluster state (default is 1 minute). The performance cost of this API was [discussed](https://lists.apache.org/thread/11zyqqnyg1wgf4jdo6pvn7hn51g3vf8r) 
upstream as part of the KIP, which should be low cost.

##### Admin Client Configuration

To obtain log dir descriptions through the Admin API we need to construct an admin client.

We require the admin client bootstrap to be configured using `client.quota.callback.static.kafka.admin.bootstrap.servers`

Additional admin client configuration can be passed using the form `client.quota.callback.static.kafka.admin.${configuration}`.
For example: `client.quota.callback.static.kafka.admin.security.protocol=SSL`

The Strimzi Cluster Operator would be responsible for configuring the admin client to connect to the
replication listener of the broker.

##### Limit Type Configuration

The existing limit types applied to the aggregate used bytes of all volumes will be meaningless now that we are
working with a description of all volumes in the cluster. Nodes enter and exit the active set as part of normal operation
and so the aggregate used bytes will vary.

Instead, we propose introducing new limit types applied on a per-volume basis. Meaning that there is a single value
for each limit which we test against each volume. i.e. we do not support limiting based on a specific volume. When
that limit is exceeded we effectively stop messages being produced (by calculating a Throttle Factor of 0.0).

We will not support the pre-existing concept of a **soft limit**. In the existing code the soft limit is where
you begin slowing down the rate of message production. As you approach the hard limit the production quota is reduced
linearly. Then at the hard limit message production is stopped. We decided this additional signal that something is
failing is not a core feature of this plugin. We expect users to have their own disk usage monitoring to prompt
intervention before this plugin shuts down all message production.

We require one limit to be configured.

Defining multiple limit types would be an invalid state.

The limits we want are:
1. throttle if [availableBytes](#volume) is less-than-or-equal-to some threshold for any volume
2. throttle if the [availableRatio](#volume) is less-than-or-equal-to some threshold for any volume

For example, to configure a limit when availableBytes is below 1GB:
- `client.quota.callback.static.storage.per.volume.limit.min.available.bytes=1000000000`

Another example, to configure a limit when availableRatio is below 0.01 (1%):
- `client.quota.callback.static.storage.per.volume.limit.min.available.ratio=0.01`

Defining more than one limit is **invalid**, for example the below is disallowed:
- `client.quota.callback.static.storage.per.volume.limit.min.available.ratio=0.05`
- `client.quota.callback.static.storage.per.volume.limit.min.available.bytes=5000000000`

#### Throttle Factor Fallback

We are going to depend on using the admin client to get volume information. This brings all the baggage of making a
connection and dealing with possible failures. Also, we can potentially have an inconsistent view of the world between
determining the active broker set and asking for a description of all log dirs (because we make two independent API calls)
. So we need to react somehow to cases where we cannot get the volume data for all active brokers.

Example inconsistent state:
1. we call `describeCluster` and get a response that says broker 1 and 2 are active
2. broker 2 shuts down cleanly and is removed from the active set
3. we call `describeLogDirs( brokerIds = [1,2] )` and only receive descriptions for logdirs on broker 1

We propose introducing a configurable **throttle factor fallback** to be applied in situations where we don't have enough
information to act. With a default value of 1.0 to optimistically allow all the quota to be used. This would allow
users to opt in to more pessimistic behaviour like using a factor of 0.0 to prevent writes when we are in an unknown
state. (note: we will also use the [Throttle Factor Validity Duration](#throttle-factor-validity-duration) to control
when the fallback is applied)

Throttle factor fallback can be in the range (0.0, 1.0)

Example configuration: `client.quota.callback.static.throttle.factor.fallback=0.0`

##### Throttle Factor Validity Duration

To make the throttle factor logic more resilient in the face of transient failures we want to continue
using a previously calculated valid throttle factor for some duration, rather than immediately using the 
throttle factor fallback after a single failure.

So we will enable the user to specify how long a valid throttle factor can be applied for with
`client.quota.callback.static.throttle.factor.validity.duration`. Where the value is an ISO8601 duration.
Default value "PT5M", 5 minutes.

For example if validity duration is 2 minutes and throttle factor fallback is 0.0, then we could expect this behaviour:

1. throttle factor recalculation at minute 0 succeeded, calculated throttle factor 1.0 and use it
2. throttle factor recalculation at minute 1 failed, continue using throttle factor 1.0
3. throttle factor recalculation at minute 2 failed, continue using throttle factor 1.0
4. throttle factor recalculation at minute 3 failed, use throttle factor fallback of 0.0
5. throttle factor recalculation at minute 4 succeeded, calculated throttle factor 1.0 and use it

## Configuration Summary

|                                                                | type   | default | valid values     |                                                                                                               |
|----------------------------------------------------------------|--------|---------|------------------|---------------------------------------------------------------------------------------------------------------|
| client.quota.callback.static.storage.check.interval            | string | PT1M    | ISO8601 duration | the interval between checking the storage usage and recalculating the quota, set to PT0S to disable the plugin |
| client.quota.callback.static.kafka.admin.bootstrap.servers     | string |         |                  | required bootstrap.servers for [admin client](#admin-client-configuration) used to get cluster data           |
| client.quota.callback.static.kafka.admin.*                     | ?      |         |                  | optionally users can configure arbitrary properties of the [admin client config](#admin-client-configuration) |
| client.quota.callback.static.throttle.factor.fallback          | double | 1.0     | (0.0, 1.0)       | sets the [Throttle Factor Fallback](#throttle-factor-fallback)                                                |
| client.quota.callback.static.throttle.factor.validity.duration | string | PT5M    | ISO8601 duration | sets the [Throttle Factor Validity Duration](#throttle-factor-validity-duration)                              |

### Limit Configuration
The user must configure a single [limit type](#limit-type-configuration)

|                                                                           | type   | default | valid values |                                                         |
|---------------------------------------------------------------------------|--------|---------|--------------|---------------------------------------------------------|
| client.quota.callback.static.storage.per.volume.limit.min.available.bytes | long   | null    | (0, ...)     | stop message production if availableBytes <= this value |
| client.quota.callback.static.storage.per.volume.limit.min.available.ratio | double | null    | (0.0, 1.0)   | stop message production if availableRatio <= this value |

## Metrics

1. Add a `io.strimzi.kafka.quotas:type=LocalThrottleFactor,name=ThrottleFactor` gauge. This would emit the most recently calculated throttle factor.
2. Add a `io.strimzi.kafka.quotas:type=LocalThrottleFactor,name=FallbackThrottleFactorApplied` counter.
3. Add a `io.strimzi.kafka.quotas:type=LocalThrottleFactor,name=LimitViolated` counter. Incremented when a volume violates a limit.
4. Add a `io.strimzi.kafka.quotas:type=ClusterVolumeSouce,name=ActiveBrokers` gauge. To expose how many brokers this node considered most recently
5. Add a `io.strimzi.kafka.quotas:type=ClusterVolumeSouce,name=ActiveLogDirs` gauge. To expose how many log dirs this node considered most recently

## Rejected Alternatives

### Using an Authorizer Plugin

Using an [authorizer plugin](https://kafka.apache.org/33/javadoc/org/apache/kafka/server/authorizer/Authorizer.html) 
to deny operations is another alternative. It would allow finer-grained control but would potentially be surprising 
to Kafka clients to see unexpected authorization denials. Also, the API would need careful attention to performance 
as it's called per-operation. 

Enabling it to chain with another user-configured authorizer is another complexity. The plugin would have to be made 
aware of the delegate class, instantiate it and drive its lifecycle.

### Using a Kafka topic to distribute metrics. 

See the original proposal PR [#51](https://github.com/strimzi/proposals/pull/51). We considered using a Kafka topic to
push volume usage metrics out to all the brokers. KIP-827 made this redundant.

### Using JMX metrics

Using JMX metrics directly would require a web of connections between brokers and the exposing of the JMX port to the
rest of the cluster. Using JMX is also problematic for tracking state across restarts of brokers as each broker would
lose state across restarts and thus lose track of any broker which is temporarily offline.

### External metrics store

Would require the following:

- The quota plugin understands the API of the external metrics system
- A metrics system endpoint exposed to the broker for consuming metrics
- A predictable and consistent naming convention

It would also make the deployment of an external metrics store a requirement for the kafka-static-quota-plugin to function.

### KIP-73

KIP-73 is designed to protect client performance while cluster re-balancing exercises are taking place by limiting the
bandwidth available to the replication traffic. This is not suitable for use in preventing out of storage issues as the
bandwidth limit is configured as part of the partition re-assignment operation. As it applies a bandwidth limit it is
configured in `units per second` which is problematic for the quota plugin to determine a sensible value for as it
should really be related to the expected rate at which data is purged from the tail of the partitions on the volume in
question. KIP-73 bandwidth limits are only applied to a specific set of `partition` & `replica` pairs which would
require the ability for the plugin to resolve the required pairs.

## Affected projects

* strimzi/kafka-quota-plugin
* strimzi/kafka-cluster-operator

## Compatibility

We will break backwards compatibility with Kafka brokers older than 3.3.0, likely Strimzi will have dropped support for
3.2.x before this proposal is implemented, so we will not support brokers without KIP-827 implemented.

We will remove the existing features which use local volume information, as well as existing soft/hard limits that are
applied to the aggregate used bytes across all volumes.

We will remove these properties:
- `client.quota.callback.static.storage.hard`
- `client.quota.callback.static.storage.soft`
- `client.quota.callback.static.storage.check-interval`

We will remove these metrics:
- `io.strimzi.kafka.quotas:type=StorageChecker,name=TotalStorageUsedBytes`
- `io.strimzi.kafka.quotas:type=StorageChecker,name=SoftLimitBytes`
- `io.strimzi.kafka.quotas:type=StorageChecker,name=HardLimitBytes`
