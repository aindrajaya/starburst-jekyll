---
layout: default
title: Configure the cache service in Kubernetes
parent: Kubernetes cluster design for Starburst Enterprise
grand_parent: Deploy with Kubernetes
nav_order: 7
permalink: /docs/kubernetes/kubernetes-cluster-design-for-starburst-enterprise/cache-service-configuration/
---

# Configure the cache service in Kubernetes

This topic covers configuring the {{sb}} cache service after you completed a basic
install of Starburst Enterprise. If you have not yet completed that, go to our
{doc}`installation guide </k8s/installation>`.

The `starburst-cache-service` Helm chart configures a standalone
{doc}`/admin/cache-service` to use with {doc}`/admin/starburst-cached-views`.

We strongly suggest that you {doc}`follow best practices
</k8s/sep-configuration>` to customize your cluster. Creating small, targeted
files to override any defaults and add any configuration properties. There is an
example file set in our deployment guide that describes the recommended way to
manage your customizations.

You must configure the following to use the {{sb}} cache service:

- The cache service
- Starburst Enterprise platform (SEP), to use the cache service
- The SEP {ref}`backend service <k8s-ex-insights>`

Make sure that the cache service is available before {ref}`configuring and
restarting SEP to use it <k8s-configure-sep-for-cache-service>`.

## Requirements

- Externally managed, {ref}`compatible relational database <cache-service-external-rdbms>`.
- Full access to a dedicated schema on the database with username and password
  credentials.
- Network access on the configured port between the database and the cache
  service container in the Kubernetes cluster.

## Configure and start the cache service

Get the latest `starburst-cache-service` Helm chart as described in our
{doc}`installation guide </installation>`.

There are several top-level nodes in the cache service Helm chart that you must
modify for a minimum cache service configuration.

- `config`
- `resources`
- `database`
- `expose`

For more information on configuring these nodes, see the
{ref}`k8s-cache-service-yaml-file` section.

As with SEP, we **strongly suggest** that you initially deploy the cache
service with the minimum configuration described in this topic, and ensure that
it deploys and is accessible before making any additional customizations
described in this documentation.

:::{note}
Store customizations in a separate file containing only changed values as
recommended in our {doc}`best practices guide </k8s/sep-configuration>`. In
this topic, for example, customizations are stored in a file named
`cache-service-values.yaml` that is used in the `helm upgrade` command.
:::

To deploy or update the cache service with any configuration changes, run the
`helm upgrade` command with the updated YAML files and the `--install`
switch. For example:

```shell
$ helm upgrade my-caching-service starburstdata/starburst-cache-service \
    --install \
    --version 434.0.0 \
    --values ./registry-access.yaml
    --values ./cache-service-prod.yaml
```

When you update the cache service, you can use the same command that you use to
upgrade to a new release. Helm compares all `--values` files and the version,
and safely ignores any that are unchanged.

(k8s-cache-service-yaml-file)=

## YAML file properties

The nodes included in the `values.yaml` file are described in the following
table.

:::{list-table} Top level `values.yaml` nodes
:widths: 30, 70
:header-rows: 1

* - Node name
  - Description
* - `image`
  - Contains the details for the cache service Docker image. Review our
    {ref}`best practices <minimal-values-YAML-file>` for managing registry
    access across all {{sb}} products. NOTE: The `image:` node is handled the
    same way as for the {ref}`Docker image and registry section for the Helm
    chart <helm-sep-image>` for SEP.
* - `keystore`
  - Contains the secret to be mounted under the specified
    `podFileLocation` within the container.
* - `config`
  - Specifies the configuration properties for the cache service
    under `config.properties`, its JVM config under `jvm.config`, and its
    logging configuration under `log.properties`. It also contains the
    `rules.json` and `type-mapping.json` nodes.
* - `registryCredentials`
  - Defines authentication details for Docker registry access. Typically, you
    need to use {ref}`your username and
    password for the Starburst Harbor instance <k8s-helm-repository>`. Cannot
    be used if using `imagePullSecrets`. NOTE: The
    `registryCredentials` node is handled the same way as for the
    {ref}`Docker image and registry section for the Helm chart
    <helm-sep-image>` for SEP.
* - `imagePullSecrets`
  - Alternative authentication for selected Docker registry using secrets.
    Cannot be used if using `registryCredentials`.
