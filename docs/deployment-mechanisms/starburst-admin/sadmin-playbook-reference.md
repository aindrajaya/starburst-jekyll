---
layout: default
title: Playbook Reference
parent: Deploy with Starburst Admin
grand_parent: SEP Deployment Mechanisms
nav_order: 4
permalink: /docs/deployment-mechanisms/starburst-admin/playbook-reference/
---

# Playbook reference

Starburst Admin includes numerous playbooks to perform specific actions.
You can use them individually or in sequence to satisfy the needs of your
specific use case for {doc}`installing <installation-guide>` and
{doc}`operating <operation-guide>`.

The following playbooks are available:
<!-- no toc -->
* {ref}`Install cluster <pb-install>`
* {ref}`Push configuration <pb-push-configs>`
* {ref}`Start cluster <pb-start>`
* {ref}`Stop cluster <pb-stop>`
* {ref}`Restart cluster <pb-restart>`
* {ref}`Check service status <pb-check-status>`
* {ref}`Graceful worker shutdown <pb-graceful-shutdown>`
* {ref}`Rolling worker restart <pb-rolling-restart-workers>`
* {ref}`Collect logs <pb-collect-logs>`
* {ref}`Collect a Java thread dump <pb-collect-jstacks>`
* {ref}`Uninstall cluster <pb-uninstall>`

(pb-install)=

## Install cluster

The `install` playbook installs the RPM or tarball package on all defined hosts.
You are required to place the `tar.gz` or `rpm` file in the `files` directory
and specify the `version` in `vars.yml` to run the playbook:

```shell
ansible-playbook playbooks/install.yml
```

Errors occurs if RPM and tarball are found or if version values mismatch.

Alternatively to the archive in the `files` directory on the control machine,
you can set the `installer_url` in `vars.yml` to point to HTTP or HTTPS URL of
the  `tar.gz` or `rpm`. Comment out the `installer_file` in `vars.yml`. Ansible
then downloads the binary from the URL directly on the hosts. This approach is
more complex to set up, but scales better since all hosts can download from the
URL in parallel. The hosts need to be able to contact the specified URL.

The playbook verifies the availability of the required Java runtime on the host,
starting first at the value of `JAVA_HOME` and then looking at common
installation paths. It uses the script `files/find_java.sh` and fails if no
runtime is found.

The playbook installs the RPM or unpack the tarball and create the necessary
directories specified via `vars.yml`.

Optionally, all playbooks read a custom `vars.yml` file by passing a `vars_yml`
parameter as an extra variable (-e), as in this example:

```shell
ansible-playbook playbooks/install.yml -e "vars_yml=/mypath/my-vars.yml"
```

The `local_files` variable in the `vars.yml` file defines where the playbooks
reads the configuration files from before copying them to the target. A relative
path is assumed to be relative to the playbooks directory, or an absolute
path can be used to source the files from any directory.

