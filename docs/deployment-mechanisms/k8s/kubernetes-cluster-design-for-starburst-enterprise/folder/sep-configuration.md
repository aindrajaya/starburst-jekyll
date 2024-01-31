---
layout: default
title: Configuring Starburst Enterprise in Kubernetes
parent: Kubernetes cluster design for Starburst Enterprise
grand_parent: Deploy with Kubernetes
nav_order: 3
permalink: /docs/kubernetes/kubernetes-cluster-design-for-starburst-enterprise/sep-configuration/
---

# Configuring Starburst Enterprise in Kubernetes

The `starburst-enterprise` Helm chart configures the Starburst Enterprise platform (SEP) coordinator
and worker nodes and the internal cluster communication with the `values.yaml`
file detailed in the following sections.

We strongly suggest that you follow best practices when customizing your
cluster, creating small, targeted files to override any defaults, and adding any
configuration properties. For guidance on creating override files, see our
{ref}`recommended customization file set <k8s-customization-file-set>`.

:::{note}
Many section in this page have links to examples of the relevant YAML in our
{doc}`YAML examples page <sep-config-examples>`. While you are configuring
your cluster, it is is helpful to refer to SEP's {ref}`directory structure
<helm-sep-directories>` to ensure you are configuring any file locations
correctly.
:::

Throughout this page there are links to more descriptive documentation for a
given configuration area. In many cases, these refer to the legacy configuration
file names by name.

## Configuring SEP with Helm charts

SEP uses a number of configuration files internally that determine how it
behaves:

- `etc/catalog/<catalog name>.properties`
- `etc/config.properties`
- `etc/jvm.properties`
- `etc/log.properties`
- `etc/node.properties`
- `etc/access-control.properties`

With our Kubernetes deployment, these files are built using Helm charts, and are
nested in the `coordinator.etcFiles` YAML node.

Catalog properties files are also built using Helm charts. They are defined
under the top-level `catalogs:` YAML node. The catalogs and properties you
create depend on what data sources you connect to with SEP. You can configure
as many as you need.

Every Helm-based SEP deployment includes a `values.yaml` file. The file
contains the default values, any of which can be overridden.

Along with basic instance configuration for the various cloud platforms,
`values.yaml` also includes the key-value pairs necessary to build the
required `*.properties` files that configure SEP.

A recommended set of customization files is included later in this topic,
including recommendations for creating specific override files, with examples.

### Default YAML files

Default values are provided for a minimum configuration only, not including
security or any catalog connector properties, as these vary by customer needs.
The configuration in the Helm chart also contains deployment information such as
registry credentials and instance size.

Each new release of SEP includes new chart version, and the default values may
change. For this reason, we **highly recommend** that you follow best practices
and leave the `values.yaml` file in the chart untouched, overriding only very
specific values as needed in one or more separate YAML files.

:::{note}
Do not change the `values.yaml`, or copy it in its entirety and make changes
to it and specify that as the override file, as new releases may change
default values or render your values non-performant. Use separate files with
changed values only, as described above.
:::

### Using version control

We **very strongly** recommend that you manage your customizations in a version
control system such as a git repository. Each cluster and deployment has to be
managed in separate files.

### Creating and using YAML files

The following is a snippet of default values from the SEP `values.yaml` file
embedded in the Helm chart:

```yaml
coordinator:
  resources:
    memory: "60Gi" requests:
      cpu: 16
```

The default 60GB of required memory is potentially larger than any of the
available pods in your cluster. As a result, the default prevents your
deployment success, since no suitable pod is available.

To create a customization that overrides the default size for a test cluster,
copy and paste **only that section** into a new file named
`sep-test-setup.yaml`, and make any changes. You must also include the
relevant structure above that section. The memory settings for workers have the
same default values and need to be overridden as well:

```yaml
coordinator:
  resources:
    memory: "10Gi" requests:
      cpu: 2
worker:
  resources:
    memory: "10Gi" requests:
      cpu: 2
```

Store the new file in a path accessible from the `helm upgrade --install`
command.

When you are ready to install, specify the new file using the `--values`
argument as in the following example. Replace 4XX.0.0 with the Helm chart
version of the desired SEP release as documented on the {doc}`versions page
</versions>`:

```shell
helm upgrade my-sep-test-cluster starburstdata/starburst-enterprise \
  --install \
  --version 4XX.0.0 \
  --values ./registry-access.yaml \
  --values ./sep-test-setup.yaml
```

You can chain as many override files as you need. If a value appears in multiple
files, the value in the rightmost, last specified file takes precedence.
Typically it is useful to limit the number of files as well as the size of the
individual files. For example, it can be useful to create a separate file that
contains all catalog definitions.

To view the built-in configuration of the Helm chart for a specific version of
SEP, run the following command:

```shell
helm template starburstdata/starburst-enterprise --version 4XX.0.0
```

Use this command with different version values to compare the configuration of
different SEP releases as part of your upgrade process.

To generate the specific configuration files for your deployment, use the
template command with your additional values files:

```shell
helm template starburstdata/starburst-enterprise \
  --version 4XX.0.0 \
  --values ./registry-access.yaml \
  --values ./sep-test-setup.yaml