* - `resources`
  - The CPU and memory resources to use for the cache service. Request and
    limit values should be identical. These settings can be adjusted to match
    your workload and available node sizes in the cluster.
* - `expose`
  - Defines the mechanisms and their options that expose the cache service to
    an outside network. `type: "clusterIp"` is configured by default.
* - `database`
  - Defines the database backend for the cache service. Defaults to
    `type: "internal"`.
* - `envFrom`
  - Allows for the propagation of environment variables from different sources
    complying with the
    [K8S schema specification](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.25/#envfromsource-v1-core).
    This can be used to deliver values to the cache service configuration
    properties files by creating a Kubernetes secret holding variable values.
* - `env`
  - Allows to define additional environment variables for the cache service.
* - `nodeSelector`, `affinity` and `tolerations`
  - Configuration to allow Kubernetes to {ref}`determine the node and pod to
    use <helm-node-assignment>`. These nodes are left empty by default.
* - `commonLabels`
  - Defines common labels to identify all cache service objects in a KRM to
    use with the [kustomize utility](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/commonlabels/)
    and other tools.
:::

% source at
% https://github.com/starburstdata/starburst-platform-charts/blob/master/starburst-cache-service/helm/values.yaml

### `image`

The following are the default values for the cache service
`image` top level node in the `values.yaml` file:

```yaml
image:
  repository: "starburstdata/starburst-cache-service"
  tag: "434-e"
  pullPolicy: "IfNotPresent"
```

### `keystore`

The `keystore` section of the Helm chart configures keystore settings for the
cache service.

```yaml
keystore:
  localFileLocation: null
  podFileLocation: null
```

:::{list-table} Keystore configuration properties
:widths: 45, 55
:header-rows: 1

* - Property name
  - Description
* - `keystore.localFileLocation`
  - Specifies the local path of the keystore file.
* - `keystore.podFileLocation`
  - Specifies the path in the pod where the keystore file is mounted. You can
    provide a file path for a new folder or use an existing folder in the root
    directory such as `/mnt` or `/etc`.
:::

:::{note}
When you create a secret for the keystore file, you must name the secret
`keystore-configuration` for the cache service pod to recognize it.
:::

### `config`

The defaults for nodes nested under `config:` are described below.

#### `config.properties`

This node specifies {ref}`configuration properties and their values
<cache-service-config-properties>` for the cache service. You must define the
service accounts and locators to be used by the cache service:

```yaml
config:
  config.properties: |
    service-database.user=alice
    service-database.password=test123
    service-database.jdbc-url=jdbc:mysql://mysql-server:13306/cachesvc
    starburst.user=bob
    starburst.jdbc-url=jdbc:trino://coordinator:8080
    rules.file=etc/rules.json
```

#### `config:jvm.config`

This node specifies the {ref}`command line configuration options for starting
the Java Virtual Machine (JVM) <cache-service-jvm-config>` used by the cache
service. The following are the default values for the cache service
`config:jvm.config:` node in the `values.yaml` file:

```yaml
config:
  jvm.config: |
    -server
    --add-opens=java-base/sun.nio.ch=ALL-UNNAMED
    --add-opens=java-base/java.nio=ALL-UNNAMED
    --add-opens=java.base/java.lang=ALL-UNNAMED
    --add-opens=java.security.jgss/sun.security.krb5=ALL-UNNAMED
    -XX:G1HeapRegionSize=32M
    -XX:+ExplicitGCInvokesConcurrent
    -XX:+HeapDumpOnOutOfMemoryError
    -XX:+ExitOnOutOfMemoryError
    -XX:ReservedCodeCacheSize=512M
    -XX:PerMethodRecompilationCutoff=10000
    -XX:PerBytecodeRecompilationCutoff=10000
    -XX:+UnlockDiagnosticVMOptions
    -XX:+UseAESCTRIntrinsics
    -XX:InitialRAMPercentage=80
    -XX:MaxRAMPercentage=80
    -Djdk.nio.maxCachedBufferSize=2000000
    -Djdk.attach.allowAttachSelf=true
```

#### `log.properties`

This optional node specifies the {ref}`logging configuration
<cache-service-log-properties>` for the cache service. The following are the
default values for the cache service `log.properties:` node in the
`values.yaml` file:

```yaml
config:
  jvm.config: |
    io.airlift=INFO
```

#### `rules.json`

The `rules.json` node is specific to table scan redirections. It specifies the
source tables and target connector for the cache, along with the schedule for
refreshing them. The following are the default values for the cache service
`rules.json:` node in the `values.yaml` file:

```yaml
config:
  rules.json: |
    {
      "rules": []
    }
```

The following example demonstrates how to implement {ref}`cache service refresh
rules <table-scan-redirection-rule-sets>` in this node by adding the JSON file
content in a multi-line segment:

```yaml
config:
  rules.json: |
    {
      "defaultGracePeriod": "42m",
      "defaultMaxImportDuration": "1m",
      "defaultCacheCatalog": "default_cache_catalog",
      "defaultCacheSchema": "default_cache_schema",
      "rules": [
        {
          "catalogName": "mysql",
          "schemaName": "marketing",
          "tableName": "events",
          "refreshInterval": "2m",
          "gracePeriod": "15m",
          "incrementalColumn": "event_id",
          "deletePredicate": "event_date < date_add('day', -31, CURRENT_DATE)"
        }
      ]
    }
```

#### `type-mapping.json`

The `type-mapping.json` node specifies {ref}`type mapping rules
<cache-service-type-mapping>` between source and target catalogs. This
node is not included in the `values.yaml` file by default. The following
example maps  three different timestamp types to `TIMESTAMP(3)` in the target:

```yaml
config:
  type-mapping.json: |
    {
      "rules": {
        "tpch": {
          "integer": "long"
        },
        "hive": {
          "timestamp(0)": "timestamp(3)",
          "timestamp(1)": "timestamp(3)",
          "timestamp(2)": "timestamp(3)"
        }
      }
    }
```

### `registryCredentials`

The following are the default values for the cache service
`registryCredentials:` top level node in the `values.yaml` file:

```yaml
registryCredentials:
  enabled: false
  registry:
  username:
  password:
```

### `imagePullSecrets`

Instead of setting registryCredentials you can pass a list of secrets in the
following format. This feature is disabled by default:

```yaml
# imagePullSecrets:
#  - name: secret1
#  - name: secret2
imagePullSecrets:
```

### `resources`

The following values must be defined in the `resources` node of the cache
service Helm chart:

- **CPU resources for requests and limits:** The defaults are sufficient for
  most environments; however, they must work with the instance type you are
  using.
- **Memory resources for requests and limits:** The defaults are sufficient for
  most environments; however, they must work with the instance type you are
  using.

The following are the default values for the cache service `resources` top
level node in the `values.yaml` file:

```yaml
resources:
  requests:
    memory: 2Gi
    cpu: 0.5
  limits:
    memory: 2Gi
    cpu: 4
```

### `expose`

You must expose the service to allow it to connect to the SEP coordinator, and
to reach it with tools such as the {doc}`cache service CLI
<../admin/cache-service-cli>`. This service-type configuration is defined by the
`expose:` top level node. You can choose from four different mechanisms by
setting the `type:` value to the [common configurations in k8s](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types).

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
    service internally within the k8s cluster using an IP address internal
    to the cluster. Use this in the early stages of configuration.
* - `nodePort`
  - Configures the internal port number of the server for requests from
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

The following are the default values for the cache service `expose:` top level
node in the `values.yaml` file:

```yaml
expose:
  port: 8180
  # one of: nodePort, clusterIp, loadBalancer, ingress
  type: "clusterIp"
  clusterIp:
    name: "cache-service"
    ports:
      http:
        port: 8180
  nodePort:
    name: "cache-service"
    ports:
      http:
        port: 8180
        nodePort: 30180
  loadBalancer:
    name: "cache-service"
    IP: ""
    ports:
      http:
        port: 8180
    annotations: {}
    sourceRanges: []
  ingress:
    ingressName: "cache-service-ingress"
    serviceName: "cache-service"
    servicePort: 8180
    ingressClassName:
    tls:
      enabled: true
      secretName:
    host:
    path: "/"
    annotations: {}
```

#### Configure TLS (optional)

:::{note}
This is separate from configuring TLS on SEP itself.
:::

If your organization uses TLS, you must enable and configure your cache service
to work with it. The most straightforward way to handle TLS is to terminate TLS
at the load balancer or ingress, using a signed certificate. We **strongly
suggest** this method, which method requires no additional configuration in the
cache service.

If you choose not to handle TLS using that method, you can instead configure it
in the `expose` top-level node of the HMS Helm chart:

```yaml
expose:
  type: "[clusterIp|nodePort|loadBalancer|ingress]"
```

The default `type` is `clusterIp`. However, this is not suitable for
production environments. If you need help choosing which type is best, refer to
the {ref}`expose documentation <helm-sep-expose>` for SEP.

### `database.internal`

The configuration properties for the backend PostgreSQL database internal to the
cache service are found in the `database` top-level node. You can either use
default PostgreSQL database internal to this service or a self-managed, existing
{ref}`external database instance <cache-service-external-rdbms>`. We **strongly
suggest** using the default internal database. Use the `type` property to
select the type of database:

```yaml
database:
  type: "[internal|external]"
```

:::{note}
If you are using a self-managed, existing external database, ensure that it is
available before proceeding.
:::

As a minimal customization, you must ensure that the following are set correctly
for your environment:

```yaml
databaseName: "cacheservice"
databaseUser: "cacheservice"
databasePassword: "CacheServicePass1234"
```

You must also configure `volume` persistence options, if desired, as well as
the `resources` for the backing database itself in the `database` node.

:::{note}
The `database.resources` node is separate from the top level `resources`
node. It defines the resources available to the backing database itself, not
the cache service.
:::

In the `.Values.config.config.properties` configuration it is required to
refer to appropriate environment variables to enable integration with the
database backend:

```yaml
config.properties: |
  service-database.user=${ENV:SERVICE_DATABASE_USER}
  service-database.password=${ENV:SERVICE_DATABASE_PASSWORD}
  service-database.jdbc-url=${ENV:SERVICE_DATABASE_JDBC_URL}
```

The following snippet shows the default configuration for the internal cache
service backend database:

```yaml
database:
  type: internal
  internal:
    image:
      repository: "library/postgres"
      tag: "10.6"
      pullPolicy: "IfNotPresent"
    volume:
      # use one of:
      # - existingVolumeClaim to specify existing PVC
      # - persistentVolumeClaim to specify spec for new PVC
      # - other volume type inline configuration, e.g. emptyDir
      # Examples:
      # existingVolumeClaim: "my_claim"
      # persistentVolumeClaim:
      #  storageClassName:
      #  accessModes:
      #    - ReadWriteOnce
      #  resources:
      #    requests:
      #      storage: "2Gi"
      emptyDir: {}
    resources:
      requests:
        memory: "1Gi"
        cpu: 2
      limits:
        memory: "1Gi"
        cpu: 2
    driver: "org.postgresql.Driver"
    port: 5432
    databaseName: "cacheservice"
    databaseUser: "cacheservice"
    databasePassword: "CacheServicePass1234"
    envFrom: []
    env: []
```

:::{list-table} Internal cache service backend database configuration
:widths: 45, 55
:header-rows: 1

* - Node name
  - Description
* - `database.type`
  - Set to `internal` to use a database in the k8s cluster, managed by the
    chart
* - `database.internal.image`
  - Docker container images used for the PostgreSQL server
* - `database.internal.volume`
  - Storage volume to persist the database. The default configuration requests
    a new persistent volume (PV).
* - `database.internal.volume.persistentVolumeClaim`
  - The default configuration, which requests a new persistent volume (PV).
* - `database.internal.volume.existingVolumeClaim`
  - Alternative volume configuration, which uses an existing volume claim by
    referencing the name as the value in quotes, e.g., `"my_claim"`.
* - `database.internal.volume.emptyDir`
  - Alternative volume configuration, which configures an empty directory on
    the pod. Keep in mind that a pod replacement loses the database content
* - `database.internal.resources`
  -
* - `database.internal.databaseName`
  - Name of the internal database
* - `database.internal.databaseUser`
  - User to connect to the internal database
* - `database.internal.databasePassword`
  - Password to connect to internal database
* - `database.internal.envFrom`
  - YAML sequence of mappings
    [to define Secret or Configmap](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.25/#envfromsource-v1-core)
    as a source of environment variables for the PostgreSQL container.
* - `database.internal.env`
  - YAML sequence of mappings
    [to define two keys](https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/)
    environment variables for the PostgreSQL container.
:::

### `database.external`

This section shows the setup for using an {ref}`external PostgreSQL or MySQL
database instance <cache-service-external-rdbms>`. You must provide the
necessary details for the external server, and ensure that it can be reached
from the k8s cluster pod. Set the `database.type` to `external` and
configure the connection properties:

In the `.Values.config.config.properties` configuration it is required to
refer to the appropriate environment variables in order to enable integration
with the database backend:

```yaml
config.properties: |
  service-database.user=${ENV:SERVICE_DATABASE_USER}
  service-database.password=${ENV:SERVICE_DATABASE_PASSWORD}
  service-database.jdbc-url=${ENV:SERVICE_DATABASE_JDBC_URL}
```

```yaml
database:
  type: external
  external:
    jdbcUrl:
    user:
    password:
```

:::{list-table} External cache service backend database configuration
:widths: 45, 55
:header-rows: 1

* - Node name
  - Description
* - `database.type`
  - Set to `external` to use an existing PostgreSQL or MySQL database
    outside the cluster.
* - `database.external.jdbcUrl`
  - JDBC URL to connect to the external database as required by the database
    and used driver, including hostname and port. Ensure you use a valid JDBC
    URL as required by the PostgreSQL or MySQL driver. Typically the syntax
    requires the host, port and database name
    `jdbc:postgresql://host:port/database` or `jdbc:mysql://host:port/database`.
* - `database.external.user`
  - Database user name to access the external database using JDBC.
* - `database.external.password`
  - Password for the user configured to access the external database using
    JDBC.
:::

### `commonLabels`

The following are the default values for the `commonLabels:` top level node in
the cache service `values.yaml` file:

```yaml
commonLabels: {}
#  environment: dev
#  myLabel: labelValue
```

(k8s-configure-sep-for-cache-service)=

## Configure SEP to use the cache service

The cache service requires a {ref}`database schema to store configuration data
<cache-service-external-rdbms>`. Ensure that you have created the schema, and
note the connection information. Once the cache service and the external RDBMS
is configured and running, SEP must be configured as in the following example,
which shows a PostgreSQL database providing the backing schema:

```yaml
coordinator:
  etcFiles:
    properties:
      cache.properties: |
        service-database.user=postgres
        service-database.password=S3cr3t1v3
        service-database.jdbc-url=jdbc:postgresql://<your_rds_endpoint>:5432/redirections
        starburst.user=starburst_service
        starburst.password=
        starburst.jdbc-url=jdbc:trino://coordinator:8080
        rules.file=secretRef:cache-rules:cache-rules.json
        rules.refresh-period=1m
        refresh-initial-delay=1m
        refresh-interval=24h
```

Many connectors {doc}`support the use of the cache service
<../admin/table-scan-redirection>`. For each supported catalog that you wish to
use with the cache service, two lines must be added to the catalog properties
configuration:

```properties
redirection.config-source=SERVICE
cache-service.uri=http://cache-service:8180
```

In the following example, the `mysalesdata` catalog is configured to use the
cache service:

```yaml
catalogs:
  mysalesdata: |
    connector.name=postgresql
    connection-url=jdbc:postgresql://<mydbhost>:5432/bootcamp
    connection-user=postgres
    connection-password=S3cr3t1v3
    statistics.enabled=true
    redirection.config-source=SERVICE
    cache-service.uri=http://cache-service:8180
```

## Configuration examples

### External secret reference

To configure the cache service to work with the cache rules as an external
secret reference, first create a k8s secret holding the file:

```shell
kubectl create secret generic cache-rules --from-file=cache-rules.json
```

When the file is created, you can configure the secret reference usage for the
above configuration:

```yaml
config:
  config.properties: |
    service-database.user=${ENV:SERVICE_DATABASE_USER}
    service-database.password=${ENV:SERVICE_DATABASE_PASSWORD}
    service-database.jdbc-url=${ENV:SERVICE_DATABASE_JDBC_URL}
    starburst.user=bob
    starburst.jdbc-url=jdbc:trino://coordinator:8080
    rules.file=secretRef:cache-rules:cache-rules.json
```

This mounts the secret named `cache-rules` in the path
`/mnt/secretsRef/cache-rules` and replaces the `secretRef:cache-rules`
occurrences with the absolute path, resulting in the following configuration
property setting:

```text
rules.file=/mnt/secretRef/cache-rules/cache-rules.json
```

This mechanism can only be applied for properties files defined under
`.Values.config` node. Specific secret values, such as passwords, can be
passed into properties files using the `.Values.envFrom`.

## Next steps

Review the following topics to enable important performance features for your
SEP deployment:

- {ref}`Enable materialized views <sb-hive-mv-catalog-config>`
- \[Optional\] {doc}`Enable table scan redirection </admin/table-scan-redirection>`
