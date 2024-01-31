---
layout: default
title: Configuring Starburst Enterprise with Ranger in Kubernetes
parent: Kubernetes cluster design for Starburst Enterprise
grand_parent: Deploy with Kubernetes
nav_order: 6
permalink: /docs/kubernetes/kubernetes-cluster-design-for-starburst-enterprise/ranger-configuration/
---

# Configuring Starburst Enterprise with Ranger in Kubernetes

The `starburst-ranger` Helm chart configures Apache Ranger 2.3.0 usage in the
cluster with the `values.yaml` file detailed in the following sections. It
allows you to implement {doc}`global access control </security/ranger-overview>`
or just {doc}`Hive access control </security/hive-ranger>` with Ranger for
Starburst Enterprise platform (SEP).

It creates the following setup:

- A pod with the {ref}`Ranger server <helm-ranger-server>` with the Ranger
  plugin deployed and configured to connect to SEP
- Optionally a container in the pod with {ref}`Ranger LDAP user synchronization
  system <helm-ranger-usersync>` deployed and configured
- Optionally a container in the pod with a {ref}`PostgreSQL database backend
  <helm-ranger-database>` for Ranger

Use your {ref}`registry credentials <minimal-values-YAML-file>`, and follow
{doc}`best practices </k8s/sep-configuration>` by
creating an override file for changes to default values as desired.

% Doc all things in
% https://github.com/starburstdata/starburst-platform-charts/blob/master/starburst-ranger/helm/values.yaml

## Before you begin

Get the latest `starburst-ranger` Helm chart as described in our
{doc}`Kubernetes guide <installation>` with the {ref}`configured registry
access <minimal-values-YAML-file>`.

This topic assumes you are familiar with Ranger, as well as with Helm charts and
Kubernetes (k8s) tools such as `kubectl`. Ensure that you are familiar with
the following Starburst Enterprise Kubernetes topics before configuring and
deploying Ranger:

- {doc}`Kubernetes best practices </k8s/sep-configuration>`
- {ref}`Kubernetes requirements <requirements>`

## Configure Ranger

There are several top-level nodes in the Ranger Helm chart that you must modify
for a minimum Ranger configuration:

- `admin`
- `usersync`
- `database`
- `expose`

If you are using TLS, this must also be considered. This section covers getting
started with these four configuration steps. It also provides details about the
content of the Ranger Helm chart.

As with SEP, we **strongly suggest** that you initially deploy
Ranger with the minimum configuration described in this topic, and ensure that
it deploys and is accessible before making any additional customizations
described in our reference documentation.

:::{note}
Store customizations in a separate file containing only changed values as
recommended in our {doc}`best practices </k8s/sep-configuration>` In this
topic, customizations are stored in a file named `ranger-values.yaml` that
is used in the `helm upgrade` command.
:::

(helm-ranger-server)=

### Configure the Ranger administration container

The following values must be defined in the `admin` node of the Ranger Helm
chart:

- **CPU resources for requests and limits:** The defaults are sufficient for
  most environments; however, they must work with the instance type you are
  using.
- **Memory resources for requests and limits:** The defaults are sufficient for
  most environments; however, they must work with the instance type you are
  using.
- **Passwords:** You must supply *all* passwords in the `passwords` node.

The `admin` section configures the Ranger server and the included user
interface for policy management.

```yaml
admin:
  image:
    repository: "harbor.starburstdata.net/starburstdata/starburst-ranger-admin"
    tag: "2.3.0-e.9"
    pullPolicy: "IfNotPresent"
  port: 6080
  resources:
    requests:
      memory: "1Gi"
      cpu: 2
    limits:
      memory: "1Gi"
      cpu: 2
  # serviceUser is used by SEP to access Ranger
  serviceUser: "starburst_service"
  passwords:
    admin: "RangerPassword1"
    tagsync: "TagSyncPassword1"
    usersync: "UserSyncPassword1"
    keyadmin: "KeyAdminPassword1"
    service: "StarburstServicePassword1"
  # optional truststore containing CA certificates to use instead of default one
  truststore:
    # existing secret containing truststore.jks key
    secret:
    # password to truststore
    password:
  # Enable the propagation of environment variables from Secrets and Configmaps
  envFrom: []
  env:
    # Additional env variables to pass to Ranger Admin.
    # To pass Ranger install property, use variable with name RANGE__<property_name>,
    # for example RANGER__authentication_method.
  securityContext: {}
    # Optionally configure a security context for the ranger admin container
```

