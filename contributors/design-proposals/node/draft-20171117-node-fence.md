# Node Fencing
Status: Pending

Version: Alpha

Implementation Owner: Andrew Beekhof (abeekhof@redhat.com)

Current Repository: https://github.com/beekhof/

## Abstract

The [pod-safety](https://github.com/kubernetes/community/blob/16f88595883a7461010b6708fb0e0bf1b046cf33/contributors/design-proposals/pod-safety.md)
work described possible implementations for at-most-one Pod and storage
semantics, as well as the need for fencing in order to provided bounded,
automated recovery.

Here we describe a native fencing solution for Kubernetes that integrates with the
[Node Problem Detector](https://github.com/kubernetes/node-problem-detector/), 
[Cluster](https://github.com/kubernetes-sigs/cluster-api), and 
[Machine](https://github.com/kubernetes-sigs/cluster-api/blob/master/docs/proposals/machine-api-proposal.md) API efforts.

## Motivation

The Cluster and Machine APIs represent an effort to create an official
interface for natively managing the addition and removal of nodes from a
Kubernetes cluster.  While reprovisioning a node would represent a rather
brutal form of fencing, it does meet the criteria.

In reusing the Machine API, we would be able to benefit from the node
(de)provisioning efforts with which there is a large overlap with our
requirements, configuration, and functionality (which means our parts can be
simplified).  

It would ensure that a custom solution does not **ACT** in conflict with the
Machine API (one deleting a node while the other is trying to create it). By
working with the Machine API effort, fencing would end up better integrated
into Kubernetes, both technically and conceptually, and therefore be more
likely to experience wider adoption.

If we continue to make nodes more self resilient and use the Machine API as a
last resort, rather than as the first step in recovery, it should be possible
to design a solution that provides a consistent experience on Cloud and Bare
Metal, and the flexibility to optimize for large Bare Metal workloads that
take a long time to send to other nodes.


## Solution

### Principles


1. The design separates the act of fencing from both the detection of errors,
and the remediation logic that drives it

1. The design must include an extendable mechanism for triggering fencing

1. The design will encompass recovery efforts aimed at avoiding fencing, such
as local monitoring for, and the automatic restarting of, wedged daemons.

1. The design will include a mechanism for delaying until triage activities
(such as kdump) have had a chance to complete.

1. The design will include the concept of rebooting a node to facilitate large
bare metal workloads that would take “tens of minutes” or longer to send to a
new worker.

1. It is understood that even if a Reboot capability is added to the Machine
API, cloud backends may choose to support it as Delete+Create, if at all.

1. The design shall detect and prevent fencing storms.

   Fencing storms refer to situations where an abnormally large number
   (usually defined as half) of peers/workers are lost within a short period
   of time. This can be indicative of a problem with the observer rather than
   the peers, in which case fencing should not proceed.

### Components

- **ACT**: A Machine API backend (Actuator) that knows how to add and remove nodes
  from a Kubernetes cluster based on where Kubernetes is deployed.

- **NPD**: node-problem-detector, existing component for detecting failures

- **REM**: Remediation logic that decides when to invoke the Machine API

- **WKR**: Local activities on the worker that detect and attempt to recover
  from internal failures. For example, detecting wedged containers and
  triggering them to be killed & respawned. This is a concept that already
  exists and is expected to expand in scope with time. It is not necessarily a
  single component but a set of capabilities and no interaction with other
  nodes or the higher level recovery mechanism required.

### Scenarios

1. A transient error occurs and is resolved before **NPD** detects it
   1. Nothing happens

1. A transient error occurs and is resolved before **REM** receives the condition change from NPD
   1. **NPD** detects the failure and reports an error condition
   1. **NPD** detects the condition has been resolved and clears the error condition
   1. **REM** receives the error condition and starts a timer
   1. **REM** sees the condition change, stops the timer, and no fencing takes place

1. A worker becomes wedged or unreachable in a manner that **WKR** can self-repair
   1. **NPD** detects the failure and reports an error condition
   1. **REM** receives the error condition and starts a timer
   1. In parallel, **WKR** attempts to detect and recover the issue before the timer expires
   1. **NPD** detects the condition has been resolved and clears the error condition,
   1. **REM** sees the condition change, stops the timer, and no fencing takes place

1. A worker becomes wedged or unreachable in a manner that **WKR** cannot self-repair
   1. **NPD** detects the failure and reports an error condition
   1. **REM** receives the error condition and starts a timer
   1. In parallel, **WKR** attempts to detect and recover the issue before the timer expires
   1. The timer expires
   1. the Machine API is invoked to Delete the worker, allowing the workloads to be rescheduled
   1. the Machine API instructs the configured **ACT** to Delete the worker
   1. **ACT** deprovisions the node
   1. Failed Jobs will be retried after DELAY until they succeed.

   In the case of the initial Bare Metal Actuator, *deprovisioning* translates to:

   1. **ACT** retrieves the relevant config object for that node
   1. **ACT** creates a Job to power off the machine using the configuration details.

   Subsequent versions will eventually deprovision the machine like other ACTs.

1. A worker becomes wedged or unreachable while **NPD** is recovering
   1. As per scenario 1, 3 or 4 depending on how long the condition persists

1. A worker becomes wedged or unreachable while **REM** is recovering
   1. As per scenario 2, 3 or 4 depending on how long the condition persists

1. A worker with a large workload becomes permanently unreachable (assumes a reboot method is available)
   1. **NPD** detects the failure and reports an error condition
   1. **REM** receives the error condition and starts a timer
   1. The timer expires
   1. **REM** checks the size of all current workloads
   1. If any exceed THRESHOLD, a Reboot is requested and the timer is restarted
   1. As per scenario 4 if the **NPD** does not report the worker is healthy before the timer expires again

1. A worker being actively triaged becomes permanently unreachable
   1. Admin configures a ‘triage’ node selector (usually by name or tag) in the REM’s configuration
   1. **NPD** detects the failure and reports an error condition
   1. **REM** receives the error condition
   1. **REM** checks if the worker matches the node selector and either uses a
     longer timer, or has a way to determin if the triage activity is complete

1. A set of workers become permanently unreachable
   1. **REM** receives the error condition(s)
   1. Before starting the timer, **REM** checks the total number of currently failed nodes
   1. If the number exceeds THRESHOLD:
   1. All existing timers are cancelled, no additional Reboot or Delete operations will be attempted
   1. In-progress Delete ops that complete successfully may result in a Create op
   1. In-progress Reboot ops that fail do not result in a Delete op


### User experience

The loss of a worker node should be transparent to user's of StatefulSets.
Recovery time for affected Pods should be bounded and short, allowing scale
up/down events to proceed as normal afterwards.


In the absence of this feature, an end-user has no ability to safely or
reliably allow StatefulSets to be recovered and as such end-users will not be
provided with a mechanism to enable/disable this functionality on a set-by-set
basis.

### Admin experience

Administrators will have control over the thresholds and timers used to tune
the cluster's recovery.

Users will have mechanisms to provide hints to the **REM** controller, but
ultimately it is the admin who defines the SLA.

Administrators will also have the ability to confirm workers are safely
stopped, initiate new fencing requests, and query the results of previous
actions.

#### Fencing Configuration

```go
type FencingConfig struct {
    // Master control switch
    Enabled    bool `json:”enabled"`

    // How many concurrent failures should constitute a “storm” that
    // suggests the problem might be with us and that recovery should
    // be aborted (for now)
    StormThresholdInstances    int `json:”stormThresholdInstances”`

    // Labels, containing a value in seconds, which if set on a
    // Machine/Node represent triage activity that must complete
    // before any recovery actions should take place.  The highest
    // value will be used.
    TriageLabels               []string `json:”triageLabels”`

    // Where each Pod/Job stores its “this is how badly I’d like to
    // avoid being recreated from scratch” number
    DisruptionLabel            string `json:”disruptionLabel”`

    // The name of one of the calculations we have implemented for
    // turning the “this is how badly I’d like to avoid being
    // recreated from scratch” number from each Pod on the node into a
    // single integer value.
    DisruptionCalculation:     string `json:”disruptionCalculation”`

    // Any calculation that produces a number greater than this value
    // will result in the node being rebooted first
    DisruptionRebootThreshold: int `json:”disruptionRebootThreshold”`

    // For any Pod without a value for ‘DisruptionLabel’
    DefaultDisruptionValue    int `json:”defaultDisruptionValue”`

    // How long to wait for the node to recover on its own before
    // taking any action 
    DefaultGraceSeconds int `json:”defaultGraceSeconds”`

    // How long to wait for the node that exceeded the
    // DisruptionRebootThreshold to recover on its own before
    // taking any action 
    DefaultRebootGraceSeconds int `json:”defaultRebootGraceSeconds”`
}
```
Example:

```yaml
Apiversion: v1
kind: ConfigMap
metadata:
  labels:
    app: machine-remediation
  Name: sample-remediation-config 
data:
  remediation-config: |
    stormThresholdInstances: 10
    triageLabels:  [ kdumpRequestedSeconds, manualTriageSeconds ]
    disruptionLabel:         recovery-seconds
    disruptionCalculation:   average
    disruptionRebootThreshold: 300 
    defaultDisruptionValue:    60
    defaultGraceSeconds:       180
    defaultRebootGraceSeconds: 360
```

#### Fencing State

```go
type RemediationState struct {
  // Map of node names to recovery efforts
  NodeState map[string]RemediationNodeState `json:"nodeState"`
}

type RemediationNodeState struct {
  // Node this remediation event applies to
  Name            string  `json:"name"`

  // When the remediation began
  InProgressSince time.Time  `json:"inProgressSince"`

  // Result of the disruptionCalculation function
  Disruption      int `json:"disruption"`

  // What phase of recovery we are in: 
  //    Init, Wait, Reboot, Delete, Create
  Phase           string `json:"phase"`

  // At what time the phase is considered done or failed
  PhaseTimeout    time.Time `json:"phaseTimeout"`

  // A history of past events affecting this node.  Allows a client
  // to query for detail of past actions.
  //
  // Either a fixed number of events will be kept indefinitely, or
  // “old” events will be periodically purged
  Log             []RemediationEvent `json:"log"`
}

type RemediationEvent struct {
  // When the event happened
  Timestamp time.Time `json:"timestamp"`

  // Begin, Progress, Result
  Kind      string `json:"kind"`
  
  // Summary of event
  Message   string `json:"message"`
  
  // Additional detail
  Detail   *string `json:"detail"`
}
```
#### Bare Metal Actuator Configuration ###

```go
type MetalMachineConfig struct {
  // Query that specifies which node(s) this config applies to
  NodeSelector map[string]string `json:"nodeSelector,omitempty"`

  // Container that handles machine operations 
  Container  *v1.Container `json:"container"`

  // Optional command to be used instead of the default when
  // handling machine Create operations (power-on/provisioning)
  CreateCmd []string `json:"createCmd,omitempty"`

  // Optional command to be used instead of the default when
  // handling machine Delete operations (power-off/deprovisioning)
  DeleteCmd []string `json:"deleteCmd,omitempty"`

  // Optional command to be used instead of the default when
  // handling machine Reboot operations
  RebootCmd []string `json:"rebootCmd,omitempty"`

  // How Secrets and DynamicConfig should be passed to the
  // container: ([env], cli)
  ArgumentFormat string `json:"argumentFormat"`

  // Parameters to use for automatic variables
  PassActionAs string `json:"passActionAs,omitempty"`
  PassTargetAs string `json:"passTargetAs,omitempty"`

  // Parameters who’s value changes depending on the affected node
  DynamicConfig []MetalDynamicConfig `json:"dynamicConfig"`

  // A list of Kubernetes secrets to securely pass to the container
  Secrets map[string]string `json:"secrets"`

  // How long to wait for the Job to complete
  TimeoutSeconds *int32 `json:"timeoutSeconds,omitempty"`

  // How long to wait before retrying failed Jobs
  RetrySeconds *int32 `json:"retrySeconds,omitempty"`
}

type MetalDynamicConfig struct {
  Field   string            `json:"field"`
  Default string            `json:"default"`
  Values  map[string]string `json:"values"`
}
```

Example:

```yaml
Apiversion: v1
kind: ConfigMap
metadata:
  labels:
    app: metal-actuator
  Name: sample-metal-config 
data:
  metal-config: |
    nodeSelector:
    - spec.Name: [ hostA, hostB, hostC ]
    container:
      name: baremetal-fencing
      image: quay.io/beekhof/fence-agents:latest
      cmd: [ "/sbin/fence_ipmilan", "--user", "admin" ]
    argumentFormat: cli
    passTargetAs: port
    passActionAs: o
    dynamic_config:
    - field: ip
      default: 127.0.0.1
      # If no default is supplied, an error will be logged and the
      # mechanism will be considered to have failed
      values:
      - hostA: 1.2.3.4
      - hostB: 1.2.3.5
    secrets:
    # An optional list of references to secrets in the same
    # namespace to use when calling fencing modules
    # (ie. IPMI passwords).
    - password: ipmi-secret
```







The design and implementation acknowledge that other entities, such as the autoscaler, are likely to be present and performing similar monitoring and recovery actions. Therefore it is critical that the fencing controller not create or be susceptible to race conditions.



### Implementation
#### Fence Controller
The fence controller is a stateless controller that can be deployed as pod in cluster or process running outside the cluster. The controller identifies unresponsive node by getting events from the apiserver, once node becomes “not ready” the controller posts crd for fence to initiate fence flows.

The controller proidically polls fencenode crds and manage them as follow:
- status:new - Controller creates job objects for each method in step, move to status:running and update jobs list in nodefence object.
- status:running - Poll running jobs and check if all done successfully. Set status:done or status:error based on jobs condition.
- status:done - Check node readiness, if node is ready move to step:recovery. If node still unresponsive, move to step:power-management. If on step:power-management already and not ready move to status:error (move to error after configurable number of pollings before changing status - see cluster-fence-config)
- status:error - Delete all related jobs and move to status:new to retrigger jobs on different nodes.

Job creation gets the authenticatoin parameters to execute fence agent - This is parsed by the controller from the configmaps as described above. On init the controller reads all fence agents' meta-data to perform parameters extraction for creating the job command. New fence-scripts can be dynamically added by dropping scripts to fence-agents folder and rebuilt the agent image.

#### Executor Job
Executor is k8s Job that is posted to cluster by the controller. The job is based on centos image including all fence scripts that are available in cluster. The job only executes one command and returns the exit status. The job is monitored and mantained by the controller as described in Fence Controller section.
Follwing is a list of agents we will integrate using fence scripts:
- Cluster fence operation - E.g: 1) cordon node 2) cleaning resources - deleting pods from apiserver.
- https://github.com/ClusterLabs/fence-agents/tree/master/fence/agents/aws - Cloud provider agent for rebooting ec2 machines.
- https://github.com/ClusterLabs/fence-agents - Scripts for executing pm devices.
- https://github.com/ClusterLabs/fence-agents/blob/master/fence/agents/compute/fence_compute.py - Fence agent for the automatic resurrection of OpenStack compute instances.
- https://github.com/ClusterLabs/fence-agents/tree/master/fence/agents/vmware_soap - Cloud provider agent for rebooting vmware machines.
- https://github.com/ClusterLabs/fence-agents/tree/master/fence/agents/rhevm - Cloud provider agent for rebooting oVirt hosts.

Cloud provider allows us to implement fencing agents that perform power management reboot. In k8s autoscaler implementation the concept of cloud provider is already [implemented](https://github.com/kubernetes/kubernetes/tree/master/pkg/cloudprovider) - we might integrate with that code to support PM operations over AWS and GCE.



### Alternatives considered
1. Create a new [Cloud Provider](https://kubernetes.io/docs/concepts/cluster-administration/cloud-providers/) allowing the [autoscaler](https://github.com/kubernetes/autoscaler) to function for bare metal deployments.
   
   This was considered however the existing APIs are load balancer
   centric and hard to map to the concept of powering on and off nodes.
   
   If the Cloud Provider API evolves in a compatible direction, it
   might be advisable to persue a Bare Metal provider and have it be
   responsible for much of the fencing configuration.

1. Combining the detection and execution logic in a single controller.
   
   While it would be possible to combine this functionality into a
   single controller, the separation of concerns makes it possible
   that other entities could trigger fencing in the future.

1. A solution that focused exclusively on power fencing.
   
   While this would dramatically simplify the configuration required,
   many admins see power fencing as a last resort and would prefer
   less destructive way to isolate a misbehaving node, such as network
   and/or disk fencing.
   
   We also see a desire from admins to use tools such as `kdump` to
   obtain additional diagnostics, when possible, prior to powering off
   the node.

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
