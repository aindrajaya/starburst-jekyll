---
layout: default
title: Operation Guide
parent: Deploy with Starburst Admin
grand_parent: SEP Deployment Mechanisms
nav_order: 3
permalink: /docs/deployment-mechanisms/starburst-admin/opeartion-guide/
---

# Operation guide

After satisfying the {ref}`initial requirements <sb-admin-requirements>` for the
control node and all the managed nodes in the cluster, you can get a cluster up
and running with the help of the {doc}`installation guide <installation-guide>`.

A fully configured deployment requires a bit more configuration and work. It is
typically performed incrementally and you use Starburst Admin to help you for a
number of scenarios. They range from simple tasks like changing a configuration
in a catalog file to more complex ones such as adding a new catalog file,
scaling the cluster up or down, or adding security configuration.

The following sections includes these and many other scenarios and refers to the
{doc}`playbook reference <playbook-reference>` and other sections as necessary.

## Targeting specific hosts

By default, Ansible runs the tasks defined in the playbooks on all cluster nodes
defined in the inventory `hosts` file.

You can target specific host groups or even specific hosts with the `-l` option.
The example hosts files uses the groups `coordinator` and `worker` to target the
groups. Individual hosts are targeted with the IP address.

**Example: Roll out configuration changes to workers only**

* Update the relevant configuration files in `files/worker`, for example changed
  JVM configuration.
* Push the configuration only to the workers
  ```
  ansible-playbook -l worker playbooks/push-configs.yml
  ```
* Restart only the workers
  ```
  ansible-playbook -l worker playbooks/restart.yml
  ```

**Example: Update security configuration on the coordinator**

* Update the relevant configuration files in `files/coordinator`, for example
  updated certificate files.
* Push the configuration only to the coordinator
  ```
  ansible-playbook -l coordinator playbooks/push-configs.yml
  ```
* Restart only the coordinator
  ```
  ansible-playbook -l coordinator playbooks/restart.yml
  ```

**Example: Restart or remove a misbehaving worker only**

* Determine the worker that has problem, for example from {ref}`collecting the
  logs <pb-collect-logs>` or {ref}`checking the service status <pb-check-status>`
* Attempt a restart of the one worker that seems to have problems
  ```
  ansible-playbook -l 172.28.0.3 playbooks/restart.yml
  ```
* Check status after restart
  ```
  ansible-playbook -l 172.28.0.3 playbooks/check-status.yml
  ```
* Check the logs after restart
  ```
  ansible-playbook -l 172.28.0.3 playbooks/collect-logs.yml
  ```

## Adding, removing and updating catalogs

You can add a new catalog, update an existing catalog or even remove a catalog
and roll that change out across the cluster with the following steps:

* Perform the desired changes in the `files/catalog` directory.
* {ref}`Stop the cluster <pb-stop>`.
* {ref}`Push the configuration <pb-push-configs>`.
* {ref}`Start the cluster <pb-start>`.

A full cluster restart of the coordinator and all workers is necessary to apply
the updated configuration.

## Changing other configuration

* Perform the desired changes in the `files` subdirectories and files.
* {ref}`Stop the cluster <pb-stop>` or selectively the
  coordinator or all workers.
