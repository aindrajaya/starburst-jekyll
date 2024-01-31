---
layout: default
title: Plan your Kubernetes Deployment
parent: Kubernetes cluster design for Starburst Enterprise
grand_parent: Deploy with Kubernetes
nav_order: 1
permalink: /docs/kubernetes/kubernetes-cluster-design-for-starburst-enterprise/requirements
---

# Plan your Kubernetes deployment

Kubernetes (k8s) support for Starburst Enterprise platform (SEP) allows you to run your SEP clusters
with your additional components such as the Hive Metastore Service (HMS) or
Apache Ranger.

The following sections detail the specific requirements and considerations when
planning for SEP deployments using k8s.

(k8s-cluster-requirements)=

## K8s cluster requirements

The following k8s versions are supported:

- 1.27
- 1.26
- 1.25
- 1.24

In addition to the SEP requirements listed in our {doc}`deployment basics
</installation/deployment>` topic, your cluster nodes must fulfill the following
requirements:

- 64 to 256 GB RAM
- 16 to 64 cores
- All nodes must be identical.
- Each node is dedicated to one SEP worker or coordinator only.
- Nodes are not shared with other applications running in the cluster.
- x86_64 (AMD64) or AArch64 (ARM64) processor architecture

## K8s platform services

The following Kubernetes platform services are tested regularly and supported:

- Amazon Elastic Kubernetes Service (EKS)
- Google Kubernetes Engine (GKE)
- Microsoft Azure Kubernetes Service (AKS)
- Red Hat OpenShift
- Rancher RKE2

This topic focuses on k8s usage and practice that applies to all services.

Other Kubernetes distributions and installations may work if they fulfill the
requirements, but they are not tested and not supported.

:::{warning}
SEP is designed to work with homogenous resource allocation in terms of
network, CPU, and memory for all workers. Resource sharing on a node or pod is
therefore **not recommended**. Failure to ensure this setup can result in
unpredictable query performance and query processing failures.

A simple and recommended approach for optimal operation is to ensure that each
node only hosts one coordinator or worker pod, and nodes and pods are not
shared with other applications. SEP performs best with exclusive access to
the underlying memory and CPU resources since it uses significant resources on
each node.

The recommended approach to achieve this is to use a dedicated cluster or
namespace for all SEP nodes. You can read more about this in our
{doc}`cluster design guidelines <sep-configuration>`.

If you operate SEP in a cluster shared with other applications, you must
ensure that all nodes have reliable access to the same amount of CPU, memory,
and network resources. You can {ref}`use nodegroups, taints and tolerations,
or pod affinity and anti-affinity <helm-node-assignment>` as well as CPU and
memory requests and limits to achieve consistency across nodes. This approach
is not recommended, since it is more complex to implement, but can be used by
experienced k8s administrators. Contact your account team for assistance, if
you are required to use this type of deployment.
:::

(k8s-scaling-requirements)=

### Scaling requirements

In your SEP deployment, install the following components:

- [Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Metrics Server](https://github.com/kubernetes-sigs/metrics-server)
- [Prometheus](https://prometheus.io/)

Automatic scaling adds and removes worker nodes based on demand. This differs
from the commonly used horizontal scaling where new pods are started on existing
nodes, and is a result of the fact that workers require a full dedicated node.
You must ensure that your k8s cluster supports this addition of nodes and has
access to the required resources.

### Access requirements

Access to SEP from outside the cluster using the {{oss}} CLI, or any other
application, requires the coordinator to be available via HTTPS and a DNS
hostname.

This can be achieved with an external load balancer and DNS that terminates
HTTPS and reroutes to HTTP requests inside the cluster.

Alternatively you can configure a DNS service for your k8s cluster and configure
ingress appropriately.

## Service database requirement

The SEP {doc}`backend service </admin/backend-service>` is required. You must
provide an externally-managed database appropriate to your environment for the
service to use.

## Installation tool requirements

- [kubectl](https://kubernetes.io/docs/reference/kubectl/kubectl/)
- [Helm](https://helm.sh/) 3.2.4+

In addition, we strongly recommend [Octant](https://octant.dev/) to simplify
cluster workload visualization and management. The [Octant Helm plugin](https://github.com/bloodorangeio/octant-helm) can simplify usage further.

:::{note}
Check out our {ref}`helm-troubleshooting-tips`.
:::

(k8s-helm-repository)=

## Helm chart repository

The Helm charts and Docker images required for deployment and operation are
available in the {{sb}} Harbor instance at [https://harbor.starburstdata.net](https://harbor.starburstdata.net/).

To obtain a customer-specific Harbor account, contact {{support}}.

To add the Helm repository to your cluster, use the following command:

```shell
$ helm repo add \
  --username yourusername \
  --password yourpassword \
  --pass-credentials \
  starburstdata \
  https://harbor.starburstdata.net/chartrepo/starburstdata
```

To confirm the Helm repository was successfully added to your cluster, use the
following command:

```shell
$ helm repo list
NAME           URL
starburstdata  https://harbor.starburstdata.net/chartrepo/starburstdata
```

To view Helm charts in the repository, use the following command:

```shell
$ helm search repo
NAME                                 CHART VERSION  APP VERSION  DESCRIPTION
starburstdata/starburst-hive         434.0.0                     Helm chart for Apache Hive
starburstdata/starburst-enterprise   434.0.0        1.0          A Helm chart for Starburst Enterprise
starburstdata/starburst-ranger       434.0.0                     Apache Ranger
```

To update to a new release of SEP, use the following command:

```shell
$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "starburstdata" chart repository
Update Complete. ⎈ Happy Helming!⎈
```

(k8s-docker-registry)=

## Docker registry

The Helm charts reference the Docker registry on Harbor to download the relevant
Docker images. You can also use your own Docker registry.

For more information, see {ref}`helm-sep-image`.

## License

In order to use SEP, you must get a license file from {{support}}. For
information on how to configure your license file, see {ref}`helm-sep-image`.

(k8s-rbac-clusters)=

## RBAC-enabled clusters

Kubernetes by default provides the *ClusterRole* `edit`. This role includes
all necessary permissions to deploy and work with Helm charts that consist of
common resources. SEP uses one additional custom type through use of the
`externalSecrets:` node. However, this custom resource definition must be
deployed to the cluster with its own chart. It is only required in specific use
cases, as in {ref}`this example <k8s-ex-rbac-enabled-clusters>`.
