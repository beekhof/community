# Node Fencing

Status: Pending

Version: Alpha

Implementation Owner: TBD

## Motivation

The network between the Kubernetes api server and worker nodes can be
interrupted, resulting in the scheduler being unable to know the
status of pods running there. Since 1.8, Kubernetes has defined an
eviction timeout such that after 5 minutes pods will enter a
termination state in the api server and will try to shutdown
gracefully when possible.

The StatefulSet controller, in order to provide the at-most-one
semantics crucial to the permanent association between its members and
their storage, must wait until the pod's status is known to be stopped
not just in the termination state.

When k8s is deployed on a cloud such as AWS or GCE, the autoscaler
uses the Cloud Provider APIs to remove failed nodes, effectively
bounding the amount of time the node will stay in the “not ready”
state and how long the scheduler will need to wait before it can
safely start the pod elsewhere.  However in the case of bare metal, no
such mechanism exists and recovery will be blocked until an admin
manually intervenes.

This is particularly problematic because all StatefulSet scaling
events (up and down) block until existing failures have been
recovered.  For this reason, it could be considered that failure of a
worker node represents a single point of failure for StatefulSets.

## Proposal

<4-6 description of the proposed solution>

## User Experience

### Use Cases

1. Bare metal deployments using StatefulSets require admin
   intervention in order to restore capacity when a member Pod was
   active on a lost worker

1. Bare metal deployments using StatefulSets require admin
   intervention in order to scale up or down when a member Pod was
   active on a lost worker

1. Cloud deployments using StatefulSets without an autoscaler require
   admin intervention in order to scale up or down when a member Pod
   was active on a lost worker

<in depth description of user experience>

<*include full examples*>

## Implementation

The core solution will center around a single Pod containing two
controllers:

1. a NodeStateDetector which uses the Kubernetes API to watch for nodes that enter failed states, and
1. a FenceController that ensures the fencing happens.

While it would be possible to combine this functionality into a single
controller, the separation of concerns makes it possible that other
entities could trigger fencing in the future.  Combining the two
controllers into a single Pod/process is a standard kibernetes design
pattern.

Upon identifying a node entering a failed state, the NodeStateDetector
(NSD) will create a NodeFencingRequest which the FenceController (FC)
will be made aware of via the informer pattern.

Once the FC receives the NodeFencingRequest it will be vetted against
several criteria:
- Grace periods: it may be desirable to give nodes a chance to recover
- De-duplication logic: avoiding redundant fencing events
- Quorum and fencing-storm detection: using heuristics to avoid a
  healthy majority being fenced by a disconnected minority
- Workload suitability: Currently only nodes running statefulset
  members require fencing to continue recovery.

If fencing is deemed appropriate, the FC will spawn one or more
NodeFenceExecution (NFE) Jobs, depending on the fencing configuration.
Each Job will co-ordinate the execution of one or more NodeFenceAction
(NFA) Pods, each of which represents a single call to a fencing
device.

The NFA Pods’ commands will be the name of a fencing agent and any
required parameters.  Agents come from the set provided by ClusterLabs
fence-agents project or conform to the same API, communicate their
result through their exit codes, and log to stderr to assist trigaing.

The allowance for multiple NFEs is necessary to support boolean-OR
semantics when evaluating alternatives.  For example, the
configuration may call for “(network fencing AND disk fencing) OR
power fencing” in order to prefer network and disk fencing but, if
those fail, fallback to power fencing.

This would be modelled as two NFE Jobs, the first containing network
and disk fencing Pods executed in series, and a second with a single
power fencing Pod.

The FC will receive notification of NFE Job completion via the
informer pattern, recording the overall result as well as that of each
Pod and any output (to aid subsequent triage).  The result of each Job
will determine which, if any, subsequent Jobs will be launched by the
FC.

The last step in handling all NodeFencingRequests is the creation of a
Job that deletes the appropriate Pods from the the Kubernetes API
server, after which the request is updated with a result and
eventually aged out by the FC.

### Client/Server Backwards/Forwards compatibility

<define behavior when using a kubectl client with an older or newer version of the apiserver (+-1 version)>

## Alternatives considered

<short description of alternative solutions to be considered>
