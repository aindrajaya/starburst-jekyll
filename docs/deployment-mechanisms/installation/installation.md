---
layout: default
title: Local Installation
parent: SEP Deployment Mechanisms
nav_order: 5
permalink: /docs/installation
has_children: true
---

# Local installation

% comment:
% NOTE: This document overrides the Trino version.
% Reason: Custom content and structure to break up server and client side
% installation stuff

Using Starburst Enterprise platform (SEP) can involve a fresh installation and deployment of one or
multiple {{oss}} server instances on a cluster. If you already have access to
a running cluster and are a user and just want to connect to the cluster, you
need to install and configure your client application of choice.

:::{note}
We strongly recommend following the guidance described in the
{doc}`Kubernetes </k8s>` or {doc}`Starburst Admin </starburst-admin>`
documentation as simpler deployment options. A manual installation is only
recommended for evaluation purposes.
:::

## Server and cluster installation and deployment

Download SEP and read about the manual deployment option:

```{toctree}
:maxdepth: 1

installation/download
```

An SEP server or a full cluster of SEP servers can be installed and
deployed on numerous systems. Choose the best installation guide for your
environment from the following:

```{toctree}
:maxdepth: 1

installation/rpm
installation/docker
installation/kubernetes
installation/containers
```

General information for configuration files, directories, launcher script, and
more is applicable to {{oss}} and SEP and all deployment methods:

```{toctree}
:maxdepth: 1

installation/deployment
installation/query-resiliency
```

