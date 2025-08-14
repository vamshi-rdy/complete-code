# complete-code
all files code

Jenkins Plugins Installed:
  a) Git , Docker, Docker pipeline, Sonarqube

NODE AFFINITY,POD AFFINITY , TAINTS & TOLERATIONS :

🔶 1. Node Affinity
✅ Purpose:
Control which nodes a pod can be scheduled on based on node labels.

🧩 How it Works:
Pods specify node selection rules (based on labels) in their spec.affinity.nodeAffinity.

Replaces older nodeSelector.

🔍 Example:
yaml
Copy
Edit
nodeSelector:
  disktype: ssd
or more powerfully:

yaml
Copy
Edit
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: disktype
          operator: In
          values:
          - ssd
🧠 Use Cases:
Schedule pods only on nodes with GPU, ARM CPU, NVMe disks, etc.

Schedule pods to specific availability zones for fault tolerance.

✅ When to Use:
You want to control which nodes your pods run on.

You rely on hardware features or regional requirements.

🔷 2. Pod Affinity / Anti-Affinity
✅ Purpose:
Control how pods are co-located (affinity) or spread apart (anti-affinity) based on the labels of other pods.

🧩 How it Works:
You specify podAffinity or podAntiAffinity in spec.affinity.

Kubernetes places pods based on existing pod labels on a node (or topology like zone).

🔍 Example: Pod Affinity
Schedule this pod with other pods having the label app: frontend:

yaml
Copy
Edit
affinity:
  podAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: frontend
      topologyKey: "kubernetes.io/hostname"
🔍 Example: Pod Anti-Affinity
Keep this pod away from other pods with label app: frontend:

yaml
Copy
Edit
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: frontend
      topologyKey: "kubernetes.io/hostname"
🧠 Use Cases:
Affinity:

Co-locate services that depend on each other (e.g., microservices communicating locally).

Place caching and application pods together.

Anti-Affinity:

Spread replicas of a deployment across nodes or zones for high availability.

Prevent noisy neighbors (e.g., multiple DB pods on the same node).

✅ When to Use:
For inter-pod relationships and HA/failure isolation.

When you care about pod-to-pod topology.

⚫ 3. Taints and Tolerations
✅ Purpose:
Taints are applied to nodes to repel pods. Tolerations are applied to pods to allow scheduling on tainted nodes.

🧩 How it Works:
You taint a node like:

bash
Copy
Edit
kubectl taint nodes node1 key=value:NoSchedule
Then only pods that tolerate that taint can run there:

yaml
Copy
Edit
tolerations:
- key: "key"
  operator: "Equal"
  value: "value"
  effect: "NoSchedule"
🧠 Use Cases:
Reserve nodes for specific workloads (e.g., GPU, billing, ML).

Prevent regular pods from being scheduled on dedicated or critical infrastructure nodes.

Isolate insecure or noisy workloads.

✅ When to Use:
You need explicit control over which pods can run on certain nodes.

You want to protect certain nodes from random scheduling.

You run multi-tenant clusters and want hard boundaries.

🆚 Comparison Table
Feature	Node Affinity	Pod Affinity / Anti-Affinity	Taints and Tolerations
Scope	Node-level	Pod-level	Node-level
Based On	Node labels	Labels of other pods	Taints on nodes / Tolerations on pods
Purpose	Prefer/restrict pods to certain nodes	Co-locate or separate pods	Prevent pods from being scheduled unless tolerated
Type	Soft or hard constraints	Soft or hard constraints	Hard/soft repel
Use Case	Place pod on SSD/GPU nodes	Group DB and app pods, or separate replicas	Keep production pods off dev nodes
Control Direction	Pod → Node	Pod → Pod	Node → Pod

🧠 Best Practices
✅ Use Node Affinity:
When specific node features are needed (e.g., hardware, labels).

For zonal placement or regional isolation.

✅ Use Pod Affinity/Anti-Affinity:
For co-locating or spreading related/unrelated services.

For high availability across nodes/zones.

✅ Use Taints and Tolerations:
To protect special nodes from unqualified pods.

For multi-tenant, isolated, or sensitive workloads.

When running dedicated infrastructure nodes (monitoring, logging, GPU).