`admin.serviceUser`

The operating system user that is used to run the Ranger application.

`admin.passwords`

A number of passwords need to be set to any desired values. They are used for
administrative and Ranger internal purposes and do not need to be changed or
used elsewhere.

(helm-ranger-usersync)=

### Configure Ranger user synchronization

User synchronization automates the process of adding users to Ranger for policy
enforcement by allowing the synchronization of users and groups from LDAP,
including Active Directory.

You can use the `usersync` block to configure the details of the
synchronization of users and groups between Ranger and your LDAP system, as
alternative to {ref}`direct sync between SEP and Ranger <ranger-sep-usersync>`.
It runs on a separate sidecar container when deployed.

At a minimum, the `env` properties in the top-level `usersync` node must be
defined correctly for your environment. The default configuration enables user
synchronization:

```yaml
usersync:
  enabled: true
  env:
    # Use RANGER__<property_name> variables to set Ranger install properties.
    RANGER__SYNC_GROUP_OBJECT_CLASS: groupOfNames
    RANGER__SYNC_GROUP_SEARCH_BASE: ou=groups,dc=ldap,dc=example,dc=org
    RANGER__SYNC_GROUP_SEARCH_ENABLED: "true"
    RANGER__SYNC_GROUP_USER_MAP_SYNC_ENABLED: "true"
    RANGER__SYNC_LDAP_BIND_DN: cn=admin,dc=ldap,dc=example,dc=org
    RANGER__SYNC_LDAP_BIND_PASSWORD: p@ssw0rd!
    RANGER__SYNC_LDAP_SEARCH_BASE: dc=ldap,dc=example,dc=org
    RANGER__SYNC_LDAP_URL: ldap://ranger-ldap:389
    RANGER__SYNC_LDAP_USER_OBJECT_CLASS: person
    RANGER__SYNC_LDAP_USER_SEARCH_BASE: ou=users,dc=ldap,dc=example,dc=org
  securityContext:
    # Optionally configure a security context for the ranger usersync container
```

:::{list-table} User synchronization configuration properties
:widths: 45, 55
:header-rows: 1

* - Node name
  - Description
* - `usersync.enabled`
  - Enables or disables user synchronization feature
* - `usersync.name`
  - Name of the pod
* - `usersync.tls.truststore.secret`
  - Name of the secret created from the truststore. This is required if you
    need to use tls for usersync.
* - `usersync.tls.truststore.password`
  - Password for the truststore. This is required if you need to use tls for
    usersync.
* - `usersync.env`
  - A map of Ranger config variables related to the user synchronization
* - `usersync.env.RANGER__SYNC_LDAP_URL`
  - URL to the LDAP server
* - `usersync.env.RANGER__SYNC_LDAP_BIND_DN`
  - Distinguished name (DN) string used to bind for the LDAP connection
* - `usersync.env.RANGER__SYNC_LDAP_BIND_PASSWORD`
  -
* - `usersync.env.RANGER__SYNC_LDAP_SEARCH_BASE`
  -
* - `usersync.env.RANGER__SYNC_LDAP_USER_SEARCH_BASE`
  - User information search base in the LDAP directory
* - `usersync.env.RANGER__SYNC_LDAP_USER_OBJECT_CLASS`
  - Object class for users
* - `usersync.env.RANGER__SYNC_GROUP_SEARCH_ENABLED`
  - Enable or disable group search
* - `usersync.env.RANGER__SYNC_GROUP_USER_MAP_SYNC_ENABLED`
  - Enable or disable synchronization of group-user mapping
* - `usersync.env.RANGER__SYNC_GROUP_SEARCH_BASE`
  - Group information search base in the LDAP directory
* - `usersync.env.RANGER__SYNC_GROUP_OBJECT_CLASS`
  - Object class for groups, typically `groupOfNames` for OpenLDAP or
    `group` for Active Directory
:::

The following steps can be used to enable TLS with the LDAP server:

- Create a truststore file named `truststore.jks` from the LDAP server

- Create a Kubernetes secret `ldap-cert` from the truststore file

  ```shell
  kubectl create secret generic ldap-cert --from-file truststore.jks
  ```

- Update values to reflect the secret name in the `tls` section

- Update truststore password in the `tls` section

  ```yaml
  tls:
    enabled: true
    truststore:
      secret: ldap-cert
      password: "truststore password"
  ```

#### SEP user synchronization

SEP can {ref}`actively sync user names and user groups to directly Ranger
<ranger-sep-usersync>` as a simpler alternative to {ref}`helm-ranger-usersync`.\`

(helm-ranger-database)=

### Configure the PostgreSQL backend database

The configuration properties for the internal PostgreSQL backend database that
stores policy information are found in the `database` top-level node.

:::{note}
Alternatively, you can use an {ref}`external PostgreSQL database
<ranger-external-backing-db>` for production usage that you must manage
yourself.
:::

As a minimal configuation, you must ensure that the following are set correctly
for your environment:

```yaml
database:
  type: "internal"
  internal:
    port: 5432
    databaseName: "ranger"
    databaseUser: "ranger"
    databasePassword: "RangerPass123"
    databaseRootUser: "rangeradmin"
    databaseRootPassword: "RangerAdminPass123"
```

You may also configure `volume` persistence and resources, as well as the
`resources` for the backing database itself in the `database` node.

:::{list-table} Internal backend database server configuration properties
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
* - `database.internal.databaseRootUser`
  - User to administrate the internal database for creating and updating
    tables and similar operations.
* - `database.internal.databaseRootPassword`
  - Password for the administrator to connect to the the internal database
* - `database.internal.envFrom`
  - YAML sequence of mappings
    [to define Secret or Configmap](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.25/#envfromsource-v1-core)
    as a source of {ref}`environment variables <helm-ranger-environment-variables>`
    for the internal PostgreSQL container.
* - `database.internal.env`
  - YAML sequence of mappings to
    [define two key environment variables](https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/)
    for the internal PostgreSQL container.
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
variables needed by `postgresql` mentioned in previous code block, and pass it
to the container with `envFrom` parameter:

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

(ranger-external-backing-db)=

#### External backend database server

This section shows the empty default setup for using of an external PostgreSQL
database. You must provide the necessary details for the external server and
ensure that it can be reached from the k8s cluster pod.

```yaml
database:
  type: "external"
  external:
    port:
    host:
    databaseName:
    databaseUser:
    databasePassword:
    databaseRootUser:
    databaseRootPassword:
```

:::{list-table} External backend database server configuration properties
:widths: 45, 55
:header-rows: 1

* - Node name
  - Description
* - `database.type`
  - Set to `external` to use a database managed externally
* - `database.external / port`
  - Port to access the external database
* - `database.external / host`
  - Host of the external database
* - `database.external / databaseName`
  - Name of the database
* - `database.external / databaseUser`
  - User to connect to the database. If the user does not already exist, it is
    automatically created during installation.
* - `database.external / databasePassword`
  - Password to connect to the database
* - `database.external / databaseRootUser`
  - The existing root user to administrate the external database. It is used
    to create and update tables and similar operations.
* - `database.external / databaseRootPassword`
  - Password for the administrator to connect to the external database
:::

(helm-ranger-expose)=

### Expose the cluster to the outside network

The `expose` section for Ranger works identical to the {ref}`expose section
<helm-sep-expose>` for SEP. It exposes the Ranger user interface for
configuring and managing policies outside the cluster.

Differences are isolated to the configured default values. The default type is
`clusterIp`:

```yaml
expose:
  type: "clusterIp"
  clusterIp:
    name: "ranger"
    ports:
      http:
        port: 6080
```

The following section shows the default values with an activated `nodePort`
type:

```yaml
expose:
  type: "nodePort"
  nodePort:
    name: "ranger"
    ports:
      http:
        port: 6080
        nodePort: 30680
```

The following section shows the default values with an activated
`loadBalancer` type:

```yaml
expose:
  type: "loadBalancer"
  loadBalancer:
    name: "ranger"
    IP: ""
    ports:
      http:
        port: 6080
    annotations: {}
    sourceRanges: []