```

:::{note}
The generated files are for review purposes only. To apply changes to your
deployment, use the `helm upgrade` command with your files specified using
the `--values` argument.
:::

(k8s-customization-file-set)=

## Recommended customization file set

The file set described below describes a series of focused configuration files.
If you have more than one cluster, such as a test cluster and a production
cluster, name the files accordingly before you begin. Examples are provided in
the sections that follow.

:::{list-table} Recommended customization file set
:widths: 35, 65
:header-rows: 1

* - File name
  - Content
* - `registry-access.yaml`
  - Docker registry access credentials file, typically to access the Docker
    registry on the Starburst Harbor instance. Include the
    `registryCredentials:` or `imagePullSecrets:` top level node in this
    file to configure access to the Docker registry. This file can be used for
    all SEP, HMS, and Ranger configuration for all clusters you operate.
* - `sep-prod-catalogs.yaml`
  - Catalog configuration for all catalogs configured for SEP on the
    `prod` cluster. It is typically useful to separate catalog
    configurations out into a separate file to allow reuse across clusters, as
    well as to separate the large amount of configuration of the catalogs from
    all the cluster configuration.
* - `sep-prod-setup.yaml`
  - Main configuration file for the `prod` cluster. Include any
    configuration for all other top level nodes that configure the
    coordinator, workers, and all other aspects of the cluster.
:::

Create and manage additional configuration files, if you are operating multiple
clusters, while reusing the credentials file. For example, if you run a `dev`
and `stage` cluster use the following additional files:

- `sep-dev-catalogs.yaml`
- `sep-dev-setup.yaml`
- `sep-stage-catalogs.yaml`
- `sep-stage-setup.yaml`

There are several supporting services available for use with SEP, each with
their own Helm chart:

- {{sb}} {doc}`cache service </k8s/cache-service-configuration>`
- {doc}`Hive Metastore Service </k8s/hms-configuration>`
- {doc}`Apache Ranger </k8s/ranger-configuration>`

If you opt to use these services, you can create a configuration file for each
of these per cluster as well:

Production `prod` cluster:

- `cache-service-prod.yaml`
- `hms-prod.yaml`
- `ranger-prod.yaml`

Development `dev` cluster:

- `cache-service-dev.yaml`
- `hms-dev.yaml`
- `ranger-dev.yaml`

Staging `stage` cluster:

- `cache-service-stage.yaml`
- `hms-stage.yaml`
- `ranger-stage.yaml`

### `registry-access.yaml`

You can get started with a minimal file that only {ref}`adds your credentials to
the Starburst Harbor instance <minimal-values-YAML-file>`, as shown below:

```yaml
registryCredentials:
  enabled: true
  registry: harbor.starburstdata.net/starburstdata
  username: <yourusername>
  password: <yourpassword>
```

In the examples throughout our documentation, this file is named
`registry-access.yaml`.

### `sep-prod-catalogs.yaml`

A catalog YAML file adds all the configurations for defining the catalogs and
their connection details to the underlying data sources. The following snippet
contains a few completely configured catalogs that are ready to use:

- `tpch-testdata` exposes the TPC-H benchmark data useful for learning SQL and
  testing.
- `tmpmemory` uses the {doc}`/connector/memory` to provide a small temporary
  test ground for users.
- `metrics` uses the {doc}`/connector/jmx` and exposes the internal metrics of
  SEP for monitoring and troubleshooting.
- `clientdb` uses the {doc}`/connector/starburst-postgresql` to access the
  `clientdb` database.
- `datalake` and `s3` are stubs of catalogs using the
  {doc}`/connector/starburst-hive` with an HMS and a Glue catalog as metastore.

```yaml
catalogs:
  tpch-testdata: |
    connector.name=tpch
  tmpmemory: |
    connector.name=memory
  metrics: |
    connector.name=jmx
  clientdb: |
    connector.name=postgresql
    connection-url=jdbc:postgresql://postgresql.example.com:5432/clientdb
    connection-password=${ENV:PSQL_PASSWORD}
    connection-user=${ENV:PSQL_USERNAME}
  datalake: |
    connector.name=hive hive.metastore.uri=thrift://hive:9083
  s3: |
    connector.name=hive hive.metastore=glue
```

## Applying configuration changes

To update SEP with any configuration changes, run the `helm upgrade`
command with the updated YAML files and the `--install` switch, as in this
example:

```shell
$ helm upgrade my-sep-prod-cluster starburstdata/starburst-enterprise \
    --install \
    --version 434.0.0 \
    --values ./registry-access.yaml
    --values ./sep-prod-setup.yaml
