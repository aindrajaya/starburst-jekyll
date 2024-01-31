---
layout: default
title: Configuring the Hive Metastore Service in Kubernetes
parent: Kubernetes cluster design for Starburst Enterprise
grand_parent: Deploy with Kubernetes
nav_order: 5
permalink: /docs/kubernetes/kubernetes-cluster-design-for-starburst-enterprise/hms-configuration/
---

# Configuring the Hive Metastore Service in Kubernetes

The `starburst-hive` Helm chart configures the Hive Metastore Service, and
optionally the backend database in the cluster detailed in the following
sections. The following Starburst Enterprise platform (SEP) {doc}`connectors
</connector/starburst-connectors>` and features require a HMS:

- {doc}`/connector/starburst-hive`
- {doc}`/connector/starburst-delta-lake`
- {doc}`/connector/iceberg`
- and others

Use your {ref}`registry credentials <minimal-values-YAML-file>`, and follow
{doc}`best practices </k8s/sep-configuration>` by
creating an override file for changes to default values as desired.

In addition to the configuration properties described in this document, you can
also use the base Hive connector's {ref}`metastore configuration properties
<general-metastore-properties>`, the {ref}`Thrift metastore configuration
properties <hive-thrift-metastore>`, and the {ref}`AWS Glue metastore
configuration properties <hive-glue-metastore>` as needed, depending on your
environment.

% doc all things from
% https://github.com/starburstdata/starburst-platform-charts/blob/master/starburst-hive/helm/values.yaml

## Before you begin

Get the latest `starburst-hive` Helm chart as described in our
{doc}`installation guide <installation>` with the {ref}`configured registry
access <minimal-values-YAML-file>`.

## Configure the Hive Metastore

There are several top-level nodes in the HMS Helm chart that you must modify for
a minimum HMS configuration:

- `serviceAccountName`
- `resources`
- `database`
- `expose`
- `hiveMetastoreWarehouseDir`
- `hdfs`
- `objectStorage`

As with SEP, we **strongly suggest** that you initially deploy
HMS with the minimum configuration described in this topic and ensure that
it deploys and is accessible before making any additional customizations
described in our reference documentation.

:::{note}
Store customizations in a separate file containing only changed values as
recommended in our best practices guide. In this topic for example,
customizations are stored in a file named `hms-values.yaml` that is used in
the Helm `upgrade` command.
:::

(helm-hms-serviceaccount)=

### Configure resources and service account

Ensure that the following top-level nodes of the Helm chart have the correct
values to reflect your environment:

- `serviceAccountName`: we strongly recommend using a [service account](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)
  for the pod.
- `resources`: ensure that the CPU and memory sizes are appropriate for your
  instance type.

The default values for the `resources` node are as follows:

```yaml
heapSizePercentage: 85

resources:
  requests:
    memory: "1Gi"
    cpu: 1
  limits:
    memory: "1Gi"
    cpu: 1
```

:::{caution}
We strongly suggest leaving the `heapSizePercentage` at the default value.
:::

(helm-hms-database)=

### Configure the PostgreSQL backend database

The configuration properties for the internal PostgreSQL backend database are
found in the `database` top-level node. As a minimum configuration, you must
ensure that the following are set correctly for your environment:

```yaml
database:
  internal:
    databaseName: hive
    databasePassword: HivePass1234
    databaseUser: hive
    driver: org.postgresql.Driver
    port: 5432
  type: internal
```

:::{note}
Alternatively, you can use an {ref}`external backend database
<hms-external-backend-db>` for production usage that you must manage
yourself.
:::

The following table lists the available backend database configuration
properties:

:::{list-table} Internal HMS backend database configuration
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
  - Alternative volume configuration, which use existing volume claim by
    referencing the name as the value in quotes, e.g., `"my_claim"`.
* - `database.internal.volume.emptyDir`
  - Alternative volume configuration, which configures an empty directory on
    the pod, keeping in mind that a pod replacement loses the database
    content.
* - `database.internal.resources`
  -
* - `database.internal.databaseName`
  - Name of the internal database
* - `database.internal.databaseUser`
  - User to connect to the internal database
* - `database.internal.databasePassword`
  - Password to connect to internal database
* - `database.internal.envFrom`
  - YAML sequence of mappings [to define Secret or Configmap](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.25/#envfromsource-v1-core)
    as a source of {ref}`environment variables <helms-hms-environment-variables>`
    for the PostgreSQL container.
* - `database.internal.env`
  - YAML sequence of mappings [to define two keys](https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/)
    environment variables for the PostgreSQL container.
:::

:::{note}
The `database.resources` node is separate from the top-level `resources`
node. It defines the resources available to the backing database itself, not
the HMS server.
:::

#### Examples

OpenShift deployments often do not have access to pull from the default Docker
registry `library/postgres`. You can replace it with an image from the Red Hat
registry, which requires additional {ref}`environment variables
<helms-hms-environment-variables>` set with the parameter
`database.internal.env`:

```yaml
database:
  type: internal
  internal:
    image:
       repository: "registry.redhat.io/rhscl/postgresql-96-rhel7"
       tag: "latest"
    env:
      - name: POSTGRESQL_DATABASE
        value: "hive"
      - name: POSTGRESQL_USER
        value: "hive"
      - name: POSTGRESQL_PASSWORD
        value: "HivePass1234"
