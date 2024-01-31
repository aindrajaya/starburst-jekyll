---
layout: default
title: Installation Guide
parent: Deploy with Starburst Admin
grand_parent: SEP Deployment Mechanisms
nav_order: 2
permalink: /docs/deployment-mechanisms/starburst-admin/installation-guide/
---

# Installation guide

You can proceed with the installation of Starburst Admin using the following steps,
after {ref}`preparing the control node and the managed cluster nodes
<sb-admin-requirements>`.

## Define the inventory

Before using the playbooks, you need to edit the [Ansible inventory
file](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html),
`hosts`, to define where to install the software:

1. Copy the `playbooks/hosts.example` inventory file to `playbooks`, name it
   `hosts` and set the hostname for your coordinator and your worker(s).
2. Set the environment variable `ANSIBLE_INVENTORY` to the absolute path
   of the hosts file, for example:

   ```
   export ANSIBLE_INVENTORY=~/.ansible/collections/ansible_collections/starburst/admin/playbooks/hosts
   ```

3. Specify the IP address of the single coordinator host and one or more worker
   hosts.
4. Set the username for Ansible to connect to the host with SSH with
   `ansible_user` for each host.
5. Set the password  value with `ansible_password` or use a path to a private
   key using `ansible_ssh_private_key_file`.

The following snippet shows a simple example with one coordinator and two
workers:

```
[coordinator]
172.28.0.2 ansible_user=root ansible_password=changeme

[worker]
172.28.0.3 ansible_user=root ansible_password=changeme
172.28.0.4 ansible_user=root ansible_password=changeme
```

Run the Ansible `ping` command to validate connectivity to the hosts:

```shell
$ ansible all -m ping -i playbooks/hosts
172.28.0.4 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
172.28.0.2 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
172.28.0.3 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
```

Alternatively to using `ANSIBLE_INVENTORY` for the location of the hosts file,
you can create an `ansible.cfg` file in the root directory of `starburst-admin`
(typically `~/.ansible/collections/ansible_collections/starburst/admin/`) with
the setting `inventory=playbooks/hosts` in the `[defaults]` section and run the
commands from that folder.

```
[defaults]
inventory=playbooks/hosts
```

You can also specify the hosts file when running the `ansible-playbook` command
with the `-i <hostsfile>` parameter.

Maintaining multiple host files and potentially multiple sets of configuration
files enable you to manage multiple clusters.

## Provide Starburst Enterprise package

Decide if you want to use the RPM or tarball installation. RPM requires `sudo`
access and installs files in specific folders. When using the tarball, custom
folders can be specified, and they don't have to be owned by root.

Download the binary archive and place it in the `files` directory.

Alternatively, configure the `installer_url` in `playbooks/vars.yml` to point to
a URL that is available from all your cluster nodes to download the package
directly.

Reference the {ref}`documentation about the install
playbook <pb-install>` to learn more about all
the relevant configuration steps and options.

(create-minimal-configuration)=

## Create minimal configuration

The default configuration is managed in separate configuration files in `files`
as well as variables defined in in `playbooks/vars.yml`. It is suitable to get
started after the following adjustments:

* Update the `version` value in `vars.yml` to the version of downloaded binary
  file.

  For example, if you are deploying `starburst-enterprise-364-e.4.rpm`, the
  correct version is `364-e.4`.

  Other examples are `365` for {{oss}}, `365-e` for the
  SEP STS release, and `360-e.8` for a specific patch version of
  another SEP LTS release.

* Set the `node_environment` value in `vars.yml` to the desired cluster name.
* Update the Java heap space memory configuration `-Xmx16G` in
  `files/coordinator/jvm.config.j2` and `files/worker/jvm.config.j2` to
  about 80% of the total available memory on each host.
* Remove the `query.*` properties in
  `files/coordinator/config.properties.j2` and
  `files/worker/config.properties.j2` if you are using Starburst Admin
  1.0.0. Newer version do not include these properties. Starburst Admin
  calculates optimal values automatically.
* Add the Starburst Enterprise license file to `files/extra/etc` as
  `etc/starburstdata.license`. For more information, see {ref}`Push
  configurations <pb-push-configs>`

For installation as a non-root user, the following additional changes are
required in `playbooks/vars.yml`:

* Set the `become_root` variable to `no`.
* Set the directory variables to point to the installation directory, similar to
  the following example for installing to `/opt/starburst`:

```shell
installation_root: /opt/starburst
data_directory: "{{ installation_root }}/data"
etc_directory: "{{ installation_root }}/etc"
log_directory: "{{ installation_root }}/log"
```

The default setup includes a `tpch` catalog, that you can use to test queries
and ensure the cluster is operational.

If you are using the tarball, you can edit the target installation paths.

Proceed with this simple configuration until you have verified that the cluster
can be started and used, and then follow up with more configuration and changes.

Reference the {ref}`documentation about the push-configs
playbook <pb-push-configs>` to learn more about
all the relevant configuration steps and options.

The {doc}`operation guides <operation-guide>` include numerous tips
and tricks for these next steps for your deployment.

(run install and start cluster)=

## Run installation and start the cluster

You are finally ready to get Starburst Enterprise installed on all cluster
nodes. The following steps copy the package to all cluster nodes and install it.
Ansible creates the necessary configuration files from `files` and the defined
variables to the nodes, and starts the application on all of them:

```shell
ansible-playbook playbooks/install.yml
ansible-playbook playbooks/push-configs.yml
ansible-playbook playbooks/start.yml
```

* {ref}`install details <pb-install>`
* {ref}`push-configs details <pb-push-configs>`
* {ref}`start details <pb-start>`


## Accessing the Web UI

A first step to validate the cluster is to log in to the {doc}`Web UI
</admin/web-interface>`:

* Navigate to port 8080 on the coordinator host with the IP address or the
  configured fully qualified domain name.
* Login with any username.
* Verify the displayed number of workers equals the number of workers in the
  `hosts` file.

## Connecting with clients

You can use the [CLI or any other
client](../../clients/index.html){.external} to connect to the
cluster and verify that the `tpch` catalog is available.

## Managing configuration

For production usage you need to manage a separate set of all configuration
files for each cluster you operate. It is also important to keep track of any
changes. Typically a version control system such as Git is used.

You can also expand your processes to automatic management with a GitOps
workflow and tools such as [Ansible
Tower](https://www.ansible.com/products/tower) or
[Concord](https://concord.walmartlabs.com/).

## Next steps

After the successful installation and first initial tests, you can proceed to
read about all the {doc}`available playbooks <playbook-reference>`
and {doc}`learn about updating and operating the
cluster <operation-guide>`.