```

You can use the same command as if you were updating to a new release. Helm will
compare all `--values` files and the version, and safely ignore any that are
unchanged.

## Top level nodes

The top level nodes are as follows, in order of appearance in the default
`values.yaml` file embedded in the Helm chart. Click on a section header to
access the relevant documentation:

:::{list-table} Top level `values.yaml` nodes
:widths: 45, 55
:header-rows: 1

* - Node name
  - Description
* - **Docker images** {ref}`section <helm-sep-image>`
  -
* - `image:`
  - Contains the details for the SEP Docker image.
* - `initImage:`
  - Contains the details for the SEP bootstrap Docker image.
* - **Docker registry access** {ref}`section <helm-sep-registry>`
  -
* - `registryCredentials:`
  - Defines authentication details for Docker registry access. Cannot
    be used if using `imagePullSecrets:`.
* - `imagePullSecrets:`
  - Alternative authentication for selected Docker registry using secrets.
    Cannot be used if using `registryCredentials:`.
* - **Internal communications** {ref}`section <helm-sep-internal-communication>`
  -
* - `sharedSecret:`
  - Sets the shared secret value for internal communications.
* - `environment:`
  - The environment name for the SEP cluster.
* - `internal:`
  - Specifies port numbers used for internal cluster communications.
* - **Ports and networking**
  -
* - `expose:` {ref}`node <helm-sep-expose>`
  - Defines the mechanisms and their options that expose the SEP cluster to
    an outside network.
* - **Coordinator and workers**
  - The coordinator and worker nodes have many properties in common. They are
    detailed in the Coordinator section, and referenced in the Worker section,
    rather than repeating them. Likewise, examples are provided for the
    `coordinator:` top level node, and can be applied to the worker top
    level node by simply replacing `coordinator` with `worker`.
* - `coordinator:` {ref}`node <helm-sep-coordinator>`
  - Extensive section that configures the SEP coordinator pod.
* - `worker:` {ref}`node <helm-sep-workers>`
  - Extensive section that configures the SEP worker pods.
* - **Startup options** {ref}`section <helm-sep-nodes>`
  -
* - `initFile: ""`
  - Specifies a shell script to run after coordinator and work pod launch, and
    before SEP startup on the pods.
* - `extraArguments: []`
  - List of extra arguments to be passed to the `initFile` script.
* - `extraSecret:`
  - Lists secret to be used for `initFile:`.
* - **Advanced security options**
  -
* - `externalSecrets:` {ref}`node <helm-sep-secretRef>`
  - Defines mounted external secrets from a Kubernetes secrets manager.
* - `userDatabase:` {ref}`node <helm-sep-user-db>`
  - Configures the user database to use for file-based user authentication.
    This is disabled by default for versions 356-e and later.
* - `securityContext: {}` {ref}`node <helm-sep-securityContext>`
  - Sets the Kubernetes security context that defines privilege and access
    control settings for all SEP pods.
* - **Pod lifecycle management** [section](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
  -
* - `readinessProbe:`
  - Specifies the Kubernetes readiness probe that monitor containers for
    readiness to accept traffic.
* - `livenessProbe:`
  - Specifies the Kubernetes liveness probe that monitor containers for the
    need to restart.
* - **SEP cache, disk and memory management**
  -
* - `query:` {ref}`node <helm-sep-concurrent-queries>`
  - Specifies query processing and management configuration properties.
* - `spilling:` {ref}`node <helm-sep-spilling>`
  - Configures spilling of query processing data to disk.
* - `cache:` node for {ref}`hive configuration <helm-sep-hive-caching>`
  - Configures Hive storage caching.
* - **Catalogs to configure connected data sources**
  -
* - `catalogs:` {ref}`node <helm-sep-catalogs>`
  - Configures the catalog properties files.
* - **Miscellaneous**
  -
* - `additionalVolumes:` {ref}`node <helm-sep-volumes>`
  - Defines mounted volumes for various uses in the cluster.
* - `prometheus:` {ref}`node <helm-sep-prometheus>`
  - Configures cluster metrics gathering with Prometheus.
* - `commonLabels: {}`
  - Defines common labels to identify objects in a KRM to use with the
    [kustomize utility](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/commonlabels/)
    and other tools.
:::

% source at
% https://github.com/starburstdata/starburst-platform-charts/blob/master/starburst-enterprise/helm/values.yaml

(helm-sep-image)=

## Docker images

The {ref}`default configuration <k8s-defaults-images-and-registry-credentials>`
automatically contains the details for the relevant Docker image on the
Starburst Harbor instance. Typically you should not configure any overrides. An
exception is if you host the Docker image on a different Docker registry.

:::{list-table} Top level docker image nodes
:widths: 30, 70
:header-rows: 1

* - Node name
  - Description
* - `image:`
  - The `image` section controls the Docker image to use including the
    version. Typically, the Helm chart version reflects the SEP version.
    Specifically, the SEP version 356-e.x is reflected in the chart version
    of 356.x.0, or patches for the chart are released as 356.x.1, 356.x.2,
    and similar.
* - `initImage:`
  - The `initImage` section defines the container image and version to use
    for bootstrapping configuration files for SEP from the Helm chart and
    values files. This functionality is internal to the chart, and you should
    not change or override this configuration.
:::

Defaults and examples:

- {ref}`k8s-defaults-images-and-registry-credentials`
- {ref}`k8s-ex-images-and-registry-credentials`
- {ref}`k8s-ex-registry-version-handling`

(helm-sep-registry)=

## Docker registry access

These nodes enable access to the Docker registry. They should be added to the
`registry-access.yaml` file to access the registry as shown in the {ref}`best
practices guide <minimal-values-YAML-file>`.

:::{list-table} Top level docker image and registry credentials nodes
:widths: 30, 70
:header-rows: 1

* - Node name
  - Description
* - `registryCredentials:`
  - The `registryCredentials` section defines authentication details for
    Docker registry access. Typically, you need to use {ref}`your username and
    password for the Starburst Harbor instance <k8s-helm-repository>`. NOTE:
    Cannot be be used if using `imagePullSecrets:`.
* - `imagePullSecrets:`
  - Alternative authentication for selected Docker registry using secrets.
    NOTE: Cannot be used if using `registryCredentials:`.
:::

:::{warning}
You can only define authentication with `registryCredentials` or
`imagePullSecrets`. Defining both is invalid, and causes an error.
:::

Defaults and examples:

- {ref}`k8s-defaults-images-and-registry-credentials`
- {ref}`k8s-ex-images-and-registry-credentials`
- {ref}`k8s-ex-private-registries`
- {ref}`k8s-ex-imagePullSecrets`

(helm-sep-internal-communication)=

## Internal communication

SEP provides three top level nodes for {doc}`internal cluster communication
</security/internal-communication>` between the coordinator and the workers.
Because internal communication configurations and credentials are unique, these
are not configured {ref}`by default <k8s-defaults-internal-comms>`.

Ensure to configure `sharedSecret` and `environment` values in all your
clusters to use secured communication with internal authentication of the nodes.

:::{list-table} Top level internal communications nodes
:widths: 30, 70
:header-rows: 1

* - Node name
  - Description
* - `environment`
  - The environment name for the SEP cluster, used to set the
    `node.environment` {ref}`node configuration property
    <node-properties>`. Examples are values such as `production`,
    `staging`, `batch_cluster`, `site_analytics`, or any other short,
    meaningful string.
* - `sharedSecret`
  - Sets the shared secret value for {doc}`secure communication
    between coordinator and workers in the cluster
    </security/internal-communication>` to a long, random string that you
    provide. If not set, the `node.environment` value from the
    `etcFiles.node.properties` configuration {ref}`on the
    coordinator <k8s-etcfiles>` is used.
* - `internalTls:`
  - When set to `true`, enables TLS for internal communication and
    autogenerates trusted certificates. Requires both `environment` and
    `sharedSecret` top level nodes to be configured, and uses the port
    defined in `internal.ports.https.port:`. The `environment` value is
    used as the CN. Set to `false` by default.
:::

Defaults and examples:

- {ref}`k8s-ex-internal-comms`
- {ref}`k8s-defaults-internal-comms`
- {ref}`k8s-ex-internal-tls`
- {ref}`k8s-ex-nonstd-https`

(helm-sep-expose)=

## Exposing the cluster

You must expose the cluster to allow users to connect to the SEP coordinator
with tools such as the CLI, applications using JDBC/ODBC drivers and any other
client application. This service-type configuration is defined by the `expose`
top level node. You can choose from four different mechanisms by setting the `type`
value to the [common configurations in k8s](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types).

% modelled after https://github.com/goharbor/harbor-helm/blob/55598b1f1ccba2bc1c12e94a2622840cae148eb8/values.yaml#L1

Depending on your choice, you only have to configure the identically-named
sections.

:::{list-table} Types for `expose:` top level node
:widths: 30, 70
:header-rows: 1

* - Type
  - Description
* - `clusterIp`
  - {ref}`Default value <k8s-defaults-external-comms>`. Only exposes the
    coordinator internally within the k8s cluster using an IP address internal
    to the cluster. Use this in the early stages of configuration.
* - `nodePort`
  - Configures the internal port number of the coordinator for requests from
    outside the cluster on the `nodePort` port number. External
    service requests are made to `<nodeIP>:<nodePort>`. The `clusterIP`
    service is automatically created to supply all internal IP addresses.
* - `loadBalancer`
  - Used with platforms that provide a load balancer. This
    option automatically creates `nodePort` and `clusterIP` services , to
    which the external load balancer routes requests.
* - `ingress`
  - This option provides a production-level, securable configuration. It
    allows a load balancer to route to multiple apps in the cluster, and
    may provide load balancing, SSL termination, and name-based virtual
    hosting. For example, the SEP coordinator and Ranger server can be
    in the same cluster, and can be exposed via `ingress:` configuration.
:::

Defaults and examples:

- {ref}`k8s-defaults-external-comms`
- {ref}`k8s-ex-external-comms`
- {ref}`k8s-ex-expose-clusterip`
- {ref}`k8s-ex-expose-nodeport`
- {ref}`k8s-ex-expose-lb`
- {ref}`k8s-ex-expose-ingress`
- {ref}`k8s-ex-expose-nginxcert`

(helm-sep-coordinator)=

## Coordinator

The top level `coordinator:` node {ref}`contains property nodes
<k8s-coordinator-yaml>` that configure the pod of the cluster that runs the
SEP coordinator. The {ref}`default values <k8s-defaults-coordinator>` are
suitable to get started with reasonable defaults on a production-sized k8s
cluster.

:::{note}
These property nodes also appear under the top level `worker:` node unless
otherwise noted.
:::

:::{list-table} `coordinator:` nodes
:widths: 30, 70
:header-rows: 1

* - Node name
  - Description
* - `etcFiles`
  - Specifies the content of the `jvm.config` file and the `*.properties`
    files associated with the coordinator. These files are described
    in greater detail in a {ref}`separate table <k8s-etcfiles>` below.
    **NOTE**: Not all files described in the detail table are applicable to
    workers. They are noted as such.
* - `resources`
  - The CPU and memory resources to use for the coordinator pod. These
    settings {ref}`can be adjusted <k8s-ex-coord-resources>` to match your
    workload needs, and available node sizes in the cluster.
* - `nodeMemoryHeadroom`
  - The size of the container memory headroom. The value needs to be less than
    resource allocation limit for memory defined in `resources`.
* - `heapSizePercentage`
  - Percentage of container memory reduced with headroom assigned to Java
    heap. Must be less than 100.
* - `heapHeadroomPercentage`
  - Headroom of Java heap memory not tracked by SEP during query execution.
    Must be less than 100.
* - `additionalProperties`
  - Any properties described in the reference documentation that are not
    specific to any other YAML node are set here. Example usages are to
    {ref}`set query time-out and memory values <k8s-ex-additional-properties>`
    and to {ref}`enable Starburst Insights <k8s-ex-insights>`. Configuration
    properties can be found throughout the reference documentation,
    including the {doc}`Properties reference <../admin/properties>` page.

    :::{warning}
    Avoid including secrets, such as credentials, in the
    `additionalProperties` section of the Helm chart. These values may
    be exposed in the init container logs. Instead, use the
    `config.properties` section to securely include such information.
    :::

* - `envFrom: []`
  - Allows for the propagation of environment variables from different sources
    complying with
    [K8S schema](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.25/#envfromsource-v1-core)
    can be used to deliver values to SEP configuration properties file by
    creating a Kubernetes secret holding variable values.
* - `nodeSelector: {}`, `affinity: {}` and `tolerations: []`
  - Configuration to {ref}`determine the node and pod to
    use <helm-node-assignment>`.
* - `deploymentAnnotations: {}`
  - Configuration to [annotate](https://helm.sh/docs/chart_best_practices/labels/)
    the coordinator deployment.
* - `podAnnotations: {}`
  - Configuration to [annotate](https://helm.sh/docs/chart_best_practices/labels/)
    the coordinator pod.
* - `priorityClassName`
  - Priority class for coordinator pod for setting [k8s pod priority and
    preemption](https://kubernetes.io/docs/concepts/configuration/pod-priority-preemption).
* - `sidecars: []`
  - Attach additional sidecar containers to the coordinator pod.
* - `initContainers: []`
  - Add extra init containers to the coordinator pod.
:::

Defaults and examples:

- {ref}`k8s-defaults-coordinator`
- {ref}`k8s-ex-coord-resources`
- {ref}`k8s-ex-additional-properties`
- {ref}`k8s-ex-insights`
- {ref}`helm-sep-coordinator-envfrom`

(k8s-etcfiles)=

### `etcFiles` on the coordinator

:::{note}
These property nodes also appear under the top level `worker:` node unless
otherwise noted.
:::

:::{list-table} `etcFiles` on the coordinator
:widths: 30, 70
:header-rows: 1

* - Node name
  - Description
* - `coordinator.etcFiles.jvm.config`
  - Defines the content of the {ref}`jvm-config` for the coordinator.
* - `coordinator.etcFiles.properties`
  - Defines configuration files located in the `etc` folder. Each nested
    section defines the filename. Defaults are provided for the main
    configuration files `etc/jvm.config`, `etc/config.properties`,
    `etc/node.properties` and `etc/log.properties`. Additional
    properties files can be added by adding a nested section and the
    desired content of the configuration file.
* - `coordinator.etcFiles.properties.config.properties`
  - Defines the content of the
    {ref}`default configuration file <config-properties>`
    for the coordinator. You can also use `additionalProperties` for
    adding other configuration values.
* - `coordinator.properties.node.properties`
  - Defines the contents of the {ref}`node.properties file <node-properties>`
    for the coordinator, including paths to installation and log directories.
    The unique identifier and environment are pulled in from the `environment`, as documented in the {ref}`internal
    communication section <helm-sep-internal-communication>`.
* - `coordinator.etcFiles.properties.log.properties`
  - Defines the contents of the {ref}`log configuration files <log-levels>`
    for the coordinator.
* - `coordinator.etcFiles.properties.password-authentication.properties`
  - Configures {doc}`password authentication <../security/password-file>`
    for SEP on the coordinator.
    **NOTE:** Does not apply to `worker.etcFiles.properties`.
* - `coordinator.etcFiles.properties.access-control.properties`
  - Enables and configures {ref}`access control <k8s-ex-access-control>`
    for SEP on the coordinator. **NOTE:** Does not apply to
    `worker.etcFiles.properties`.
* - `coordinator.etcFiles.properties.exchange-manager.properties`
  - Configures an optional exchange manager for
    {doc}`/admin/fault-tolerant-execution`.
* - `coordinator.etcFiles.other`
  - Other files that need to be placed in the `etc` directory
    ({ref}`example <k8s-ex-etcfiles-other>`).
:::

(helm-node-assignment)=

### Node assignment

All charts allow you to configure criteria to define which node and pod in the
cluster is suitable to use for running the relevant container. In SEP, this
may be useful if you have a cluster spanning availability zones, or if you are
using Ranger or an HMS in your cluster with smaller node sizes. The following
configurations are available, and by default are not defined:

```yaml
nodeSelector: {}
affinity: {}
tolerations: []
```

Example configurations are available in the [k8s documentation](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/).
Specific usage and values are highly dependent on your k8s cluster
configuration.

Further resources:

- [Node labels for pod assignment](https://kubernetes.io/docs/user-guide/node-selection/)
- [Tolerations](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#taints-and-tolerations-beta-feature)
- [Affinity and anti-affinity](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#affinity-and-anti-affinity)

### Coordinator defaults and examples

- {ref}`k8s-defaults-coordinator`
- {ref}`k8s-ex-jvm-config`
- {ref}`k8s-ex-etcfiles-other`
- {ref}`k8s-ex-coord-resources`
- {ref}`k8s-ex-additional-properties`
- {ref}`helm-sep-coordinator-envfrom`
- {ref}`k8s-ex-insights`
- {ref}`k8s-ex-access-control`
- {ref}`k8s-ex-sep-user-db`

(helm-sep-workers)=

## Workers

The `worker` section configures the pods of the cluster that run the
SEP workers.

The top level `worker:` node {ref}`contains property nodes <k8s-worker-yaml>`
that configure the pod of the cluster that runs the SEP workers. The
{ref}`default values <k8s-defaults-worker>` are suitable to get started with
reasonable defaults on a production-sized k8s cluster.

Many of the properties for this node also appear in the `coordinator:` top
level node. They are fully documented in the {ref}`Coordinator
section <helm-sep-coordinator>`:

- `worker.etcFiles.*`
- `worker.resources`
- `worker.nodeMemoryHeadroom`
- `worker.heapSizePercentage`
- `worker.heapHeadroomPercentage`
- `worker.additionalProperties`
- `worker.envFrom`
- `worker.nodeSelector`
- `worker.affinity`
- `worker.tolerations`
- `worker.deploymentAnnotations`
- `worker.podAnnotations`
- `worker.priorityClassName`
- `worker.sidecars`
- `worker.initContainers`

The following properties apply only to `worker`:

:::{list-table} `worker:`-only nodes
:widths: 30, 70
:header-rows: 1

* - Node name
  - Description
* - `etcFiles`
  - Specifies
* - `worker.replicas`
  - The number of {ref}`worker pods <k8s-defaults-worker>` for a static
    cluster.
* - `worker.autoscaling`
  - Configuration for the minimum and maximum number of workers. Ensure the
    additional {ref}`requirements for scaling <k8s-scaling-requirements>` are
    fulfilled on your k8s cluster. Set `enabled` to `true` to activate
    {ref}`autoscaling <k8s-scaling>`.
    The `targetCPUUtilizationPercentage` sets the threshold value that
    triggers scaling up by adding more workers until `maxReplicas` is
    reached.

    Scaling down proceeds until `minReplicas` is reached and is controlled
    by `deploymentTerminationGracePeriodSeconds` and
    `starburstWorkerShutdownGracePeriodSeconds`.

    **WARNING:** The autoscaling feature does not yet work with OpenShift
    clusters (as of the latest release 4.6), due to a known limitation with
    OpenShift clusters. HPA
    [does not work](https://bugzilla.redhat.com/show_bug.cgi?id=1749468)
    with pods having init containers in OpenShift.

    Read more information in our {ref}`scaling section <k8s-scaling>`.
* - `worker.kedaScaler`
  - An alternative method of scaling the number of workers is implemented
    using the [KEDA scaler for Kubernetes workloads](https://keda.sh/docs/2.6/scalers/kubernetes-workload/)
    in the SEP Helm chart. Scaler configuration is described in a
    {ref}`dedicated section <keda-sep-scaler>`.
* - `worker.deploymentTerminationGracePeriodSeconds`
  - Specifies the termination grace period for workers. Workers are not
    terminated until queries running on the pod are finished and the grace
    period passes.
* - `worker.starburstWorkerShutdownGracePeriodSeconds`
  - Sets `shutdown.grace-period` to configure the grace period for worker
    process shutdown.
:::

The following configuration properties for workers are identical to the
coordinator properties, documented in preceding section.

(helm-sep-nodes)=

## Startup shell script for coordinator and workers nodes

`initFile:` is a top level node that allows you to create a startup shell
script ({ref}`example <k8s-ex-startup-script>`) to customize how SEP is
started on the coordinator and workers, and pass additional arguments to it.
These are undefined {ref}`by default <k8s-defaults-initfile>`.

:::{note}
To execute commands in your `initFile` script that are not available in the
main SEP Docker image, {{sb}} recommends using init containers for
initialization and setup tasks. Init containers can use any Docker image. For
information on configuring init containers within your Kubernetes deployment,
see {ref}`helm-sep-initcontainers`.
:::

:::{list-table} Startup script nodes
:widths: 30, 70
:header-rows: 1

* - Node name
  - Description
* - `initFile`
  - A shell script to run before SEP is launched. The content of the file
    has to be an inline string in the YAML file. The script is started as
    `/bin/bash <<init_file>>`. When called, it is passed the single
    parameter value `coordinator` or `worker` depending on the type of
    pod. The script needs to invoke the launcher script
    `/usr/lib/starburst/bin/run-starburst` for a successful start of SEP.
    **NOTE**: In the `initFile`, for chart versions released before middle
    of Oct 2020 use the launcher script
    `exec /usr/lib/starburst/bin/launcher run`. For newer releases use
    `exec /usr/lib/starburst/bin/run-starburst` to enable graceful shutdown
    of workers .
* - `extraArguments`
  - List of extra arguments to be passed to the `initFile` script.
:::

(k8s-security-considerations)=

## User security considerations

SEP has {doc}`extensive security options <../security>` that allow you specify
how to authenticate users. Because user security configurations and credentials
are unique, these are not configured {ref}`by default
<k8s-defaults-security-considerations>`.

:::{note}
We **strongly suggest** that you watch our {doc}`Securing Starburst Enterprise
</admin-topics/security>` training video before customizing your SEP
security configuration.
:::

(helm-sep-securitycontext)=

### Security context

You can configure a [security context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-the-security-context-for-a-pod)
to define privilege and access control settings for the SEP pods.

```yaml
securityContext:
  runAsNotRoot: true
  runAsUser: 1000
  runAsGroup: 0
