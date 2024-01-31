---
layout: default
title: Install Starburst Enterprise in a Kubernetes cluster
parent: Kubernetes cluster design for Starburst Enterprise
grand_parent: Deploy with Kubernetes
nav_order: 2
permalink: /docs/kubernetes/kubernetes-cluster-design-for-starburst-enterprise/installation
---

# Installing Starburst Enterprise in a Kubernetes cluster

When {doc}`all requirements </k8s/requirements>` have been met, you
are ready to proceed with the initial installation or upgrade of Starburst Enterprise platform (SEP).

:::{admonition} Before you begin
This topic assumes that you understand the following key concepts and
components from other, related topics:

- {doc}`Starburst Enterprise deployment basics </installation/deployment>`
- {doc}`Helm chart deployments best practices </k8s/sep-configuration>`
- {ref}`Helm chart repository <k8s-helm-repository>`
- {ref}`Docker registry access <k8s-docker-registry>`

Familiarizing yourself with the concepts in the preceding list simplifies
management of credentials, installation, and subsequent updates.

Other relevant topics to review before proceeding are as follows:

- {ref}`Kubernetes cluster sizing requirements <k8s-cluster-requirements>`
- {ref}`Image pull secrets <k8s-ex-imagePullSecrets>`
- SEP {ref}`license file <helm-sep-license>`
:::

After you have completed the initial installation or upgrade as described in
this topic, you are ready to begin fine-tuning your SEP configuration.

(k8s-installation-overview)=

## Overview

The workflow to install SEP with Helm charts on your cluster is as follows:

- Establish access to the Helm chart repository and configure credentials to
  access the Docker registry in a registry access YAML file for use in all of
  your {{sb}} k8s deployments.
- Create a YAML file specific to each chart and cluster, for example,
  `sep-prod-setup.yaml` for your production cluster.
- Ensure your Helm/kubectl configuration points at the correct cluster with
  `kubectl cluster-info`.
- Run Helm to install the chart.
- Access the cluster and check for success.

Each Helm chart includes a `values.yaml` file that sets a reasonable set of
default values. This default setup does not include catalog definitions and
others that are necessary for your cluster.

You have to change your values file to add or update any configuration and run a
Helm upgrade to apply the changes to the cluster.

Iterate on the configuration in the values YAML file with minimal setup until
you have a working system. Depending on your cluster, you have to adjust memory
requirements for the worker and coordinator and other settings. Inspect the
message with `kubectl` or Octant to determine details.

After you've achieved a running cluster, ensure to store the values YAML file as
a reference and then add more details as required for the specific cluster need.

(k8s-installation-step-by-step)=

## Installing SEP

The SEP installation is managed with the `starburst-enterprise` Helm chart.
Installation and upgrades are done with the `helm upgrade` command with the
following options at a minimum:

- the `--install` flag used with the `helm upgrade` command allows you to
  consistently use the same command to both install and upgrade SEP
- a YAML file with your registry access credentials, discussed later in this
  topic
- a minimal YAML configuration file with the memory and CPU resource
  configurations for `coordinator:` and `worker:` that reflect your
  cluster's resources, for example, `sep-prod-setup.yaml`
- a YAML file with one or more {doc}`catalogs defined </catalogs>`, for example,
  `sep-prod-catalogs.yaml`

The following example command assumes the registry access file, the production
cluster configuration file, and the catalog configuration file are located in
the current directory:

```shell
$ helm upgrade  my-sep starburstdata/starburst-enterprise \
    --install \
    --version 434.0.0 \
    --values ./registry-access.yaml
    --values ./sep-prod-setup.yaml
    --values ./sep-prod-catalogs.yaml
```

The version value is available from the Helm repository.

The default values result in one coordinator and two worker nodes in the
cluster, with very specific memory and CPU resources that likely will not match
your cluster's specifications. We **strongly suggest** that you initially
install SEP with the minimal changes needed to reflect your cluster's memory
and CPU resources; then make small, focused customizations to suit your
organization's needs.

The following sections describe the initial installation and how to begin
customizing SEP.

(minimal-values-yaml-file)=

### Create this one file before you begin

No matter what other configurations you need for your deployments, create your
`registry-access.yaml` file for reuse across all clusters and charts. This
helps to ensure a smooth install and deployment experience.

In your `registry-access.yaml` file, add the following to configure access for
{{sb}}'s Harbor registry:

```yaml
registryCredentials:
  enabled: true
  registry: harbor.starburstdata.net/starburstdata
  username: <yourusername>
  password: <yourpassword>
```

The contents of this file override the default, empty values and ensure that the
Helm charts can download the required Docker containers.

If you have multiple clusters, this same file is used for all of them. You can
also use the same file for the optional Ranger and Hive Metastore Service
charts. Other configurations should be managed in separate files as per
best practices.

As an alternative to using a username and password directly, you can use a
Kubernetes secret by creating a secret containing the access token for your
registry and configuring your pod to {doc}`use the secret
</starburst-admin/operation-guide>`.

(k8s-private-registries)=

#### Using private registries and repositories instead of {{sb}} Harbor

Typically, you must use your username and password for accessing the Helm chart
repositories and the Docker registry on the Starburst Harbor instance. You can
instead {ref}`use private Docker and Helm chart repositories
<k8s-ex-private-registries>`.

### Initial installation checklist

The following checklist describes the initial installation process:

1. Gather repository credentials for the Helm chart repository and the Docker
   registry provided by {{support}}.
2. Create the `registry-access.yaml` file to override the default,
   empty values.
3. Create your correctly-sized Kubernetes cluster.
4. Ensure your Helm/`kubectl` configuration points at the correct cluster
   with `kubectl cluster-info`.