Ansible can operate as root (default), or as a non-root user.
For root privileges, `sudo` commands are used to elevate privileges.
Since `sudo` prompts for the user password, run every `ansible-playbook` command
with the `--ask-become-pass` parameter as detailed in the [become command
documentation](https://docs.ansible.com/ansible/latest/user_guide/become.html#become-command-line-options).

The RPM installation automatically adds a dedicated system user to run SEP
as a service. This user owns all configuration files, which should not be
readable by other users on the hosts. The tarball installation uses the
`installation_owner` and `installation_group` defined in `vars.yml`. The install
playbook create the user and group.

SEP depends on Linux's limit of the [number of open
files](https://docs.oracle.com/cd/E19623-01/820-6168/file-descriptor-requirements.html).
For root installs, this playbook automatically sets the limit to an
appropriate value. Non-root installs cannot change the open file limit, so this
playbook instead displays a warning message at the end of the playbook
if the current limit is inadequate. Use the `ulimit_nofile_min` variable
`vars.yml` to define the minimum value. For root installs, this same variable
is used when automatically setting the value.

(pb-push-configs)=

##  Push configurations

The `push-configs` playbook generates the configuration files for the
coordinator and workers and distributes them to all hosts.

The following resources are used to create the set of configuration files:

:::{list-table}
:widths: 30, 50
:header-rows: 1

* - File name and relative path
  - Description
* - `files/vars.yml`
  - Contains variable definitions to use in the configuration files that are
    templatized with Jinja2. These files use the `.j2` extension. Ansible uses
    the set values and replaces the variable placeholders. For example, the
    placeholder `{% raw %}{{ node_environment }}{% endraw %}` is replaced with the
    value `production` from `node_environment: production` [catalog properties
    files](/catalogs).
* - `files/coordinator`
  - Contains the configuration files for the coordinator.
* - `files/worker`
  - Contains the configuration files used for all workers.
* - `files/catalog`
  - Contains the [catalog properties files](/catalogs), which define data source
    connections.
* - `files/extra/etc`
  - Includes additional files such as the {doc}`license file</license-requirements>`, as
    well as other configuration files.  These files are placed in the directory
    specified by the `etc_directory` Ansible playbook variable on all nodes.
* - `files/extra/lib`
  - Additional files that are placed in the `lib` directory on all nodes.
    Typically, these are binary files or configuration files that are added to
    the `classpath`, such as user-defined functions.
* - `files/extra/plugin`
  - Additional files that are placed in the `plugin` directory on all nodes.
    Typically, these are complete directories with binaries, used for
    custom-developed plugins, such as a security extension or a custom
    connector.
:::

Any changes, including deletion of catalog files is synchronized to the hosts.

Starburst Admin automatically uses the folder name `starburst` for
SEP deployments and uses `trino` for {{oss}}
deployments.

Other file deletions are not synchronized but can be performed with the [Ansible
file
module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/file_module.html).
For example, if you made a file `files/extra/etc/foo.sh`, the file is copied
into `/etc/starburst/foo.sh` on the hosts. You can use the following
command to delete it:

```shell
ansible all -m file -a "path=/etc/starburst/foo.sh state=absent"
```

The supported files and configuration methods are {doc}`identical to the general
configuration files and properties from SEP </index>` for the identical
version you
deploy with Starburst Admin.

After you created or updated all the configuration as desired, you can run the
playbook with the following command:

```shell
ansible-playbook playbooks/push-configs.yml
```

Keep in mind that most of the configuration files need to be distributed to all
nodes in the cluster and that they need to be identical for all workers.

The best approach to apply any new configuration involves the following steps:

* Ensure no users are active and {ref}`stop <pb-stop>` the cluster.
* Alternatively {ref}`shut it down gracefully<pb-graceful-shutdown>`.
* Update the configuration files.
* Push the configuration.
* {ref}`Start <pb-start>` the cluster.
* Verify the changes work.

(pb-auto-mem-config)=

### Automated memory configuration

For SEP versions 370-e and above, the `push-configs` playbook
optionally sets up SEP's memory configuration properties for you
using the `memory_auto_config` section of the `vars.yml` file. The values you
set are dependent upon the resources available in your cluster nodes:

In this example which assumes instances with 256GB of memory, automated memory
configuration is enabled, and the `total_memory` set at 245GB:

```yaml
# Memory Auto Configuration parameters
memory_auto_config:
  use_auto_config: yes
  max_concurrent_queries: 3
  coordinator:
    total_memory: 245GB
    node_memory_headroom: 2GB
    heap_size_percentage: 90
    heap_headroom_percentage: 30
  worker:
    total_memory: 245GB
    node_memory_headroom: 2GB
    heap_size_percentage: 90
    heap_headroom_percentage: 30
```

:::{note}
We suggest using the default values provided in
the `vars.yml` file for `heap_size_percentage` and `heap_headroom_percentage`.
:::

The following table describes the available parameters in the
`memory_auto_config` YAML section:

:::{list-table}
:widths: 30, 50, 50
:header-rows: 1

* - Parameter name
  - Description
  - Required?
* - `use_auto_config`
  - Disabled by default. When set to `yes`, the `push-configs` playbook is
    enabled, and requires valid settings for all memory configuration
    parameters.
  -
* - `max_concurrent_queries`
  - Defines the maximum number of queries allowed to run simultaneously.
  - When `use_auto_config` is set to `yes`, this must be set.
* - `total_memory`
  - Defines the amount of memory available on your cluster nodes nodes for
    SEP to use. This must be expressed in a human-readable quantity such as
    `30GB` or `100MB`.
  - When `use_auto_config` is set to `yes`, this must be set for both the
    `coordinator` and `worker` YAML nodes.
* - `node_memory_headroom`
  - Defines the amount of memory to keep open beyond what SEP can use. This
    must be expressed in a human-readable quantity such as `30GB` or `100MB`.
  - When `use_auto_config` is set to `yes`, this must be set for both the
    `coordinator` and `worker` YAML nodes.
* - `heap_size_percentage`
  - Defines the percentage of the node's memory to assign to the Java heap. This
    must be expressed as an integer between 0 and 100.
  - When `use_auto_config` is set to `yes`, this must be set for both the
    `coordinator` and `worker` YAML nodes.
* - `heap_headroom_percentage`
  - Defines the percentage of the Java heap to not be tracked by SEP. This
    must be expressed as an integer between 0 and 100.
  - When `use_auto_config` is set to `yes`, this must be set for both the
    `coordinator` and `worker` YAML nodes.
:::

When `use_auto_config` is set to "yes", `push-configs` uses the memory
parameters to automatically calculate the following values:

:::{list-table}
:widths: 30, 30, 50
:header-rows: 1

* - Name
  - Configuration file
  - Calculation
* - `-Xmx`
  - {ref}`jvm.config <cache-service-jvm-config>`
  - Set to `total_memory` * `heap_size_percentage` / 100 -
    `node_memory_headroom`.
* - `-Xms`
  - {ref}`jvm.config <cache-service-jvm-config>`
  - Set to the same value as `-Xmx`.
* - `memory.heap-headroom-per-node`
  - {doc}`config.properties </admin/properties-resource-management>`
  - Set to (`-Xmx ` * `heap_headroom_percentage`) / 100.
* - `query.max-memory-per-node`
  - {doc}`config.properties </admin/properties-resource-management>`
  - Set to (`-Xmx` \ - `memory.heap-headroom-per-node`) /
    `max_concurrent_queries`.
* - `query.max-memory`
  - {doc}`config.properties </admin/properties-resource-management>`
  - Automatically set to `1PB` to limit the total memory consumed on very large
    clusters.
:::

(pb-start)=

## Start cluster

The `start` playbook start the coordinator and all worker processes on the
hosts. It starts SEP the `service` command for RPM-based
installations or with the `launcher` script for tarball installation.

You need to {ref}`install the cluster <pb-install>` and {ref}`push the
configuration <pb-push-configs>` before starting the cluster the first time.

```shell
ansible-playbook playbooks/start.yml
```

The playbook relies on the hosts to be up and running and available to Ansible.
When restarting the hosts on the operating system or hardware level, the
RPM-based installation automatically starts the SEP processes.
The tarball installation does not start automatically. You can however configure
it to perform the start by using the `launcher` script as daemon script. Refer
to the documentation of your used Linux distribution for details.

(pb-stop)=

##  Stop cluster

The `stop` playbook stops the coordinator and all worker processes on the hosts.
It does not take into account if SEP actively processes any
workload, and simply terminates.

```shell
ansible-playbook playbooks/stop.yml
```

You can use the {ref}`graceful shutdown <pb-graceful-shutdown>` as an
alternative.

(pb-restart)=

##  Restart cluster

The `restart` playbook stops and then starts the coordinator and all worker
processes on the hosts.

```shell
ansible-playbook playbooks/restart.yml
```

It is equivalent to running the `stop` and the `start` playbook sequentially.

(pb-check-status)=

##  Check service status

The `check-status` playbook checks the status of the the coordinator and all
worker processes and displays the results.

```shell
ansible-playbook playbooks/check-status.yml
```

The playbook uses the `init.d` script for the RPM-based installation or the
`launcher` script for a tarball installation to get the status of each service.
If the service is running, you see a log for each address in your inventory file
stating `Running as <pid>`:

```shell
TASK [Print status] ********
ok: [172.28.0.2] => {
    "msg": "Running as 1965"
}
ok: [172.28.0.3] => {
    "msg": "Running as 1901"
}
ok: [172.28.0.4] => {
    "msg": "Running as 1976"
}
```

If a service is not active, you see `Not running`:

```shell
TASK [Print status] ****
ok: [172.28.0.2] => {
    "msg": "Not running"
}
ok: [172.28.0.3] => {
    "msg": "Not running"
}
ok: [172.28.0.4] => {
    "msg": "Not running"
}
```

(pb-graceful-shutdown)=

##  Graceful worker shutdown

The `graceful-shutdown` playbook stops the worker processes after all tasks are
completed.

```shell
ansible-playbook playbooks/graceful-shutdown.yml
```

Using {doc}`graceful shutdown </admin/graceful-shutdown>` takes
longer than using the `stop` playbook, because it allows workers to complete any
assigned work. If all workers are shut down, no further query processing is
performed by the cluster. The coordinator remains running at all times, until
manually shut down.

(pb-rolling-restart-workers)=

##  Rolling worker restart

The `rolling-restart-workers` playbook stops and starts all worker processes
sequentially one after the other using a {ref}`graceful shutdown
<pb-graceful-shutdown>` and a new start.

```shell
ansible-playbook playbooks/rolling-restart-workers.yml
```

You can configure the following variables in `files/vars.yml` to manage
graceful shutdowns:

* `graceful_shutdown_user` - user name to pass to the `X-Presto-User` or
  `X-Trino-User` header when issuing the graceful shutdown request via the HTTP
  API
* `graceful_shutdown_retries` -  number of times to check for successful
  shutdown before failing
* `graceful_shutdown_delay`- inactive duration between shutdown status checks
* `rolling_restart_concurrency` - number of workers to restart at the same time

By default, the playbook waits up to 10 minutes for any individual worker to
stop and start. Each operation has a 10 minute timeout. If this timeout is
reached, then the playbook execution fails. If you have a longer shutdown
grace period configured, you may want to extend this timeout.

Keep in mind that this playbook does not change any configuration of the
workers. If you push configuration changes to the cluster before a rolling
restart, the cluster can be in an inconsistent state until the restart is
completed. This can lead to query failures and other issues. A simple addition
of a catalog file is possible. The new catalog only becomes usable after all
workers are restarted. Updates to catalog and other configuration files
typically result in problems.

(pb-collect-logs)=

## Collect logs

The `collect-logs` playbook downloads the log files from all hosts to the
control machine.

```shell
ansible-playbook playbooks/collect-logs.yml
```

The server, HTTP request, and launcher logs files from each host are copied into
`logs/coordinator-<hostname>` for the coordinator or `logs/worker-<hostname>`
for the workers in the current directory.

(pb-collect-jstacks)=

## Collect Jstack Dump

The `collect-jstacks` playbook runs the
[`jstack`](https://docs.oracle.com/en/java/javase/11/tools/jstack.html)
command to capture the Java thread dump of the `StarburstTrinoServer` process on
all hosts, and download each dump to a local file on the control machine. The
playbook displays the local file name at the end of the play.

```shell
ansible-playbook playbooks/collect-jstacks.yml
```

(pb-uninstall)=

## Uninstall cluster

The `uninstall` playbook removes all modifications made on the hosts by other
playbooks. {ref}`Stop the cluster <pb-stop>` before running the playbook.

```shell
ansible-playbook playbooks/uninstall.yml
```

The playbook deletes all data, configuration and log files. It also removes the
deployed binary packages and the created user accounts and groups.