```

Another option is to create a Secret (ex. `postgresql-secret`) containing
variables needed by `postgresql` which are mentioned in previous code block,
and pass it to the container with `envFrom` parameter:

```yaml
database:
  type: internal
  internal:
    image:
       repository: "registry.redhat.io/rhscl/postgresql-96-rhel7"
       tag: "latest"
    envFrom:
      - secretRef:
          name: postgresql-secret
```

(hms-external-backend-db)=

### External backend database for HMS

This section shows the setup for using of an external PostgreSQL, MySQL, Oracle,
or Microsoft SQL Server database. You must provide the necessary details for the
external server, and ensure that it can be reached from the k8s cluster pod. Set
the `database.type` to `external` and configure the connection properties:

```yaml
database:
  type: external
  external:
    driver:
    jdbcUrl:
    password:
    setPasswordsViaEnvFrom: false
    user:
```

:::{list-table} External HMS backend database configuration
:widths: 45, 55
:header-rows: 1

* - Node name
  - Description
* - `database.type`
  - Set to `external` to use an existing PostgreSQL, MySQL, Oracle, or SQL
    Server database outside the cluster.
* - `database.external.jdbcUrl`
  - JDBC URL to connect to the external database as required by the database
    and used driver, including hostname and port. Ensure you use a valid JDBC
    URL as required by the PostgreSQL, MySQL, Oracle, or SQL Server driver.
    Typically, the syntax requires the host, port, and database name as
    follows:

    - For **PostgreSQL**: `jdbc:postgresql://host:port/database`.

    - For **MySQL**: `jdbc:mysql://host:port/database`.

    - For **Oracle**, using the thin or OCI driver:
      `jdbc:oracle:<driver>:@//host:port/<service_name>`.

    - For **SQL Server**: The database name must be passed as an
      {ref}`environment variable <helms-hms-environment-variables>`
      `HIVE_METASTORE_DB_NAME` and the JDBC connection string syntax
      `jdbc:sqlserver://host:port`.
* - `database.external.driver`
  - Valid values are as follows:

    - For **PostgreSQL**: `org.postgresql.Driver`.
    - For an external **MySQL** or a compatible database:
      `com.mysql.jdbc.Driver`.
    - For **Oracle**: `oracle.jdbc.OracleDriver`.
    - For **SQL Server**: `com.microsoft.sqlserver.jdbc.SQLServerDriver`.
* - `database.external.user`
  - Database user name to access the external database using JDBC.
* - `database.external.password`
  - Password for the user configured to access the
    external database using JDBC.
:::

(helm-hms-expose)=

## Expose the pod to the outside network

The `expose` section of the `starburst-hive` Helm chart works similarly to
the SEP server {ref}`expose section <helm-sep-expose>`. Differences are
isolated to the configured default values. Additionally, `ingress` is not
supported, as the HMS service uses the TCP protocol and not HTTP.

By default, the HMS is available at the hostname `hive` and port `9083`. As
a result, the default Thrift URL for the cluster is `thrift://hive:9083`.
Ensure to adapt your configured catalogs to use the correct Thrift URL. You can
use the URL for {ref}`any catalog <helm-sep-catalogs>`:

```yaml
catalog:
  datalake: |
    connector.name=hive
    hive.metastore.uri=thrift://hive:9083
```

The default `type` is `clusterIp`:

```yaml
expose:
  type: "clusterIp"
  clusterIp:
    name: "hive"
    ports:
      http:
        port: 9083
```

You can also use `nodePort`:

```yaml
expose:
  type: "nodePort"
  nodePort:
    name: "hive"
    ports:
      http:
        port: 9083
        nodePort: 30083
```

Additionally, you can use `loadBalancer`:

```yaml
expose:
  type: "loadBalancer"
  loadBalancer:
  name: "hive"
  IP: ""
  ports:
    http:
      port: 9083
  annotations: {}
  sourceRanges: []
```