5. Add your license file.

:::{note}
We **strongly suggest** using a shared secret to add the license file.
:::

6. Create a minimal YAML configuration file with the memory and CPU resource
   configurations for `coordinator:` and `worker:` that reflect your
   cluster's available resources, for example `sep-prod-setup.yaml`.

:::{warning}
**Do not skip this step.** The default values for memory and CPU resources
likely vary significantly from your cluster's available resources. If you
attempt to run SEP with the defaults, SEP may not start.
:::

7. Run Helm to install the default chart, as well as any override YAML files
   using the `--values` argument, as shown in the following example:

```shell
$ helm upgrade sep-prod-cluster starburstdata/starburst-enterprise \
    --install \
    --version 434.0.0 \
    --values ./registry-access.yaml \
    --values ./sep-prod-setup.yaml
```

8. Determine the IP address or the DNS hostname of the coordinator by running
   the `kubectl get pods` command.
9. Use the IP address or hostname to verify the coordinator is running by
   accessing the {doc}`Starburst Enterprise web UI </overview/sep-ui>`.
   You can use the same information to connect with the CLI or the JDBC driver.

(k8s-updates)=

## Updating to a new release

If you have created focused, well-managed override files following our best
practices guide, the upgrade process is a straightforward Helm-based process. As
with any enterprise-scale application, we do recommend that you test upgrades
from one release to another using a test cluster. This allows you to catch any
configuration changes and update Helm charts before deploying into production:

1. Review the {doc}`Helm charts release notes <./release-notes>` for any
   relevant changes that affect your override files. For example, there may be
   new configuration options to add, or deprecated properties to remove.
2. Review the SEP {doc}`release notes <../release>` for new capabilities
   and breaking changes.
3. Create a backup of your {doc}`backend service database </admin/backend-service>`.
   Consult the documentation for the particular database that you use.
4. Run Helm with the updated Helm chart version and with the updated YAML
   configuration files, as in the following example:

```shell
$ helm upgrade my-sep-staging-cluster starburstdata/starburst-enterprise \
    --install \
    --version 434.0.0 \
    --values ./registry-access.yaml \
    --values ./sep-stage-setup.yaml
```

:::{warning}
{{sb}} does not support downgrading to older versions of SEP. Downgrading
may lead to incompatible schemas with the backend service database, data
loss, and unexpected issues.
:::

## Operating SEP in a Kubernetes cluster

Once your cluster is up and running, you can connect your clients like the CLI
or a BI application using the JDBC driver.

With that usage you gain insights on the correct configuration, data sources,
and sizing of the cluster for the desired usage and performance. This in turn
leads you to further work such as updating configuration, scaling your cluster,
or even upgrading to a newer release.

This topic covers these and other aspects associated to running your SEP
cluster, or even clusters.

### Running SEP

Once you have {ref}`successfully installed <k8s-installation-step-by-step>`
SEP, you can confirm that the coordinator pod is running with tools such as
`kubectl` or Octant.

You can also check the log in the server pod. A successful start of the
coordinator, and also each worker, shows with a log entry similar to the
following snippet:

```shell
INFO  main    io.trino.server.Server  ======== SERVER STARTED ========
```

With these checks performed, you can access the Starburst Enterprise {doc}`web UI
</overview/sep-ui>` on the coordinator. Determine the FQDN
and port of the coordinator as configured in the {ref}`expose section of the
Helm chart <helm-sep-expose>`.

A default deployment exposes the UI via HTTP and does not require a password.
Production deployments should use HTTPS and have authentication configured.

The main page of the web UI displays the number of connected workers. Confirm
that the number is identical to the number specified by the `worker.replicas`
value of the {ref}`worker section of the Helm chart <k8s-defaults-worker>`. The
default number of workers is two.

### Updating and upgrading

From time to time you need to make incremental changes to your configuration or
move to a new release. To do so:

- Create or update the correct YAML files with the new or changed
  configuration properties.
- Run the {ref}`helm upgrade <k8s-updates>` command with the updated version of
  the chart.
- Verify that the desired changes are applied successfully. We suggest making
  changes to one area of the application at a time.
- Repeat the process with the next desired change.

Typically Helm applies the relevant changes in a granular fashion. For example,
if you change the worker configuration, no coordinator changes are performed and
the coordinator continues to run while workers are updated.

If you need to ensure that pods are recreated completely, you can use the
`--recreate-pods` option for your `helm` command. This essentially restarts
all workers as well as the coordinator and therefore restarts the cluster. This
includes a downtime for users.

(k8s-scaling)=

### Scaling

You can scale your SEP cluster manually by adding more workers.
This is done by updating {ref}`the worker replicas <helm-sep-workers>` and
updating your deployment. Sufficient resources available in the cluster is a
requirement.

Alternatively, you can enable autoscaling in your worker configuration and
updating the deployment. Sufficient resources available in the cluster is a
requirement.

You can also increase the memory resources allocated to the workers and updating
your deployment, if sufficiently sized pods are available and you recreate pods
as part of the update.

## Next steps

- {doc}`Customize SEP <sep-configuration>`.
- {doc}`Define and configure catalogs <../catalogs>`.
- Learn about {doc}`operating your cluster <installation>`.
- Install and configure [a
  metastore](../../introduction/metastores.html){.external} if you use object
  storage, the {{sb}} caching service, or data products.
- \[Optional\] Install and configure {doc}`Ranger <ranger-configuration>`.
- \[Optional\] Install and configure the {{sb}} {doc}`cache service
  <cache-service-configuration>`.