```

The following section shows the default values with an activated `ingress`
type:

```yaml
expose:
  type: "ingress"
  ingress:
    ingressName: "ranger-ingress"
    serviceName: "ranger"
    servicePort: 6080
    ingressClassName:
    tls:
      enabled: true
      secretName:
    host:
    path: "/"
    annotations: {}
```

(helm-ranger-tls)=

### Configure TLS (optional)

:::{note}
This is separate from configuring TLS in SEP itself.
:::

If your organization uses TLS, you must enable and configure Ranger to work with
it. The most straightforward way to handle TLS is to terminate TLS at the load
balancer or ingress, using a signed certificate. We **strongly suggest** this
method, which requires no additional configuration in Ranger. Ranger can
also be configured to [listen on HTTPS directly](https://cwiki.apache.org/confluence/display/RANGER/Index).

If you choose not handle TLS using those methods, you can instead configure it
in the {ref}`usersync <helm-ranger-usersync>` and {ref}`expose
<helm-ranger-expose>` nodes of the Ranger Helm chart. The following snippets
show the nodes of each of these sections:

```yaml
usersync:
    tls:
    # optional truststore containing CA certificate for ldap server
    truststore:
      # existing secret containing truststore.jks key
      secret:
      # password to truststore
      password:

expose:
  type: "[clusterIp|nodePort|loadBalancer|ingress]"
```

The default `expose` type is `clusterIp`. However, this is not suitable for
production environments. If you need help choosing which type is best, refer to
the {ref}`expose documentation <helm-sep-expose>` for SEP.

The process of enabling TLS between Ranger and SEP with the Helm chart is
{ref}`identical to the normal process <ranger-tls>`. The keystore files have to
be added as {ref}`additional files <k8s-ex-sep-adding-files>`.

The configuration in the YAML file has to use the {ref}`coordinator nodes
<helm-sep-coordinator>` mechanism to add a properties file, inline the XML
configuration file that references the keystore files, and add additional
properties to `config.properties`:

```yaml
coordinator:
  etcFiles:
    properties:
      access-control-ranger.properties: |
        access-control.name=ranger
        ...
    other:
      ranger-policymgr-ssl.xml: |
        <configuration>
        ...
  additionalProperties: |
    access-control.config-files=/etc/starburst/access-control-ranger.properties
```

Add the SSL config file path to all catalogs in the {ref}`helm-sep-catalogs`
node using the default path and configured filename:

```yaml
catalogs:
  examplecatalog: |
    connector.name=...
    ...
    ranger.plugin-policy-ssl-config-file=/etc/starburst/ranger-policymgr-ssl.xml
```

#### TLS Encryption with Ranger

If your organization requires network traffic to be encrypted to the Ranger pod
rather than terminating TLS at the load balancer, you must configure the Ranger
`values.yaml` file as shown in the following example:

```yaml
expose:
  type: "loadBalancer"
  loadBalancer:
    name: "ranger-lb"
    ports:
      http:
        port: 6182

admin:
  port: 6182
  probes:
    readinessProbe:
      httpGet:
        path: /
        port: 6182
        scheme: HTTPS
      failureThreshold: 10
      periodSeconds: 10
    livenessProbe:
      httpGet:
        path: /
        port: 6182
        scheme: HTTPS
  env:
    RANGER__policymgr_http_enabled: false
    RANGER__policymgr_external_url: https://localhost:6182
    RANGER__policymgr_https_keystore_file: /tmp/rangercert/ranger-admin-keystore.jks
    RANGER__policymgr_https_keystore_keyalias: ranger-admin.docker.cluster
    RANGER__policymgr_https_keystore_password: password

additionalVolumes:
  - path: /tmp/rangercert
    volume:
      secret:
        secretName: "ranger-admin-keystore"