```

:::{note}
These settings are typically not required, and are highly dependent upon your
Kubernetes environment. For example, OpenShift requires `audit_log` to be
set in `securityContext` in order to run `sudo`, while other platforms use
arbitrary user IDs. Refer to the Kubernetes documentation and to the
documentation for your particular platform to ensure your `securityContext`
is configured correctly.
:::

(helm-sep-serviceaccount)=

### Service account

You can configure a [service account](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)
for the SEP pods using:

```yaml
serviceAccountName:
```

(helm-sep-secretref)=

### External secret reference

% todo: No idea how this relates if at all to the section that follows. Can
% they work together? Seems they should be able to. Need a "not to be confused
% with the top level sharedSecret: node blurb. Note at the end needs to be
% promoted to an actual passage, with example. Is this the LDAP integration
% method? That needs to be called out in a section header explicitly, not buried.
% THOUGHT: should the two following sections be recast as "Configuring access
% control with external secrets" and "Configuring access control with file-based
% authentication"? If correct, that would be a heck of a lot clearer

There are several locations where properties require pointing to files delivered
outside of SEP, such as CA certificates. In such cases, you can use a
special notation that allows you to point to a k8s secret.

For example, you can configure password authentication using LDAP ({ref}`example
<k8s-ex-secretRef>`). This requires k8s to create the
`etc/password-authenticator.properties` configuration file, which in turn
points to the `ca.crt` certificate file.

(k8s-external-secrets)=

### Defining external secrets

You can automatically mount external secrets, for example from the AWS Secrets
Manager, using the `secretRef` or `secretEnv` notation.

:::{list-table} `externalSecrets` nodes
:widths: 30, 70
:header-rows: 1

* - Node name
  - Description
* - `externalSecrets.type`
  - Type of the external secret provider. Currently, only `goDaddy` is
    supported. NOTE: The external secret provider needs to be configured with
    proper access to the specified `backendType` in order to successfully
    pull in the external secrets.
* - `externalSecrets.secretPrefix`
  - Prefix of all secrets that need to be mapped to external secret.
* - `externalSecrets.goDaddy.backendType`
  - The [external-secrets
    backend](https://github.com/external-secrets/external-secrets)
    type, such as `secretsManager` or `systemManager`.
:::

The Helm chart scans for all `secretRef` or `secretEnv` references in the
referenced YAML files which start with the configured `secretPrefix` string.
For each secret found, it generates an\`\`ExternalSecret\`\` K8s manifest
({ref}`example <k8s-ex-external-secrets>`).

:::{note}
The selected external secrets provider needs to be deployed and configured
separately. The secret names in the external storage must match names of K8s
secrets you reference. When using `secretEnv`, the external storage secret
must contain only a single value. For each external secret a single K8s secret
is created, including one key with external secret value.
:::

(helm-sep-user-db)=

### File-based authentication

The unix command `htpasswd` can generate a user database, which can be used to
configure file-based user authentication. It creates the file under
`/usr/lib/starburst/etc/auth/{% raw %}{{ .Values.userDatabase.name | default "" }}{% endraw %}`. This allows you to
statically deliver user credentials to the file
`etc/password-authenticator.properties`
({ref}`example <k8s-ex-sep-user-db>`).

% todo: BUG! I seriously doubt this non-SEP utility magically just creates it
% in the right directory for SEP. Need a better/more correct blurb here.

Alternatively, you can use a secret to add an externally created user database
file to SEP. Set the `file.password-file` property to the user database file,
and ensure to disable the built-in user database
({ref}`example <k8s-ex-sep-user-db>`).

% todo: with what node does this get set etcFiles.properties or
% additionalProperties? Is it an additional files thing as well?

(k8s-performance-considerations)=

## Performance considerations

SEP has performance management configuration options that help you to
optimize aspects of SEP's performance.

:::{note}
We **strongly suggest** that you watch our {doc}`cluster tuning training video
</admin-topics/performance-tuning>` before customizing your SEP security
configuration.
:::

(helm-sep-concurrent-queries)=

### Query memory usage control

The top level `query:` node lets you to set the `maxConcurrentQueries`
configuration property. The `maxConcurrentQueries` configuration property
divides the query memory space by the value set. By setting this value, you
establish a limit on the maximum memory usage for each query. If a query exceeds
the allocated memory, it fails with an out-of-memory error.

This configuration property defaults to `3`, which means that a single query
can use up to one-third of the available memory space. To allocate more memory
for concurrent queries, decrease the value of this property.

All other {doc}`query processing properties
</admin/properties-query-management>` must be set using the
`additionalProperties:` node {ref}`on the coordinator <helm-sep-coordinator>`.

(helm-sep-spilling)=

### Spilling

The top level `query:` node ({ref}`example <k8s-ex-sep-hive-caching>`) allows
you to configure SEP's {doc}`spilling properties
<../admin/properties-spilling>`.

{doc}`Spilling <../admin/spill>` uses internal node storage, which is mounted
within the container.

:::{warning}
Spilling is disabled {ref}`by default <k8s-defaults-performance-spilling>`,
and we **strongly suggest** to leave it disabled. Enabling spill should be
used as a method of last resort to allow for rare, memory-intensive queries to
succeed on a smaller cluster at the expense of query performance and overall
cluster performance.
:::

(helm-sep-hive-caching)=

### Hive connector storage caching

The `cache:` top level node ({ref}`example <k8s-ex-sep-hive-caching>`) allows
you to configure {doc}`Hive connector storage caching
</connector/hive-caching>`. It is disabled {ref}`by default
<k8s-defaults-performance-cache>`.

:::{list-table} Hive connector caching property nodes
:widths: 30, 70
:header-rows: 1

* - Node name
  - Description
* - `cache.enabled`
  - Enable or disable caching for all catalogs using the Hive connector. If
    you want to only enable it for a specific catalog, you have to configure
    it with the {ref}`catalog configuration <helm-sep-catalogs>` and
    `additionalVolumes`.
* - `cache.diskUsagePercentage`
  - Set the value for the `hive.cache.disk-usage-percentage` property.
* - `cache.ttl`
  - Set the value for the `hive.cache.ttl` property.
* - `cache.volume`
  - Configure the volume in which to store the cached files.
:::

(helm-sep-catalogs)=

## Catalogs

The `catalogs:` top level node allows you to configure a catalog for each of
your data sources. The catalogs defined in this node are used to create catalog
properties files, which contain key-value pairs for catalog properties.
Information for specific properties supported in each catalog can be found with
the {doc}`documentation for the connectors </connector>`. At the very minimum, a
catalog definition must consist of the name of the name of the catalog, and the
`connector.name` property.

For best practices, use the YAML multi-line syntax shown in the {ref}`examples
<k8s-ex-sep-catalogs>` to configure the content in multiple lines indented
below the catalog name.

Defaults and examples:

- {ref}`k8s-defaults-catalogs`
- {ref}`k8s-ex-sep-catalogs`
- {ref}`k8s-ex-teradata-direct-connector`
- {doc}`Using the built in HMS with Hive connector and others <hms-configuration>`

(helm-sep-volumes)=

## Additional volumes

Additional [k8s volumes](https://kubernetes.io/docs/concepts/storage/volumes/#types-of-volumes) can be
necessary for persisting files, for {doc}`Hive object storage caching
</connector/hive-caching>`, and for a number of other use cases. These can be
defined in the `additionalVolumes` section ({ref}`example
<k8s-ex-sep-volumes>`). None are defined {ref}`by default
<k8s-defaults-additionalvolumes>`.

:::{list-table} `additionalVolume` property nodes
:widths: 30, 70
:header-rows: 1

* - Node name
  - Description
* - `path`
  - Specifies the path to the mounted volume. If you specify `path` only, a
    directory named in `path` is created. When mounting ConfigMap or Secret,
    files are created in this directory for each key.
* - `volume`
  - A directory, with or without data, which is [accessible to the
    containers in a pod](https://kubernetes.io/docs/concepts/storage/volumes/#background).
* - `hostpath`
  - A volume with a `hostPath` defined [mounts a file or
    directory](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath)
    from the host node's filesystem into your pod.
* - `hostpath.path`
  - **REQUIRED** when using `hostPath`-mounted volumes.
* - `subPath`
  - When specified, a specific key named `subPath` from ConfigMap or Secret
    is mounted as a file with name provided by `path`. This is an
    alternative to using `hostpath`; you cannot define both for a volume.
:::

(helm-sep-adding-files)=

### Adding files

Various use cases around security and event listeners need additional config
files as properties or XML files. You can add any file to a pod using [config
maps](https://kubernetes.io/docs/concepts/configuration/configmap/).

Types of files you may need to add:

- LDAP authentication file
- Hive site xml file

Alternatively you can also use {ref}`additionalVolumes <helm-sep-volumes>` to
mount the files and copy the files to appropriate location using `path` and
`subPath` parameter ({ref}`example <k8s-ex-sep-adding-files>`).

(helm-sep-prometheus)=

## Prometheus

This top level node configures the cluster to create [Prometheus metrics](https://kubernetes.io/docs/concepts/cluster-administration/system-metrics/).
It is not to be confused with the connector of the same name. It is enabled
{ref}`by default <k8s-defaults-prometheus>`.

We strongly suggest reviewing our {ref}`example Prometheus rules
<k8s-ex-sep-prometheus>`.

(helm-sep-directories)=

## SEP built-in directories

The location of specific directories on the SEP container is important, if you
configure additional files or otherwise want to tweak the container.

SEP is configured to use the following directories in the container:

- `/usr/lib/starburst`: Top level folder for the SEP binaries.
- `/usr/lib/starburst/plugin`: All plugins for SEP including connectors
- `/usr/lib/starburst/etc`: `etc` folder for SEP configuration files such as
  `config.properties` and others.
- `/usr/lib/starburst/bin`: Location of the `run-starburst` script, which is invoked
  with the container start.
- `/data/starburst/var/run`: Contains `launcher.pid` file used by the launcher
  script.
- `/data/starburst/var/log`: Contains secondary, rotated log files. Main log is
  redirected to stdout as recommended on containers.

(helm-sep-initcontainers)=

## Init containers

Init containers are a Kubernetes feature designed for initialization and setup
tasks within a pod. {{sb}} recommends using init containers as they provide
alignment with Kubernetes [best practices](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/#using-init-containers).

Using an init container within a pod allows you to prepare configuration and
plugin files with a different Docker image than your main SEP containers. The
init container's Docker image may contain additional tools and utilities that
are not available in the SEP image.

### Images

The SEP Docker image does not include command line tools such as `curl`,
`vi`, `nano`, `sed`, `awk`, and `grep`. However, there are several
lightweight images you can use for your init container that provide access to
some of these tools. Here are some images to consider:

- `ubi9/ubi-minimal` provides tools like `curl`, `sed`, and `grep`.
  However, it lacks `vi`.

  - For more information, see [Red Hat Universal Base Image 9 Minimal](https://catalog.redhat.com/software/containers/ubi9/ubi-minimal/615bd9b4075b022acc111bf5).

- `ubi9/ubi` provides tools like `curl`, `sed`, `grep`, and `vi`.

  - For more information, see [Red Hat Universal Base Image 9 Full](https://catalog.redhat.com/software/containers/ubi9/ubi/615bcf606feffc5384e8452e).

Using init containers with native cloud authentication capabilities may simplify
your tasks. There are official images provided by AWS, GCP, or Azure that come
with the relevant CLI tools pre-installed:

- `amazon/aws-cli`

  - For more information, see [AWS CLI](https://aws.amazon.com/cli/).

- `google/cloud-sdk`

  - For more information, see [Google Cloud SDK](https://cloud.google.com/sdk/docs/downloads-docker).

- `mcr.microsoft.com/azure-cli`

  - For more information, see [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/run-azure-cli-docker).

### Shared volumes

To make the files and data stored in your init container accessible to the main
containers in your pod, you must mount shared volumes in both the init
container and main containers. This ensures that your main containers can access
your init container's resources.

The following example demonstrates mounting a shared volume in all of your pod's
containers by defining an `emptyDir` volume in the `additionalVolumes`
section of the Helm chart:

```yaml
coordinator:
  initContainers:
  - name: prep-node-properties
    image: busybox:1.28
    command: ["sh"]
    args:
    - -c
    - |
      grep -v node.environment /etc/starburst/node.properties > /mnt/node.properties
      echo node.environment=$SEPENV >> /mnt/node.properties