* {ref}`Push the configuration <pb-push-configs>`.
* {ref}Start the cluster <pb-start>` or selectively the
  coordinator or all workers..

The safest option is to restart the complete cluster. Depending on the
configuration changes it can be possible to just restart all the workers or the
coordinator only. For example, for authentication configuration to
SEP a coordinator restart can be sufficient.

## Upgrading Starburst Enterprise

Upgrading SEP using Ansible is similar to initial installation. The process has
to be performed without any users running queries.

Use the following steps to perform the update:

* Download the binary package for the new version and place it in `files`.
* Alternatively place the binary package on a server and update the installer URL.
* Update the version number in `vars.yml`.
* Update all the configuration and add any new configuration files.
* {ref}`Stop the cluster <pb-stop>`.
* {ref}`Install the new packages to the cluster <pb-install>`.
* {ref}`Push the configuration <pb-push-configs>`.
* {ref}`Start the cluster <pb-start>`.

```
ansible-playbook playbooks/stop.yml
ansible-playbook playbooks/install.yml
ansible-playbook playbooks/push-configs.yml
ansible-playbook playbooks/start.yml
```

To rollback the upgrade, revert any configuration changes, change the value
of `version` to the previous one, and execute the same playbooks above.

Note that because all nodes in the cluster must use the same version,
it is not possible to perform a rolling restart and avoid interruptions.
To minimize down time, we recommend installing the new version on a set
of new hosts, and reconfigure clients to connect to this cluster or update
DNS entries to point to it. Old cluster can be decommissioned after all queries
running on it are done.

To manage separate installations on more than one sets of hosts, copy
the `hosts` file and update values inside it. Then, to use the new file,
add the `-i <new-hosts>` parameter when running the `ansible-playbook` command.

### Blue/Green deployments and upgrades

With sufficient resources, you can significantly improve your upgrade process by
managing two production clusters behind a load balancer (LB) with a defined
fully qualified domain name (FQDN). The upgrade process can then follow a
blue-green deployment process:

1. The current production cluster in use is *blue*, and the LB uses the FQDN to
  point users to it.
2. Update the configuration and resources for the inactive *green* cluster to the
  new version and configuration.
3. Install and push the configuration, then start the *green* cluster.
4. Test the *green* cluster with the direct IP address or an alternative FQDN
  configured on the LB.
5. Switch the LB configuration to point the *green* cluster.
6. Shut down the *blue* cluster.

For the next upgrade, the process is identical, but the roles of the two
clusters are reversed.

The inactive cluster can be kept running for failover and high availability
usage, or decommissioned and recreated again for future upgrades.

## Adding or removing workers

You can use Starburst Admin to help with scaling your cluster up and
down by adding or removing workers with the following steps.

**Scaling up**

* Provision the new machine with {ref}`necessary requirements for cluster
  nodes <sb-admin-requirements>`.
* Add the IP address and access details to the `worker` section in the `hosts`
  file.
* Install on the new node
* Push configuration to the new node
* Start the new node
* Check the log of the new node

For example, if you add the new host at `172.28.0.5` as worker you can run the
installation just to this node:

```
[coordinator]
172.28.0.2 ansible_user=root ansible_password=changeme

[worker]
172.28.0.3 ansible_user=root ansible_password=changeme
172.28.0.4 ansible_user=root ansible_password=changeme
172.28.0.5 ansible_user=root ansible_password=changeme
```

Playbook invocations:

```
ansible-playbook -l 172.28.0.5 playbooks/install.yml
ansible-playbook -l 172.28.0.5 playbooks/push-configs.yml
ansible-playbook -l 172.28.0.5 playbooks/start.yml
```

Specifying the host is optional and can be omitted since the playbooks can run
without side effects if the desired state is already reached. As a result you
can add a number of workers and then just install, push configuration and start
them all together.

**Scaling down**

* Stop the worker or perform a graceful shutdown.
  ```
  # Hard stop
  ansible-playbook -l 172.28.0.5 playbooks/stop.yml

  # Graceful shutdown
  ansible-playbook -l 172.28.0.5 playbooks/graceful-shutdown.yml
  ```
* Optionally run {ref}`uninstall <pb-uninstall>` for the specific worker only
  ```
  ansible-playbook -l 172.28.0.5 playbooks/uninstall.yml
  ```
* Remove the worker from `hosts`


## Configuring TLS

You can configure {doc}`TLS for the coordinator </security/tls>` as
well as {doc}`cluster internal authentication and
TLS </security/internal-communication>` as usual.

For the functionality of the playbooks you need to additionally update the ports
configured in `vars.yml` so that they can continue to interoperate with the
nodes for status checks and other aspects.

Using 8443 on the coordinator and workers:

```
coordinator_port: 8443
worker_port: 8443
```

## Using secrets

You can use {doc}`secrets in environment variables </security/secrets>` to avoid
clear text password and other sensitive data in the configuration files.

To inject a secret value into an environment variable, you can modify the
`env.sh` shell scripts in `files/worker` and `files/coordinator`. The script is
executed before SEP is started. For example, you can insert code to retrieve
secret values from a secret manager such as [Hashicorp
 Vault](https://www.hashicorp.com/products/vault),
[Keywhiz](https://github.com/square/keywhiz), or a secret manager
from your cloud provider.
