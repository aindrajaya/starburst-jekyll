---
layout: default
title: Requirements
parent: Get Started
grand_parent: Deploy with Starburst Admin
nav_order: 1
permalink: /docs/deployment-mechanisms/starburst-admin/get-started/requirements
---
# Requirements for managed cluster nodes

Starburst Admin does not manage the cluster hardware, operating system
or package installation. It relies on the existence of all the nodes in
the cluster and the fact they fulfill the requirements detailed in this section.

Typically provisioning systems such as Puppet, Chef, Terraform and others are
used to prepare the cluster nodes. All cluster nodes need to fulfill the
normal {ref}`Starburst Enterprise platform (SEP) requirements <requirements>`,
including the following:

* RedHat Enterprise Linux
* Java runtime environment

Memory and hardware resource requirements depend on the planned capacity of the
cluster. Following are a few high level guidelines:

* Use identical hardware configurations for all workers.
* Start with at least two workers, scale up as needed.
* Prefer fewer, more powerful worker nodes over many smaller ones.
* For performance reasons, nodes are ideally located on the same subnet and
  within the same data center. All nodes communicate using TCP/IP.

Specific testing is performed with Red Hat Enterprise Linux (RHEL) versions 7,
8, and 9. Use other 64-bit Linux distributions at your own risk.

Additional requirements:

* [Enabled SSH access and connectivity from the control
  node](https://docs.ansible.com/ansible/latest/user_guide/connection_details.html),
  the configured user must have root or sudo access. If the sudo user requires a
  password, use `ask-become-pass` when running playbooks. Alternatively,
  Starburst Admin can be installed as a non-root user, as long as
  the non-root requirements are satisfied.
* `rsync`, often an optional package that needs to be installed.
* `bash`, typically installed by default.

The following requirements must be met in order to install {{sb}}
as a non-root user:

* The user that Ansible uses to SSH into the target nodes must already
  exist on the target node, prior to running any playbooks.
* The base directory that SEP is installed into, such as
  `/opt/starburst`, must already exist on the target node and the Ansible user
  must have read and write access to that directory.
* The directory variables in the `playbooks/vars.yml` file must reflect the
  SEP base directory, as described in the {ref}`Installation
  guide <create-minimal-configuration>`.

When using Starburst Admin with an RPM archive:

* An RPM-based Linux distribution is required.
* The `rpm` command, `yum`, `dnf`, and other similar commands are not required.
* RPM-based install is not allowed when installing Starburst Admin as
  non-root, because the directories it installs into require root access.

When Starburst Admin with an `tar.gz` archive:

* GNU `tar` command
* `unzip` command