additionalVolumes:
- path: /mnt
  volume:
    emptyDir: {}
initFile: |
  #!/bin/bash
  mv /mnt/node.properties2 /etc/starburst/node.properties
  exec /usr/lib/starburst/bin/run-starburst
```

:::{note}
The `volumeMounts` entries in the `initContainers` and `sidecars`
sections are automatically added for all `additionalVolumes`.
:::

For more information on additional volumes, see {ref}`additional volumes
<helm-sep-volumes>`.

### Moving files

You can use init containers to move files within your pod. This is useful for
transferring certificates or data from your init container to your pod's main
containers. The following example demonstrates how to use an init container to
move a file within your pod:

```yaml
coordinator:
  initContainers:
  - name: coordinator-pre-init
    image: registry.access.redhat.com/ubi9/ubi:latest
    command: ["sh"]
    args:
    - -c
    - |
      curl https://jdbc.postgresql.org/download/postgresql-42.2.16.jar --output postgresql.jar
      ls -l /downloads

worker:
  initContainers:
  - name: worker-pre-init
    image: registry.access.redhat.com/ubi9/ubi:latest
    command: ["sh"]
    args:
    - -c
    - |
      curl https://jdbc.postgresql.org/download/postgresql-42.2.16.jar --output postgresql.jar
      ls -l /downloads

