---
layout: default
title: Deploy with Starburst Admin
nav_order: 4
parent: SEP Deployment Mechanisms
permalink: /docs/starburst-admin
has_children: true
---

# Deploy with Starburst Admin

Starburst Admin is a
[collection](https://docs.ansible.com/ansible/latest/user_guide/collections_using.html#collections)
of [Ansible](https://www.ansible.com/) playbooks for installing and
managing Starburst Enterprise platform (SEP) or {{oss}} clusters.

Starburst Admin includes the following features:

* Installation and upgrade of Starburst Enterprise platform (SEP) or {{oss}}
  using the RPM or `tar.gz` archives
* Update the coordinator and worker nodes configuration files,
  including catalog properties files for data source configuration
* Service management of the cluster and on all nodes (start/stop/restart/status)
* Collection of logs and Java thread dumps
* Support for adding custom binary files, such as custom connectors or UDF
* Support for adding custom configuration files

Find more information about Starburst Admin in the following documentation:

:::{toctree}
:maxdepth: 1

starburst-admin/get-started
starburst-admin/installation-guide
starburst-admin/operation-guide
starburst-admin/update-guide
starburst-admin/playbook-reference
starburst-admin/release-notes
:::