```

The following table explains the relevant YAML sections:

:::{list-table} Ranger YAML configuration sections for TLS
:widths: 35, 65
:header-rows: 1

* - YAML section
  - Purpose
* - `expose`
  - Configures Ranger to deploy its own load balancer, encrypting network
    traffic to the Ranger pod. Without this section, TLS encrypted network
    traffic intended for Ranger is decrypted before reaching Ranger.
* - `admin`
  - Declares the TLS configuration for Ranger's administration container
    that processes external traffic. The port must be configured to use the
    Ranger TLS port number, 6182.
* - `admin.probes`
  - Redefines the
    [readiness and liveness probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes)
    to use the https protocol, and to use port 6182.
* - `admin.env`
  - Defines several environment variables used by Ranger:

    - `RANGER__policymgr_http_enabled` - Must be set to `false` to
      enable TLS.
    - `RANGER__policymgr_external_url` - Defines the URL that the admin
      policy manager container will listen for traffic on, and must specify
      port 6182 as part of the URL.
    - `RANGER__policymgr_https_keystore_*` - These settings define the
      details of the Java keystore file that is to be used for TLS
      encryption, including the name of the file used when making the
      keystore, the keystore alias name, and the keystore's password.
* - `additionalVolumes`
  - Declares the k8s volume containing the Java keystore for Ranger to use.
    This must be deployed to the k8s cluster as a
    [secret](https://kubernetes.io/docs/tasks/configmap-secret/managing-secret-using-kubectl)
    whose name is given in the `secretName` setting. The
    `RANGER__policymgr_https_keystore_file` setting must match the path in
    this section. The key name given the file must match the value given when
    creating the secret.
:::

More information about working with Java keystore certificates is available in
the {doc}`JKS files </security/inspect-jks>` and
{doc}`PEM files </security/inspect-pem>` documentation.

## Deploy Ranger

When Ranger is configured, run the following command to deploy it. In this
example, the minimal values YAML file  with the registry credentials named
`registry-access.yaml` is used along with the `ranger-values.yaml`
containing the Ranger customizations:

```shell
$ helm upgrade ranger starburst/starburst-ranger \
    --install \
    --values ./registry-access.yaml \
    --values ./ranger-values.yaml
```

## Additional settings

(helm-ranger-startup)=

### Server start up configuration

You can create a startup shell script to customize how Ranger is started,
and pass additional arguments to it.

The script receives the container name as input parameter. Possible values are
`ranger-admin` and `ranger-usersync`. Additional arguments can be
configured with `extraArguments`.

:::{list-table} Startup script nodes
:widths: 30, 70
:header-rows: 1

* - Node name
  - Description
* - `initFile`
  - A shell script to run before Ranger is launched. The content of the file
    has to be an inline string in the YAML file. The script is started as
    `/bin/bash <<init_file>>`. When called, it is passed the single
    parameter value `ranger-admin` or `ranger-usersync` depending
    on the type of pod. The script needs to invoke the launcher script
    `/init/initFile.sh` for a successful start of Ranger.
* - `extraArguments`
  - List of extra arguments to be passed to the `initFile` script.
:::

(helm-ranger-registry)=

### Docker registry access

Same as {ref}`Docker image and registry section for the Helm chart
<helm-sep-image>` for SEP.

```yaml
registryCredentials:
  enabled: false
  registry:
  username:
  password:

imagePullSecrets:
 - name:
```

(helm-ranger-volumes)=

### Additional volumes

Additional volumes can be necessary for storing and accessing persisted files.
They can be defined in the `additionalVolumes` section. None are defined by
default:

```yaml
additionalVolumes: []
```

You can add one or more [volumes supported by k8s](https://kubernetes.io/docs/concepts/storage/volumes/#types-of-volumes), to
all nodes in the cluster.

If you specify `path` only, a directory named in `path` is created. When
mounting ConfigMap or Secret, files are created in this directory for each key.

This supports an optional `subPath` parameter which takes in an optional
key in the ConfigMap or Secret volume you create. If you specify `subPath`, a
specific key named `subPath` from ConfigMap or Secret is mounted as a file
with the  name provided by `path`.

The following example snippet shows both use cases:

```yaml
additionalVolumes:
  - path: /mnt/InContainer
    volume:
      emptyDir: {}
  - path: /tmp/config.txt
    subPath: config.txt
    volume:
      configMap:
        name: "configmap-in-volume"
