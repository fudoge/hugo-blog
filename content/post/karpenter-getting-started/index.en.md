---
title: "Karpenter Explained: Efficient Kubernetes Node Autoscaling on EKS"
slug: "karpenter-explained-efficient-kubernetes-node-autoscaler"
description: "Learn how Karpenter improves Kubernetes node autoscaling by quickly provisioning right-sized nodes, reducing infrastructure waste, and replacing traditional Cluster Autoscaler workflows."
date: 2026-05-01T16:34:04+09:00
image: simple-architecture.png
math: 
license: 
hidden: false
comments: true
draft: false
---

## ❓ What is Karpenter?

![Karpenter: Node Autoscaler](simple-architecture.png)

Karpenter is a Kubernetes node autoscaler designed for cloud-based Kubernetes environments. \
It is most commonly used with Amazon EKS, which is the standard and recommended approach. \
It can also work in non-EKS AWS environments, though this is not generally recommended. \
In addition, Karpenter-style autoscaling is available or supported in other cloud environments such as AKS and Alibaba Cloud. 

Karpenter provides several key advantages:
- **Automatic node scaling**: When there are unscheduled pods, Karpenter provisions new worker nodes so that those pods can run. \
  It can also remove or replace nodes when they are no longer needed.
- **Cost efficiency**: Karpenter optimizes worker node usage based on pod resource requests, using bin-packing to reduce waste.
- **Speed**: Karpenter is generally faster than Cluster Autoscaler, the traditional node autoscaling approach for EKS.

When you run EKS with Auto Mode, AWS manages Karpenter automatically for you. \
However, EKS Auto Mode can cost more than operating a self-managed EKS cluster.

---
## 🚀 How It Works

### Provisioning

![Provisioning](provisioning.png)

Karpenter provisions new nodes when Kubernetes cannot schedule pods due to insufficient cluster capacity.

The provisioning flow works as follows:

1. Karpenter continuously watches for pods that are pending or unschedulable.
2. Karpenter selects an appropriate `NodePool` and uses its associated `NodeClass`, such as `EC2NodeClass`, to determine how the node should be created.
3. Karpenter creates a `NodeClaim` and requests AWS to launch a new EC2 instance.
4. The new EC2 instance initializes as an EKS worker node.
5. The worker node joins the EKS cluster, allowing the pending pods to be scheduled.

### Disruption

Karpenter also manages node disruption. It automatically detects nodes that can be disrupted and creates replacement nodes when needed. \
Disruption may happen for several reasons:

- **Drift**
  - The node no longer matches the desired configuration defined in the `NodePool` or `NodeClass`.
- **Consolidation**
  - The node is empty.
  - The node is underutilized.
  - The workloads can be moved to fewer or cheaper nodes.
- **Expiration**
  - The node has reached its configured lifetime, such as `expireAfter` in the `NodePool` spec.
- **Interruption**
  - AWS sends an interruption or termination event for the underlying instance.

Among these, drift-based disruption is usually more important than consolidation because it ensures that nodes stay aligned with the desired cluster configuration. \
The disruption flow is shown in the following diagram:

![Disruption](disruption.png)

#### Disruption Controller

1. Checks whether the pods on the node can be evicted.
2. Checks the disruption budget.
3. Runs a scheduling simulation to verify that workloads can be rescheduled.
4. Adds the taint `karpenter.sh/disrupted:NoSchedule` to prevent new pods from being scheduled on the node.
5. Creates a new `NodeClaim` and waits for the replacement node to become ready.
6. Deletes the node.

#### Termination Controller

1. A `DeletionTimestamp` is set on the node, and the finalizer blocks immediate deletion.
2. The taint `karpenter.sh/disrupted:NoSchedule` is added.
3. Pods are evicted through the Kubernetes Eviction API while respecting PDBs.
4. Karpenter verifies that all `VolumeAttachments` have been deleted.
5. The related `NodeClaim` and EC2 instance are terminated.
6. The finalizer is removed.
7. Node deletion is completed.

> **Why is disruption divided into two phases?**  \
> A Karpenter-managed node can be deleted in different ways, such as by running `kubectl delete node <node-name>`. \
> Because of this, the termination process must be separated from the disruption decision-making process. The Termination Controller acts as a graceful node shutdown mechanism, regardless of how the node deletion was triggered. \
> This is also why the taint `karpenter.sh/disrupted:NoSchedule` is added during both processes. 

### Interruption

![Interruption](interruption.png)

Interruption is a special type of disruption triggered by AWS events. \
When an interruption event occurs, Karpenter starts the disruption process for the affected node. 

The following events can trigger interruption handling:
- Spot Interruption Warnings
- Scheduled Change Health Events, such as maintenance events
- Instance Terminating Events
- Instance Stopping Events
- Instance Status Check Failures

The interruption flow works as follows:
1. An EventBridge rule receives the AWS event.
2. The event message is sent to an SQS queue.
3. The Karpenter controller periodically polls the SQS queue.
4. When Karpenter detects an interruption event, it taints, drains, and terminates the affected node.