(helm-hms-storage)=

### Configure storage

Configure a connection to the Hive Metastore Service (HMS), the Hadoop
Distributed File System (HDFS), or an object store using the following three
top-level nodes:

- `hiveMetastoreWarehouseDir`
- `hdfs`
- `objectStorage`

:::{note}
You must also {ref}`configure the catalog <helm-sep-catalogs>` with the
appropriate credentials.
:::

The default configuration for each of these properties is empty.

Add the location of your Hive metastore's warehouse directory to the
`hiveMetastoreWarehouseDir` node to enable the HMS to store metadata and
gather statistics. In the `hdfs` node, add the `hadoopUserName` you use to
connect to the warehouse directory.

There are several templates for configuring object storage in the
`objectStorage` node. For example, you can define how to connect to S3:

```yaml
objectStorage:
awsS3:
  accessKey:
  endpoint:
  pathStyleAccess: false
  region:
  secretKey:
```

The Helm chart also includes templates for Azure, Azure Data Lake, and Google
object storage.

:::{note}
For AWS, you can provide access to S3 using secrets, or by using IAM
credentials attached to the metastore pod.
:::

The following table lists the available storage configuration properties:

:::{list-table} HMS storage configuration
:widths: 45, 55
:header-rows: 1

* - Node name
  - Description
* - `hiveMetastoreWarehouseDir`
  - The location of your Hive Metastore's warehouse directory. For example,
    `s3://example/hive-default-warehouse/`.
* - `hdfs.hadoopUserName`
  - User name for Hadoop HDFS access
* - `objectStorage.awsS3.*`
  - Configuration for AWS S3 access
* - `objectStorage.awsS3.region`
  - AWS region name
* - `objectStorage.awsS3.endpoint`
  - AWS S3 endpoint, for example
    `http[s]://<bucket>.s3-<AWS-region>.amazonaws.com`.
* - `objectStorage.awsS3.accessKey`
  - Name of the access key for AWS S3
* - `objectStorage.awsS3.secretKey`
  - Name of the secret key for AWS S3
* - `objectStorage.awsS3.pathStyleAccess`
  -
* - `objectStorage.gs.*`
  - Configuration for Google Storage access
* - `objectStorage.gs.cloudKeyFileSecret`
  - Name of the secret with the file containing the access key to the cloud
    storage. The key of the secret must be named `key.json`.
* - `objectStorage.azure.*`:
  - Configuration for Microsoft Azure storage systems
* - `objectStorage.azure.abfs.*`
  - Configuration for Azure Blob Filesystem (ABFS)
* - `objectStorage.azure.abfs.authType`
  - Authentication to access ABFS, Valid values are`accessKey` or `oauth`,
    configuration in the following properties
* - `objectStorage.azure.abfs.accessKey.*`
  - Configuration for access key authentication to ABFS
* - `objectStorage.azure.abfs.accessKey.storageAccount`
  - Name of the ABFS account to access
* - `objectStorage.azure.abfs.accessKey.accessKey`
  - Actual access key to use for ABFS  access
* - `objectStorage.azure.abfs.oauth.*`
  - Configuration for OAuth authentication to ABFS
* - `objectStorage.azure.abfs.oauth.clientId`
  - Client identifier for OAuth authentication
* - `objectStorage.azure.abfs.oauth.secret`
  - Secret for OAuth
* - `objectStorage.azure.abfs.oauth.endpoint`
  - Endpoint URL for OAuth
* - `objectStorage.azure.wasb.*`
  - Configuration for Windows Azure Storage Blob (WASB)
* - `objectStorage.azure.wasb.storageAccount`
  - Name of the storage account to use for WASB
* - `objectStorage.azure.wasb.accessKey`
  - Key to access WASB
* - `objectStorage.azure.adl`
  - Configuration for Azure Data Lake (ADL)
* - `objectStorage.azure.adl.oauth2.*`
  - Configuration for OAuth authentication to ADL
* - `objectStorage.azure.adl.oauth2.clientId`
  - Client identifier for OAuth access to ADL
* - `objectStorage.azure.adl.oauth2.credential`
  - Credential for OAuth access to ADL
* - `objectStorage.azure.adl.oauth2.refreshUrl`:
  - Refresh URL for the OAuth access to ADL
:::

More information about the configuration options is available in the following
resources:

- {doc}`/connector/starburst-hive`
- {doc}`/connector/hive`
- {doc}`/connector/hive-azure`
- {doc}`/connector/hive-s3`
- {doc}`/connector/starburst-hive-cdp`
- {doc}`/connector/hive-security`