```

(helm-ranger-datasources)=

### Data sources

Data sources are mounted inside the container as a file named
`/config/datasources.yaml`. The file is processed by an init script.

The following YAML configuration block defines a list of SEP data sources:

```yaml
datasources:
  - name: "fake-starburst-1"
    host: "starburst.fake-starburst-1-namespace"
    port: 8080
    username: "starburst_service1"
    password: "Password123"
  - name: "fake-starburst-2"
    host: "starburst.fake-starburst-2-namespace"
    port: 8080
    username: "starburst_service2"
    password: Password123
```

(helm-ranger-secrets)=

### Extra secrets

You can configure additional secrets that are mounted in the `/extra-secret/` path on each container.

```yaml
extraSecret:
  # Replace this with secret name that should be used from namespace you are deploying to
  name:
  # Optionally 'file' may be provided which will be deployed as secret with given 'name' in used namespace.
  file:
```

### Node assignment

You can configure your cluster to determine the node and pod to use for the
Ranger server:

```yaml
nodeSelector: {}
tolerations: []
affinity: {}
```

Our {ref}`SEP configuration documentation <helm-node-assignment>` contains
examples and resources to help you configure these YAML nodes.

### Annotations

You can add configuration to annotate the deployment and pod:

```yaml
deploymentAnnotations: {}
podAnnotations: {}
```

(helm-ranger-securitycontext)=

### Security context

You can optionally configure [security contexts](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-the-security-context-for-a-pod)
to define privilege and access control settings for the Ranger containers. You
can separately configure the security context for {ref}`Ranger admin
<helm-ranger-server>`, {ref}`usersync <helm-ranger-usersync>` and {ref}`database
<helm-ranger-database>` containers.

```yaml
securityContext:
```

If you do not want to set the serviceContext for the `default`
service account, you can restrict it by configuring the {ref}`service account
<helm-ranger-serviceaccount>` for the Ranger pod.

For a restricted environment like OpenShift, you may need to set the
`AUDIT_WRITE` capability for Ranger {ref}`usersync <helm-ranger-usersync>`:

```yaml
securityContext:
  capabilities:
    add:
      - AUDIT_WRITE
```

Additionally OpenShift clusters, need [anyuid](https://docs.openshift.com/container-platform/latest/authentication/managing-security-context-constraints.html)
and `privileged` security context constraints set for the {ref}`service
account <helm-ranger-serviceaccount>` used by Ranger. For example:

```shell
oc create serviceaccount <k8s-service-account>
oc adm policy add-scc-to-user anyuid system:serviceaccount:<k8s-namespace>:<k8s-service-account>
```

(helm-ranger-serviceaccount)=

### Service account

You can configure a [service account](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)
for the Ranger pod using:

```yaml
serviceAccountName:
```

(helm-ranger-environment-variables)=

### Environment variables

You can pass environment variables to the Ranger container using the same
mechanism used for the internal database:

```yaml
envFrom: []
env: []
```

Both are specified as a mapping sequences for example:

```yaml
envFrom:
  - secretRef:
      name: my-secret-with-vars
env:
  - name: MY_VARIABLE
    value: some-value
```

### Configure startup probe timeouts

You can define [startup probe timeouts](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-startup-probes)
for the Ranger Admin as well as the user synchronization container. This allows
the applications to connect to the database backend and complete starting the
application. The default setting of 300 seconds is the result of
`failureThreshold * periodSeconds` and the default values of 30 seconds and 10
seconds.

```yaml
admin:
  probes:
    startupProbe:
      failureThreshold: 30
      periodSeconds: 10

usersync:
  startupProbe:
    failureThreshold: 30
    periodSeconds: 10
```

The default is sufficient for a database backend running locally within the
cluster due to the low latency between the services. If the database is
operating externally you can change these timeout values to adjust for the
reduced network latency, and the resulting slower startup, to avoid startup
failures. For example, you can raise the value to ten minutes for both
containers:

```yaml
admin:
  probes:
    startupProbe:
      failureThreshold: 60
      periodSeconds: 10

usersync:
  startupProbe:
    failureThreshold: 60
    periodSeconds: 10
```

## Next steps

Review the following topics for next configuration steps:

- Read about {doc}`Starburst Enterprise and Ranger </security/ranger-overview>`
- Complete your {doc}`Ranger configuration <hms-configuration>`