For Spot Interruption Warnings, Karpenter provisions a replacement node in parallel while terminating the interrupted node. \
AWS also publishes Spot Rebalance Recommendation events. However, these events do not trigger Karpenter’s taint, drain, and terminate logic in the same way. \
To enable interruption handling, Karpenter must be started with the following flag:

```bash
--interruption-queue=<queue-name>
```

---
## 🍪 Karpenter Prerequisites

Before installing Karpenter, you need to prepare several AWS resources and permissions. \
The required prerequisites are:

- **IAM role for Karpenter**
  - Used by the Karpenter controller to call AWS APIs.
- **IAM role for Karpenter-provisioned nodes**
  - Attached to worker nodes launched by Karpenter.
- **Subnets for Karpenter-provisioned nodes**
  - Each subnet must have the `karpenter.sh/discovery` tag.
- **Security groups for Karpenter-provisioned nodes**
  - Each security group must have the `karpenter.sh/discovery` tag.
- **EKS access entry for Karpenter-provisioned nodes**
  - The access entry type should be `EC2_LINUX` or `EC2_WINDOWS`, depending on the node operating system.
- **Optional: SQS queue**
  - Required when using Karpenter’s interruption handling feature.
- **Optional: EventBridge rule and target**
  - Used to forward AWS interruption events to the SQS queue.

You can refer to the following IaC examples:
- [(Official) CloudFormation](https://karpenter.sh/docs/reference/cloudformation)
- [Author's Terraform code](https://github.com/Hallym-Workerbees/hivewiki-infra/blob/main/modules/karpenter-prerequisite/main.tf)

---
## 📊 Installing Karpenter with Helm

Karpenter can be installed easily using the official Helm chart.

```bash
helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter \
  --version <KARPENTER_VERSION> \
  --namespace <KARPENTER_NAMESPACE> \
  --create-namespace \
  --set "settings.clusterName=<CLUSTER_NAME>" \
  --set "settings.interruptionQueue=<INTERRUPTION_QUEUE_NAME>" \
  --set "settings.enableZonalShift=<ENABLE_ZONAL_SHIFT>" \
  --wait
```
The main values are:
- `settings.clusterName`: The name of the EKS cluster.
- `settings.interruptionQueue`: The SQS queue used for interruption handling.
- `settings.enableZonalShift`: Enables zonal shift support, allowing Karpenter to avoid impaired Availability Zones when supported.

---
## ⚙️ Configuration Example

### NodePool

Here is a simple `NodePool` example. 

To use Karpenter, you need at least one `NodePool` and one corresponding `NodeClass`, such as `EC2NodeClass`.

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  # Priority of this NodePool.
  # A higher weight gives the NodePool higher scheduling priority.
  weight: 10

  template:
    spec:
      # Node requirements
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]

        - key: kubernetes.io/os
          operator: In
          values: ["linux"]

        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand", "reserved"]

        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["c", "m", "r"]

        - key: karpenter.k8s.aws/instance-generation
          operator: Gt
          values: ["2"]

      # Reference to the EC2NodeClass
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default

      # Node expiration time
      expireAfter: "720h"

  limits:
    # Maximum CPU capacity that this NodePool can provision
    cpu: 64

  disruption:
    # If set to WhenEmptyOrUnderutilized, Karpenter may remove or replace
    # empty or underutilized nodes to reduce cost.
    #
    # If set to WhenEmpty, Karpenter only consolidates nodes that do not
    # have any workload pods.
    consolidationPolicy: WhenEmptyOrUnderutilized

    # The amount of time Karpenter waits before consolidating a node
    # after a pod has been added or removed.
    consolidateAfter: 1m
```


### NodeClass

Here is an `EC2NodeClass` example.

`EC2NodeClass` defines AWS-specific node settings, such as the node IAM role, AMI selection, subnet selection, security group selection, and EC2 tags.

```yaml
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  # You must specify either `role` or `instanceProfile`.
  role: <NODE_ROLE_NAME>

  # AMI selection
  amiSelectorTerms:
    - alias: al2023@latest

  # Subnet discovery
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: <CLUSTER_NAME>

  # Security group discovery
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: <CLUSTER_NAME>

  # Tags added to EC2 instances created by Karpenter
  tags:
    ManagedBy: "Karpenter"
```

---
## 🧑‍🔬 Testing Node Autoscaling

Once your setup is complete, you can test whether Karpenter provisions nodes correctly.

Create a sample `nginx` deployment with enough replicas and CPU requests to trigger node autoscaling:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 10
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          resources:
            requests:
              cpu: "500m"
EOF
```

This deployment creates 10 pods, each requesting 500m CPU. \
If your current cluster does not have enough available capacity, some pods will remain pending. Karpenter should detect those pending pods and provision new worker nodes automatically. \
You can watch the pods and nodes with the following commands:

```bash
kubectl get pods -w
kubectl get nodes -w
```

You can also check whether the newly created EC2 instances have the tag defined in your EC2NodeClass:
```bash
ManagedBy: Karpenter
```
![Node](node.png)


---
## 📚 References

- [Karpenter - documentations](https://karpenter.sh/docs/)
