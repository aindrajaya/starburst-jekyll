---
layout: default
title: SEP Docker container
parent: Local Installation
grand_parent: SEP Deployment Mechanisms
nav_order: 1
permalink: /docs/installation/docker-installation
---

# SEP Docker container

It is possible to run Starburst Enterprise platform (SEP) in a Docker container for initial exploration
and testing. The SEP Docker image `starburstdata/starburst-enterprise` is
available on [DockerHub](https://hub.docker.com/r/starburstdata/starburst-enterprise) and the
Starburst Harbor instance. It contains a full SEP distribution, including all
connectors and other features such as the {{oss}} CLI.

Out of the box, SEP container images provide a single node cluster with the
following catalogs configured:

- {doc}`blackhole </connector/blackhole>`
- {doc}`jmx </connector/jmx>`
- {doc}`memory</connector/memory>`
- {doc}`system</connector/system>`
- {doc}`tpcds </connector/tpcds>`
- {doc}`tpch </connector/tpch>`

## Using the Docker container in production

While it is possible to deploy SEP as a full cluster from the Docker container
by mounting separate sets of configuration files for the cluster, coordinator
and workers, it is not recommended. Instead, {{sb}} offers a full {doc}`Kubernetes
deployment option <../k8s>`, including Docker containers and Helm charts for
{doc}`Apache Ranger <../k8s/ranger-configuration>` and {doc}`Hive Metastore
<../k8s/hms-configuration>`.
