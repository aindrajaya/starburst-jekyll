---
layout: default
title: Get Started
parent: Deploy with Starburst Admin
grand_parent: SEP Deployment Mechanisms
nav_order: 1
permalink: /docs/deployment-mechanisms/starburst-admin/get-started/
has_children: true
---

# Getting started

Starburst Admin is a
[collection](https://docs.ansible.com/ansible/latest/user_guide/collections_using.html#collections)
of [Ansible](https://www.ansible.com/) playbooks for installing and
managing Starburst Enterprise platform (SEP) or Trino clusters.

:::{note}
Current release: Starburst Admin 1.8.0.
:::

Starburst Admin includes the following features:

* Installation and upgrade of Starburst Enterprise platform (SEP) or Trino
  using the RPM or `tar.gz` archives
* Update the coordinator and worker nodes configuration files,
  including catalog properties files for data source configuration
* Service management of the cluster and on all nodes (start/stop/restart/status)
* Collection of logs and Java thread dumps
* Support for adding custom binary files, such as custom connectors or UDF
* Support for adding custom configuration files

All target machines must meet the {ref}`requirements <sb-admin-requirements>`
outlined below prior to installing Starburst Admin.

Starburst Admin does not manage the creation of the servers, the
operating system installation and configuration, or the Python and Java
installation. It is also not designed to manage other related tools such as
Apache Ranger, a Hive Metastore Service, or any data source.

It is most suitable for managing clusters installed on
[bare-metal](../../glossary.html#bare-metal){.external} servers or [virtual
machines](../../glossary.html#virtual-machine){.external}. Use the
{doc}`Kubernetes with Helm support </k8s>` instead of Starburst Admin if you use
containers and Kubernetes.

:::{note}
The legacy Presto Admin is deprecated, and no longer supported for Starburst
Enterprise version 354-e and higher.
:::

(sb-admin-requirements)=

## Requirements

Deep knowledge of Ansible is not expected for usage, but familiarity with
Ansible and Ansible playbooks is required.

The following sections detail the requirements for the machine where you run
Ansible and Starburst Admin, called the *control node*, and the requirements for
the machines where you install and manage SEP, called the
*cluster nodes*.

### Requirements for the control node

The [control
node](https://docs.ansible.com/ansible/latest/network/getting_started/basic_concepts.html)
is used to run Starburst Admin, and therefore Ansible playbooks.
[Standard Ansible
requirements](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#prerequisites)
apply:

* Ansible 2.10 or higher
* Linux/Unix operating system
* Python 3.5 and higher

In addition, the following resources are needed:

* x86_64 (AMD64) or AArch64 (ARM64) processor architecture
* [SSH connectivity to the cluster nodes](https://docs.ansible.com/ansible/latest/user_guide/connection_details.html)
* Downloaded Starburst Enterprise platform (SEP) or Trino `tar.gz` or
  {doc}`RPM </installation/rpm>` archive files on the control
  node, or alternatively URL to the files that is accessible on all cluster
  nodes

The controller node can be any machine that is configured to fulfill these
requirements. For initial testing you can use your workstation or even a node in
the cluster directly. Production usage should follow Ansible best practices, and
use dedicated workflow or Ansible orchestration and automation tools such as
[Ansible Tower](https://www.ansible.com/products/tower) or
[Concord](https://concord.walmartlabs.com/).



## Install Starburst Admin on the control node

Starburst Admin is a collection of Ansible playbooks that you install on
the control node:

* Contact Starburst Support for the Starburst Admin `tar.gz` binary
  package. Alternatively, if you have access to the [Starburst Admin
  repository](http://software.starburstdata.net/#starburst-admin/),
  download the `tar.gz` file for the latest release tag.
* Move it onto the control node into any directory, such as `~/tmp`.
* Access the directory in a command line interface.
* Install the collection with the following command:
  ```shell
  ansible-galaxy collection install starburst-admin-*.tar.gz
  ```
* Confirm the command finishes successfully:
  ```shell
  Starting galaxy collection install process
  Process install dependency map
  Starting collection install process
  Installing 'starburst.admin:1.8.0' to '....'
  starburst.admin:1.8.0 was installed successfully
  ```

The collection is installed into `/home/<username>/.ansible/collections` by
default. The installation path `/home/<username>/.ansible/collections/ansible_collections/starburst/admin/files`
is used for the binaries and all the configuration files for a cluster. Make
sure you manage the files in this directory with a version control system.

You can override the installation path with the option `-p <installation-path>`.

### Install on multiple control nodes

If you need to install the collection into numerous control nodes, you can make
the binary available on a remote URL:

* Make the binary available on a server via HTTP, for example,
  `https://repo.example.com/files/starburst-admin-1.8.0.tar.gz`.
* Create a file `requirements.yml` that includes a link to the binary.
  ```yaml
  ---
  collections:
      # Example link to tar.gz package
      - https://repo.example.com/files/starburst-admin-1.8.0.tar.gz
  ```
* Use the YAML file for the installation
  ```sh
  ansible-galaxy collection install -r requirements.yml
  ```

## Next steps

Now that you have set up the control nodes and the managed cluster nodes, you
can proceed with the {doc}`initial installation on the
cluster <installation-guide>`. Before proceeding with the installation,
review the additional services that may be required, as discussed in the
following sections.

### Automated memory configuration

The amount of memory that SEP consumes can be fine-tuned by adjusting
settings in the {ref}`jvm.config <cache-service-jvm-config>` and
{doc}`config.properties </admin/properties-resource-management>` files.
Starburst Admin has a {ref}`memory_auto_config <pb-auto-mem-config>` parameter
section which provides a convenient way to set these memory-related
configurations for you, based on a few questions.

### Hive Metastore Service or AWS Glue

SEP must be configured to work with an existing Hive Metastore
Service or AWS Glue if any of the following are true:

* If one of your catalogs such as {doc}`Hive </connector/hive>`, {doc}`Delta Lake </connector/starburst-delta-lake>`,
  or {doc}`Iceberg </connector/starburst-iceberg>` needs a metastore configured.
* You are configuring the {doc}`cache service </admin/cache-service>`.
* You are enabling {doc}`data products </data-products/configuration>`.

### Backend service

You must enable and configure the SEP backend service, which requires an
externally-managed database to be available. Read more about this in our
{doc}`backend service topic </admin/backend-service>`.

### Cache service

The {{sb}} cache service provides the ability to configure and
automate the management of table scan redirections and materialized views in
supported connectors. It is disabled by default.

All Starburst Admin deployments enabling the cache service must use the
cache service in {ref}`embedded mode <cache-service-embedded>`.

### Data products

Data products provides a collection of curated, high-quality related datasets
and relevant metadata for important data in your organization. You must
configure SEP to {doc}`enable data products </data-products/configuration>`.

### Insights

Recent query and cluster activity are enabled by default; however, you must
explicitly configure SEP to enable
{ref}`query history and usage metrics <insights-configuration>`.
