|---|---|
| `dedicated` | Key: identifies the property or category. |
| `database` | Value: specifies the value assigned to the key. |
| `NoSchedule` | Effect: prevents new pods that do not tolerate this taint from being scheduled on the node. |

The key and value are names chosen by the administrator. They are not special Kubernetes keywords.

### Taint Effects

| Effect | Meaning |
|---|---|
| `NoSchedule` | New pods that do not tolerate the taint cannot be scheduled on the node. Existing pods are not automatically evicted. |
| `PreferNoSchedule` | Kubernetes tries to avoid scheduling pods that do not tolerate the taint, but this is not a strict restriction. |
| `NoExecute` | Pods that do not tolerate the taint cannot be scheduled on the node, and existing non-tolerating pods are evicted. |

### Commands to Inspect Taints

**Step 1: List nodes**

```bash
kubectl get nodes

Step 2: Inspect a specific node

kubectl describe node <node-name>

Look for the Taints section.

Example:
Taints: node-role.kubernetes.io/control-plane:NoSchedule

If the node has no taints, you may see:
Taints: <none>

Step 3: Display only the taints

kubectl get node <node-name> -o jsonpath='{.spec.taints}'
echo

Add a Taint

kubectl taint nodes <node-name> dedicated=database:NoSchedule

This adds the taint dedicated=database:NoSchedule to the selected node.
Remove a Taint

kubectl taint nodes <node-name> dedicated=database:NoSchedule-

The trailing - removes the specified taint.
3. Tolerations
Theory

A toleration is a rule in a pod's specification that allows the pod to tolerate a matching taint.

Suppose a node has this taint:

dedicated=database:NoSchedule

A pod without a matching toleration cannot be scheduled on that node because of this taint.

To allow the pod to tolerate the taint, add a toleration to its YAML manifest.
Example YAML

tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "database"
    effect: "NoSchedule"

Understanding the Fields
Field	Meaning
key	The taint key to match.
operator: Equal	Requires the taint's key and value to match the toleration's key and value.
value	The expected taint value.
effect	The taint effect this toleration applies to.
How Matching Works

Node taint:
dedicated=database:NoSchedule

Pod toleration:

key: "dedicated"
operator: "Equal"
value: "database"
effect: "NoSchedule"

The key, value, and effect match.

Therefore, this taint does not prevent the pod from being scheduled on the node.

If the pod instead has:

value: "web"

the value does not match database, so the pod does not tolerate this taint.

Important: A toleration does not guarantee that Kubernetes will schedule the pod on that node. It only allows the pod to tolerate the matching taint.
Where Does Toleration Go in a Deployment Manifest?

Tolerations belong inside the pod template's specification:

spec.template.spec.tolerations

Example:

apiVersion: apps/v1
kind: Deployment

metadata:
  name: example-app

spec:
  replicas: 1

  selector:
    matchLabels:
      app: example-app

  template:
    metadata:
      labels:
        app: example-app

    spec:
      tolerations:
        - key: "dedicated"
          operator: "Equal"
          value: "database"
          effect: "NoSchedule"

      containers:
        - name: nginx
          image: nginx:latest

This is an example for learning scheduling. It uses NGINX, not a real database.
4. Node Selector
Theory

A node selector tells Kubernetes to schedule a pod only on nodes with matching labels.

Suppose a node has this label:

workload=database

The pod can use the following configuration:

nodeSelector:
  workload: database

Kubernetes will only consider nodes with the label workload=database for this pod.

If no suitable node has that label, the pod may remain in the Pending state.
Add a Label to a Node

kubectl label node <node-name> workload=database

Inspect Node Labels

kubectl get node <node-name> --show-labels

Remove a Label

kubectl label node <node-name> workload-

Node Selector vs. Toleration
Node Selector	Toleration
Selects nodes based on labels.	Allows a pod to tolerate matching taints.
Determines which nodes are eligible based on the selector.	Does not select or reserve a node.
Configured in the pod specification.	Configured in the pod specification.
Use Both Together

Suppose the target node has:

Label:

workload=database

Taint:

dedicated=database:NoSchedule

The pod can use both configurations:

spec:
  nodeSelector:
    workload: database

  tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "database"
      effect: "NoSchedule"

  containers:
    - name: nginx
      image: nginx:latest

The node selector selects nodes with the required label, while the toleration allows the pod to tolerate the matching taint.

The node must also have sufficient resources and satisfy the pod's other scheduling requirements.
5. Cordon
Theory

Cordon marks a node as unschedulable.

This means Kubernetes will not schedule new ordinary pods on that node.

However, existing pods generally continue running.

Cordon is useful when preparing a node for maintenance.
Cordon a Node

kubectl cordon <node-name>

Example:

kubectl cordon worker-1

Check the Node Status

kubectl get nodes

The node may show:
Ready,SchedulingDisabled

SchedulingDisabled means the node has been marked unschedulable.
Important Difference: Cordon vs. Drain

    Cordon: Prevents new pods from being scheduled on the node.
    Drain: Evicts eligible workloads from the node, typically before maintenance.

Cordon alone does not evict existing pods.
6. Uncordon
Theory

Uncordon makes a cordoned node schedulable again.

After uncordoning, Kubernetes can schedule new pods on that node if it is otherwise suitable.

Uncordon does not automatically move existing pods back onto the node.
Uncordon a Node

kubectl uncordon <node-name>

Example:

kubectl uncordon worker-1

Verify the Status

kubectl get nodes

The SchedulingDisabled status should disappear if the node is otherwise Ready.
7. Quick Comparison
Requirement	Concept	Command or YAML
Restrict scheduling on a node	Taint	kubectl taint nodes worker-1 dedicated=database:NoSchedule
Allow a pod to tolerate a taint	Toleration	spec.template.spec.tolerations
Select nodes using labels	Node Selector	spec.template.spec.nodeSelector
Prevent new pods from being scheduled on a node	Cordon	kubectl cordon worker-1
Allow scheduling on a node again	Uncordon	kubectl uncordon worker-1
8. Safe Inspection Checklist

Before making changes to a cluster, inspect its current configuration.

1. List the nodes

kubectl get nodes

2. Inspect a node and its taints

kubectl describe node <node-name>

3. Display only the taints

kubectl get node <node-name> -o jsonpath='{.spec.taints}'
echo

4. Display node labels

kubectl get node <node-name> --show-labels

Lab Safety Note

If your cluster has only one node, cordoning it may leave new pods in the Pending state because no other node is available.

Adding a NoSchedule taint can also prevent new pods from being scheduled if they do not tolerate it.

Inspect your node's existing taints and workloads before experimenting.
9. Memory Aid

    Taint: The node says, "Keep out unless you tolerate this restriction."
    Toleration: The pod says, "I can tolerate that restriction."
    Node Selector: The pod says, "Only consider nodes with this label."
    Cordon: "Do not schedule new pods on this node."
    Uncordon: "This node can accept newly scheduled pods again."
