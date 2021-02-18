## Cluster lifecycle
### 1. Provisioning

You initiate cluster creation. Refer to [getting started](https://github.com/v1dm45/docs/blob/main/docs/getting-started.md) to learn how to create a cluster.


### 2. Reconciliation & healing

The cluster enters a reconciliation loop. The platform periodically re-checks that actual infrastructure on your cloud reflects the specified configuration, and performs upgrades and patching. Reconciliation performs checks such as:

  - [x] Cluster network configuration is up to date;
  - [x] No nodes are missing, for example, were accidentally deleted;
  - [x] Any unused resources to clean up;

### 3. Resizing

CAST AI clusters do not use the "node pool" concept. Instead, you can: 

   - Manually add or remove nodes with a specified configuration.
   - Enable autoscaling policies to scale up and down per-node level.

### 4. Cleanup

When you delete a cluster, the platform will collapse cloud resources in the fastest way. Nodes will not be drained before being deleted.

The platform is designed to minimize unintended removals. If you have any extra virtual machines that do not contain CAST AI cluster UUID - the delete operation will fail.

## Cluster architecture

### Context

When you [create a cluster](https://github.com/v1dm45/docs/blob/main/docs/getting-started.md#create-cluster) you can [download kubeconfig](https://github.com/v1dm45/docs/blob/main/docs/getting-started.md#deploy-application) to access your cluster directly. Some of the middleware running on the cluster (Grafana, the Kubernetes dashboard) is directly reachable from the [console UI](https://github.com/v1dm45/docs/blob/main/docs/Dashboard%20Overview/Console%20overview.md#console-overview) through the single-signon gateway.

You can see on the diagram below that there is a bi-direction link between your cluster and the CAST AI platform. The platform not only connects to your cloud infrastructure or the cluster itself, it also relies on the cluster to "call back" and inform about certain events:

* Cluster control plane nodes actions with the provisioning engine, e.g., when to join the cluster;
* Nodes inform about operations being completed, like finishing to join the cluster;
* Relevant cloud events get propagated to the provisioning engine & autoscaler, for example, "spot instance is being terminated by cloud provider";

Your app users do not interact with CAST AI in any way. You own your Kubernetes cluster infrastructure by 100%, including any [ingress infrastructure](https://github.com/v1dm45/docs/blob/main/docs/concepts/architecture-overview.md#ingress) to reach your cluster workloads.

The diagram below highlights the primary groups of components that define a relationship between CAST AI platform and your cluster:

![](architecture-overview/component-relationships.png)

## Cluster infrastructure

### Nodes

Overview on where cluster virtual machines will be provisioned on your cloud:

![](architecture-overview/nodes-infrastructure.svg)

### Ingress

The CAST AI provisioned clusters contain all the infrastructure needed to equip your app with an external TLS endpoint:

* DNS entry to round-robin;
* Load-balancing infrastructure: cloud-native load balancers that route traffic to a sub-section of your cluster (e.g., traffic that hits AWS load balancer will route to AWS nodes);
* Nginx ingress controller paired with TLS certificate manager that listen to your deployed resources and maintain routing&TLS configuration;
* Metrics collection for your ingress traffic;

All that is left for you as an application developer is to deploy your app, ingress resource, and configure a domain alias of your choice. See the [ingress guide](../guides/ingress.md) for more details.

![](architecture-overview/ingress.png)

### Network details

#### Region & zone

Each cloud maps a selected CAST AI region to a matching region on that cloud.

For example, **US East (Ashburn)** region maps to:

* AWS: us-east-1
* GCP: us-east4
* Azure: eastus
* Digital Ocean: nyc1

Currently, on each cloud CAST AI builds a single-zone setup of your cluster. Zone selection is cloud-specific.

#### Master nodes inbound

| Protocol | Port | Source | Description |
|---|---|---|---|
| tcp | 6443 | 0.0.0.0/0 | k8s API server |  
| udp | 51820 | 0.0.0.0/0 | WireGuard (if used)|

#### Worker nodes inbound

| Protocol | Port | Source | Description |
|---|---|---|---|
| udp | 51820 | 0.0.0.0/0 | WireGuard (if used) |
| tcp/udp | NodePort | 0.0.0.0/0 | k8s Service with type=LoadBalancer |

#### Subnets

| Range | Description |
|---|---|
| 10.96.0.0/12 | k8s services |
| 10.217.0.0/16 | k8s pods |
| 10.4.0.0/16 | WireGuard|
| 10.0.0.0/16 | GCP VPC. Smaller /24 blocks are allocated for subnets. |
| 10.10.0.0/16 | AWS VPC. Smaller /24 blocks are allocated for subnets. |
| 10.20.0.0/16 | AZURE VPC. Smaller /24 blocks are allocated for subnets. |
| 10.100-255.0.0/20 | DigitalOcean VPC. There is only one subnet which is allocated dynamically. |