additionalVolumes:
  - name: downloads
    path: /downloads
    volume:
      emptyDir: {}
```

### Sidecars

Extra sidecars and init containers can be specified to run together with
coordinator or worker pods. You can use these containers for a number of use
cases, such as to prepare an additional driver for the SEP runtime. The
following example demonstrates declaring both sidecars and init containers:

```yaml
coordinator:
  sidecars:
    - name: sidecar-1
      image: alpine:3
      command: [ "ping" ]
      args: [ "127.0.0.1" ]
  initContainers:
    - name: init-1
      image: alpine:3
      command: [ "bash", "-c", "printenv > /mnt/InContainer/printenv.txt" ]

worker:
  sidecars:
    - name: sidecar-3
      image: alpine:3
      command: [ "ping" ]
      args: [ "127.0.0.1" ]

additionalVolumes:
  - path: /mnt/InContainer
    volume:
      emptyDir: {}
```

(keda-sep-scaler)=

## KEDA scaler

[KEDA](https://keda.sh/) is a Kubernetes-based Event Driven Autoscaler. SEP
can be configured with an [external KEDA scaler](https://keda.sh/docs/2.6/concepts/external-scalers/) to adjust the number of
workers automatically based on JVM performance metrics available via JMX. Once
it is enabled with the `.Values.worker.kedaScaler.enabled` property, the
coordinator pod runs an additional container called `keda-trino-scaler`. This
container works as a GRPC service, and communicates with KEDA to scale workers.

:::{note}
The `.Values.worker.autoscaling` and `.Values.worker.kedaScaler.enabled`
properties enable mutually-exclusive features that cannot be enabled together
as [KEDA also uses HPA](https://keda.sh/docs/2.6/concepts/scaling-deployments/#scaling-of-deployments-and-statefulsets)
to scale Kubernetes Deployments.
:::

### Prerequisites

- [KEDA with Helm](https://keda.sh/docs/2.6/deploy/#helm) - versions 2.6.x and
  higher are required.
- Prometheus JMX Exporter - The SEP Helm chart ensures that the exporter
  is enabled, and configured with all necessary rules.

### Configuration

The following nodes and node sections configure the KEDA scaler:

:::{list-table} KEDA configuration nodes and sections for SEP
:widths: 30, 70
:header-rows: 1

* - Node or section name
  - Description
* - `.Values.worker.kedaScaler`
  - Set this node to `true` to enable the KEDA scaler.
* - `scaledObjectSpec` node structure
  - Corresponds to the parameters such as `pollingInterval`,
    `minReplicaCount`, `maxReplicaCount`, and others found in the [KEDA
    ScaledObject
    spec](https://keda.sh/docs/2.6/concepts/scaling-deployments/#scaledobject-spec).
* - `scaledObjectSpec.triggers[0].metadata` section
  - Defines the scaling method implemented by this scaler. Only the
    `query_queue` scaling method is available for SEP. It is enabled by
    default. The value for `query_queue` is the number of outstanding
    queries present on the cluster regardless of processing status.
* - `numberOfQueriesPerWorker`
  - Defines the maximum number of concurrent queries handled by a single
    worker. For example, if set to `10`, and there are currently `34`
    queries submitted but not completed, KEDA scales out the workers to
    `4` replicas if `maxReplicaCount >= 4`.
* - `scaleInToIdleReplicaCountIfNoQueuedQueriesLeft`
  - Reduces the number of workers to `idleReplicaCount` when the query queue
    is empty. KEDA requires the value for `idleReplicaCount` to be less than
    the value of `minReplicaCount`. Can be set to `0`.
:::

:::{note}
If you are using JMX metrics to monitor your deployment, you must explicitly
list those metrics in the JMX metrics whitelist located in the
`prometheus.whitelistObjectNames` section of your `values.yaml` file. This
allows KEDA to access the required JMX metrics for scaling.
:::

### Troubleshooting

When the KEDA scaler does not behave as expected, review the following logs:

- `keda-trino-scaler` container logs in the coordinator pod
- `keda-operator` logs in the `keda` namespace

(helm-troubleshooting-tips)=

## Helm troubleshooting tips

Here are some things to keep in mind as you implement SEP with Kubernetes and
Helm in your organization.

- Helm is space-sensitive and tabs are not valid. Our default
  `values.yaml` file uses 2-space indents. Ensure that your code editor
  preserves the correct indentation as it copies and pastes text.

- Special characters such as square brackets `[ ]`, curly braces `{ }`, and
  non-alphanumeric characters such as `$`, `&`, and `!` must be escaped or
  enclosed in quotes to prevent them being misinterpreted as part of YAML
  syntax. Escaping with a backslash indicates that a character should be treated
  as a literal character. Enclosing strings in single or double quotes ensures
  that the entire string is treated as plain text. For example:

  ```yaml
  password: 'ExamplePa$$word!'
  ```

- If your YAML files fail to parse, The Helm documentation provides [several
  tools to debug this issue](https://helm.sh/docs/chart_template_guide/debugging/).

- Sometimes problems can arise from improperly formatted YAML files.
  SEP uses Helm to build its `*.properties` files from YAML both with
  YAML key-value pairs and with multi-line strings. We recommend that you
  review the [YAML techniques](https://helm.sh/docs/chart_template_guide/yaml_techniques/)
  used in Helm to ensure that you are comfortable with using the Helm multi-line
  strings feature, and to understand the importance of consistent indentation.

- Units can also be a source of confusion. We strongly suggest that you pay
  close attention to any units regarding memory and storage. In general, units
  provided for multiline strings that feed into SEP configuration files are in
  traditional metric bytes, such as megabytes (MB) and gigabytes (GB) as used by
  SEP. However, machine sizing and other values  are in binary multiples such
  as mebibytes (Mi) and gibibytes (Gi), since these are used directly by Helm
  and Kubernetes.
