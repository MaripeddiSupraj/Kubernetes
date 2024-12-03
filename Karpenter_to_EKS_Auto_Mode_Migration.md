## Migrating from Karpenter to EKS Auto Mode


---

### **What is EKS Auto Mode?**

Amazon EKS Auto Mode automates Kubernetes cluster management on AWS with a single click. It handles infrastructure provisioning, instance selection, resource scaling, cost optimization, OS patching, and security integration. This reduces the need for deep Kubernetes expertise and ongoing management, enabling you to focus on building innovative applications while AWS manages scalability and performance.

### Purpose
This guide provides a step-by-step process for migrating your workloads from Karpenter to Amazon EKS Auto Mode using `kubectl`. You can carry out the migration gradually, moving workloads at your own pace while maintaining cluster stability and application availability throughout the transition.

The approach outlined below allows you to run Karpenter and EKS Auto Mode simultaneously during the migration period. This dual-operation strategy helps ensure a seamless transition by enabling you to validate how your workloads behave on EKS Auto Mode before completely phasing out Karpenter. You have the flexibility to migrate applications individually or in batches, accommodating your specific operational needs and risk tolerance.

### Prerequisites

Before beginning the migration, ensure you have:

> Karpenter v1.1 or later installed on your cluster. For more information, see Upgrading to 1.1.0+ in the Karpenter docs. https://karpenter.sh/

> kubectl installed and connected to your cluster. For more information, see https://docs.aws.amazon.com/eks/latest/userguide/setting-up.html

This topic assumes you are familiar with Karpenter and NodePools. For more information, see the Karpenter Documentation.

### Step 1: Enable EKS Auto Mode on the cluster

Enable EKS Auto Mode on your existing cluster using the AWS CLI or Management Console. For more information, see Enable EKS Auto Mode on an existing cluster.


> While enabling EKS Auto Mode, don’t enable the general purpose nodepool at this stage during transition. This node pool is not selective. For more information, see Enable or Disable Built-in NodePools.
>
### Step 2: Create a tainted EKS Auto Mode NodePool


Create a new NodePool for EKS Auto Mode and apply a taint to it. This ensures that existing pods won’t be scheduled on the new EKS Auto Mode nodes by default. The NodePool leverages the default NodeClass provided by EKS Auto Mode.

Example NodePool

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: eks-auto-mode
spec:
  template:
    spec:
      requirements:
        - key: "eks.amazonaws.com/instance-category"
          operator: In
          values: ["c", "m", "r"]
      nodeClassRef:
        group: eks.amazonaws.com
        kind: NodeClass
        name: default
      taints:
        - key: "eks-auto-mode"
          effect: "NoSchedule"
```
### Explanation

1. **Requirements Field**:
   - **`key: "eks.amazonaws.com/instance-category"`**: Defines the instance categories (`c`, `m`, `r`) for the NodePool:
     - `"c"`: Compute-optimized for high CPU workloads.
     - `"m"`: General-purpose for balanced workloads.
     - `"r"`: Memory-optimized for large memory needs.
   - This ensures consistency with your Karpenter configuration.

2. **Minimum Requirement**:
   - The `requirements` field must include at least one key.
   - Specifying `eks.amazonaws.com/instance-category` ensures nodes are provisioned to support your workloads.

This setup ensures the NodePool matches your Karpenter configuration while leveraging EKS Auto Mode capabilities.


### Step 3: Update workloads for migration

Identify the workloads you want to migrate to EKS Auto Mode and update their configurations by adding both tolerations and node selectors. Here's an example configuration:

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      tolerations:
        - key: "eks-auto-mode"
          effect: "NoSchedule"
      nodeSelector:
        eks.amazonaws.com/compute-type: auto
```

- **Tolerations**: Allow the workload to be scheduled on nodes with the `eks-auto-mode` taint.
- **NodeSelector**: Ensures that the workload is specifically scheduled on EKS Auto Mode nodes.

Note that EKS Auto Mode uses different labels compared to Karpenter. Labels for EC2 managed instances in EKS Auto Mode start with `eks.amazonaws.com`. For more details, refer to the AWS documentation on creating a Node Pool for EKS Auto Mode.
### **Step 4: Gradually Migrate Workloads**

Repeat the process outlined in Step 3 for each workload you plan to migrate. You can choose to migrate workloads one at a time or in groups, depending on your operational needs and risk tolerance.

This gradual approach ensures:

- **Minimal Disruption**: Workloads are moved incrementally, reducing the risk of downtime.
- **Validation**: Allows you to test workload behavior on EKS Auto Mode nodes before fully transitioning.
- **Flexibility**: Provides the ability to adjust the migration pace based on your requirements.

### Step 5: Decommission the Original Karpenter NodePool

After successfully migrating all workloads, you can delete the original Karpenter NodePool to finalize the transition:

```bash
kubectl delete nodepool <original-nodepool-name>
```

---

### Step 6: Remove Taint from EKS Auto Mode NodePool (Optional)

If you want EKS Auto Mode nodes to be the default for new workloads, you can remove the taint from the EKS Auto Mode NodePool. This allows workloads to be scheduled on these nodes without requiring specific tolerations.

**Updated YAML (Taint Removed):**

```yaml
apiVersion: karpenter.sh/v1alpha5
kind: NodePool
metadata:
  name: eks-auto-mode
spec:
  template:
    spec:
      nodeClassRef:
        group: eks.amazonaws.com
        kind: NodeClass
        name: default
      # Taints section removed
```

- **Why Remove the Taint?**
  - Ensures that new workloads can be scheduled on EKS Auto Mode nodes by default.
  - Simplifies workload deployment without needing additional tolerations.
### Step 7: Remove Node Selectors from Workloads (Optional)

If you have removed the taint from the EKS Auto Mode NodePool and want to simplify your workloads, you can optionally remove the `nodeSelector` from their configuration. This allows EKS Auto Mode nodes to become the default for scheduling workloads.

**Updated YAML (Node Selector Removed):**

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      # NodeSelector section removed
      tolerations:
        - key: "eks-auto-mode"
          effect: "NoSchedule"
```

- **Why Remove the Node Selector?**
  - Simplifies workload configurations.
  - Allows new and existing workloads to be scheduled automatically on EKS Auto Mode nodes without explicitly targeting them.

---

### Step 8: Uninstall Karpenter from Your Cluster

After migrating all workloads and ensuring they are running smoothly on EKS Auto Mode, you can uninstall Karpenter from your cluster. The uninstallation process varies based on how Karpenter was initially installed. For detailed steps, refer to the [Karpenter installation documentation](https://karpenter.sh/docs/getting-started/) and the [Helm Uninstall Command](https://helm.sh/docs/helm/helm_uninstall/).

- **Steps May Include**:
  1. Removing Karpenter resources, such as the Karpenter controller.
  2. Deleting namespaces or specific configurations associated with Karpenter.

