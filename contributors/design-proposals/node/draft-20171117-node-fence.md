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

## Proposal

The core solution will center around a single Pod containing two
controllers:

1. a NodeStateObserver which uses the Kubernetes API to watch for
   nodes that enter failed states, and

1. a FenceController that ensures the fencing happens.

Upon identifying a node entering a failed state, the NodeStateObserver
will create a NodeFencingRequest which the FenceController will be
made aware of via the informer pattern.

Once the FC receives the NodeFencingRequest it will be vetted against
suitability criteria:

- Grace periods: it may be desirable to give nodes a chance to recover

- De-duplication logic: avoiding redundant fencing events

- Quorum and fencing-storm detection: using heuristics to avoid a
  healthy majority being fenced by a disconnected minority

- Workload suitability: Currently only nodes running statefulset
  members require fencing to continue recovery.

If fencing is deemed appropriate, the FenceController will spawn one
or more NodeFenceExecution Jobs, depending on the fencing
configuration.  Each Job will co-ordinate the execution of one or more
NodeFenceAction Pods, each of which represents a single call to a
fencing device.

The Pods’ commands will be the name of a fencing agent and any
required parameters.  Agents come from the set provided by ClusterLabs
fence-agents project or conform to the same API, communicate their
result through their exit codes, and log to stderr to assist trigaing.

The allowance for multiple Jobs is necessary to support boolean-OR
semantics when evaluating alternatives.  For example, the
configuration may call for “(network fencing AND disk fencing) OR
power fencing” in order to prefer network and disk fencing but, if
those fail, fallback to power fencing.

The FenceController will use again the informer API to receive
notification of Job completion, recording the overall result as well
as that of each Pod and any output (to aid subsequent triage).  The
result of each Job will determine which, if any, subsequent Jobs will
be launched by the FenceController.

The last step in handling all NodeFencingRequests is the creation of a
Job that deletes the appropriate Pods from the the Kubernetes API
server, after which the request is updated with a result and
eventually aged out by the FenceController.

## User Experience

The loss of a worker node should be transparent to user's of
StatefulSets.  Recovery time for affected Pods should be bounded and
short, allowing scale up/down events to proceed as normal afterwards.

### Use Cases

1. In a bare metal Kubernetes deployment, StatefulSets should not
   require admin intervention in order to restore capacity when a
   member Pod was active on a lost worker

1. In a bare metal Kubernetes deployment, StatefulSets should not
   require admin intervention in order to scale up or down when a
   member Pod was active on a lost worker

1. In a Cloud Kubernetes deployment, StatefulSets without an
   autoscaler should not require admin intervention in order to scale
   up or down when a member Pod was active on a lost worker

In other words, the failure of a worker node should not represent a
single point of failure for StatefulSets.

## Implementation


### Client/Server Backwards/Forwards compatibility

<define behavior when using a kubectl client with an older or newer version of the apiserver (+-1 version)>

## Alternatives considered

1. Create a new Cloud Provider allowing the autoscaler to function for
   bare metal deployments.
   
   This was considered however the existing APIs are load balancer
   centric and hard to map to the concept of powering on and off nodes.
   
   If the Cloud Provider API evolves in a compatible direction, it
   might be advisable to persue a Bare Metal provider and have it be
   responsible for much of the fencing configuration.

1. A solution that focused exclusively on power fencing.
   
   While this would dramatically simplify the configuration required,
   many admins see power fencing as a last resort and would prefer
   less destructive way to isolate a misbehaving node, such as network
   and/or disk fencing.
   
   We also see a desire from admins to use tools such as `kdump` to
   obtain additional diagnostics, when possible, prior to powering off
   the node.

1. Combining the detection and execution logic in a single controller.
   
   While it would be possible to combine this functionality into a
   single controller, the separation of concerns makes it possible
   that other entities could trigger fencing in the future.

1. Attaching fencing configuration to nodes.
   
   While it is tempting to add details on how to fence a node to the
   kubernetes Node objects, this scales poorly from a maintenance
   perspective, preventing nodes from sharing common methods (such as
   `kdump`).
   
   This is especially true for cloud deployments where all nodes are
   controlled with the same credentials. However, even on bare metal
   the only point of differention is often the the IP addresses of the
   IPMI device, or the port number for a network switch, and it would
   be advantageous to manage the rest in one place.


