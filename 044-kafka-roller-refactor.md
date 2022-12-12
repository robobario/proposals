# Kafka Availability During Reconfiguration

Strimzi aims to keep kafka available to clients when it is restarting/reconfiguring the cluster. We do not want to
block producers from producing by dropping below min ISR for a topic. We want to avoid degrading a cluster
further if nodes are stuck in pending before we start the roll. We do not want to shut down all brokers containing
replicas of a specific topic-partition.

With the introduction of KRaft we have some new requirements coming soon. We want to consider the state of
the quorum, preferring to keep the quorum leader alive while rolling the other controllers. We may need
to know more state that just the readiness exposed by kubernetes, like whether the broker is in log recovery.
There may be differences in behaviour if we are using combined broker+controller or if they are separate.

These requirements push us towards building logic that can take the state of the whole cluster into account.
The current kafka rolling logic has evolved from a node-by-node algorithm and retrofitting these "holistic"
decisions onto it is painful.

We propose introducing a re-designed Kafka Roller implementation and deprecating the existing implemetation.

## Current situation

Currently Strimzi uses a KafkaRoller class with responsibilities including:
1. restart kafka brokers if required
2. dynamically reconfigure brokers if required
3. ensure the controller broker is restarted last
4. detect stuck pods and restart them
5. avoid breaking min-ISR availability during the roll
6. await readiness of pods that aren't ready at the start of the roll
7. manages an executor/thread per roll (used as a queue to defer restart)

The class has several disadvantages when it comes to adding futher "holistic" logic about which node to roll:

1. the current queue-based solution is built on top of considering the pods one-by-one and deferring processing
   if required. The fact the algorithm is based on cycling through the nodes works against cluster-aware decision
    making.
2. Observations of the cluster state are spread across a lot of methods.
3. It uses a set of Exception classes to control logic flow and deferring which leads to a lot of catching and
   rethrowing. Adding more dimensions to this tangle of exceptions is unsustainable.
4. It's big! The class handles queuing, knows how to observe/classify interesting cluster states, restart pods,
   reconfigure pods.
5. Testing is somewhat painful since the queuing/observation/action is so bound together, requiring integration
   tests to check all it's behaviour.

## Motivation

The motivation is to support adding further dimensions to the Kafka Rolling logic, change it to be more oriented
around decisions that account for the whole-cluster's state and decompose the responsibilities to make it clearer
what the roller is doing.

## Proposal

### High Level Proposal
1. We extract an interface from the existing KafkaRoller and mark the old implementation deprecated
2. We create a new implemention of KafkaRoller that does nothing.
3. We call the new KafkaRoller and then redundantly call the old KafkaRoller
4. We start by making the new roller responsible for awaiting pods that are not ready at the start of roll. This
   includes making it wait for brokers that are in log recovery.

From there we could add features to the new roller, like making it responsible for dynamic reconfiguration.
The existing implementation would continue handling the restarts. Eventually once we have made the new
roller handle all the logic the old KafkaRoller should do nothing and we could delete it.

### Risks
We want to avoid any situation where the new and old implementation both sit around until their own timeouts,
doubling the time taking.

### Kafka Roller Architecture

The new Roller would follow the prototype by Tom Bentley and Shubham Rawat. The responsibilities of the Roller
would be split roughly into:

1. control - operates a loop which
  1. observes - collect facts about the state of the whole cluster
  2. classifies - impose our own logical view of what state we are in (this pod is stuck, this pod is in state X)
  3. acts - decide on the next action to take and execute it

Noticable differences are:
1. We will have decomposed responsibilities into classes we can test independently
2. We will have a clearer decision point, rather than nodes deferring their own restart
3. We can avoid using nested exception handling for control flow.

## Affected/not affected projects

strimzi/kafka-cluster-operator

## Compatibility

This should be invisible to users as the new roller should accumulate responsibilities until the old implementation
can be deleted.

## Rejected alternatives

### Refactoring the existing Kafka Roller

We think it would be a harder path to try and internally move parts of the KafkaRoller to use a new algorithm.