(helm-hms-avro)=

#### Metastore configuration for Avro

In order to enable {ref}`Avro tables <hive-avro-schema>`
when using Hive 3.x, you need to add the following property definition to the
Hive metastore configuration file `hive-site.xml`:

```xml
<property>
     <name>metastore.storage.schema.reader.impl</name>
     <value>org.apache.hadoop.hive.metastore.SerDeStorageSchemaReader</value>
 </property>
```

For more information about additional files, see {ref}`helm-sep-adding-files`.

### Configure TLS (optional)

:::{note}
This is separate from configuring TLS on SEP itself.
:::

If your organization uses TLS, you can enable and configure your HMS to work
with it. The most straightforward way to handle TLS is to terminate TLS at the
load balancer or ingress, using a signed certificate. We **strongly suggest**
this method, which requires no additional configuration in the HMS.

If you choose not to handle TLS using that method, you can instead configure it
in the `expose` top-level node of the HMS Helm chart:

```yaml
expose:
  type: "[clusterIp|nodePort|loadBalancer|ingress]"
```

The default `type` is `clusterIp`. For details on configuring each of these
types, see {ref}`exposing the pod to the outside network <helm-hms-expose>`.

## Additional settings

(helm-hms-startup)=

### Server start up configuration

You can configure a startup shell script for the HMS using the following
variables:

- `initFile`
- `extraArguments`

#### `initFile`

Use `initfile` to pass a shell script to run before HMS is launched. The
content of the script must be an inline string. The original startup command is
passed as the first argument at the end of the script as `exec "$@"`. Use
`exec "$1"` to any additional arguments.

#### `extraArguments`

Use `extraArguments` to pass a list of additional arguments to the
`initFile` script.

The following example shows how you can use `initFile` and `extraArguments`
to run a custom startup script. The `initFile` script must end with `exec
"$@"`:

```yaml
initFile: |
  #!/bin/bash
  echo "Custom init for $2"
  exec "$@"
extraArguments:
  - TEST_ARG
```

(helm-hms-image)=

### Docker image and registry

The Helm chart for the HMS uses a similar configuration for its {ref}`Docker
image and registry section <helm-sep-image>` as the Helm chart for SEP.

```yaml
image:
  pullPolicy: "IfNotPresent"
  repository: "harbor.starburstdata.net/starburstdata/hive"
  tag: "3.1.3-e.3"

registryCredentials:
  enabled: false
  password:
  registry:
  username:

imagePullSecrets:
```

(helm-hms-volumes)=

### Additional volumes

Additional volumes may be necessary for persisting files. These can be defined
in `additionalVolumes`. None are defined by default:

```yaml
additionalVolumes: []
```

You can add one or more [volumes supported by k8s](https://kubernetes.io/docs/concepts/storage/volumes/#types-of-volumes) to
all nodes in the cluster.

Specify a `path` to create a directory to store the keys for your ConfigMap or
Secret.

You may also specify an optional `subPath` parameter which takes an optional
key in the ConfigMap or Secret volume you create. If you specify `subPath`, a
key named `subPath` from your ConfigMap or Secret is mounted as a file within
the directory specified in `path`.

```yaml
additionalVolumes:
  - path: /mnt/InContainer
    volume:
      emptyDir: {}
  - path: /etc/hive/conf/test_config.txt
    subPath: test_config.txt
    volume:
      configMap:
        name: "configmap-in-volume"
```

### Node assignment

You can configure your cluster to use a specific node and pod for the HMS:

```yaml
nodeSelector: {}
tolerations: []
affinity: {}
```

Our {ref}`SEP configuration documentation <helm-node-assignment>` contains
examples and resources to help you configure these YAML nodes.

### Annotations

You can annotate your deployment or pods using the following variables:

- `deploymentAnnotations`
- `podAnnotations`

(helm-hms-securitycontext)=

### Security context

You can optionally configure [security contexts](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-the-security-context-for-a-pod)
to specify privileges and access control settings for your HMS pods.

```yaml
securityContext:
```

If you do not want to set the serviceContext for the `default`
service account, you can restrict it by configuring the {ref}`service account
<helm-hms-serviceaccount>` for the HMS pod.

(helms-hms-environment-variables)=

### Environment variables

You can pass environment variables to your HMS container using the same
variables as the internal database:

```yaml
envFrom: []
env: []
```

Both variables are specified as mapping sequences. For example:

```yaml
envFrom:
  - secretRef:
      name: my-secret-with-vars
env:
  - name: MY_VARIABLE
    value: some-value
```
