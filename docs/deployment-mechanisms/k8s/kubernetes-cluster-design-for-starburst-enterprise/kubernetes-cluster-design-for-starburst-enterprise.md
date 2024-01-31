---
layout: default
title: Kubernetes cluster design for Starburst Enterprise
parent: Deploy with Kubernetes
grand_parent: SEP Deployment Mechanisms
nav_order: 1
permalink: /docs/kubernetes/kubernetes-cluster-design-for-starburst-enterprise
has_children: true
---

# Kubernetes cluster design for Starburst Enterprise

SEP by its nature is built for performance. How it operates differs from other
typical applications running in Kubernetes.

Typically, an enterprise application comprises many stateless microservices,
each of which can be run on a small instance. SEP's exceptional
performance comes from its powerful query optimization engine, which expects all
nodes to be identically sized for query planning. It also depends on each node
to have large amounts of memory, to allow parallel processing within a node as
well as processing of large amounts of data per node.

By default, once work is divided up among worker nodes, it is not redirected if
a node dies, as this can impact performance gains. SEP coordinator and worker
nodes are therefore stateful, and clusters by design rely on fewer, larger
nodes. However, {doc}`fault-tolerant execution
</admin/fault-tolerant-execution>` is available as an optional feature that
redirects work to other nodes in order to offer more resiliency for larger batch
operations.

Ideally SEP runs within a namespace dedicated to it and it alone.
Separate pods can be defined for worker nodes and coordinator nodes in that
namespace, and taints and tolerations can be defined for node selection in
SEP.

Review the SEP k8s {doc}`requirements </k8s/requirements>` before you begin
installing the Helm charts to ensure that you have the correct credentials in
place and understand sizing requirements. After you have prepared your cluster,
cluster networking, and learned about configuration options using our getting
started topics, you are directed to the following topics: