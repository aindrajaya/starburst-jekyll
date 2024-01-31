---
layout: default
title: Kubernetes configuration examples
parent: Kubernetes cluster design for Starburst Enterprise
grand_parent: Deploy with Kubernetes
nav_order: 4
permalink: /docs/kubernetes/kubernetes-cluster-design-for-starburst-enterprise/sep-config-examples/
---

# Kubernetes configuration examples

You can find the default configuration and numerous practical examples for use
cases around {doc}`sep-configuration` in the following sections:

(helm-sep-license)=

## Adding the license file

Starburst provides customers a license file to unlock additional features of
SEP. The license file needs to be provided to SEP in the cluster:

1. Rename the file you received to `starburstdata.license`.

2. Create a k8s secret that contains the license file with a name of your
   choice in the cluster.

   > ```shell
   > kubectl create secret generic mylicense --from-file=starburstdata.license
   > ```

3. Configure the secret name as the Starburst platform license.

   > ```shell
   > starburstPlatformLicense: mylicense
   > ```

(k8s-ex-images-and-registry-credentials)=

## Images and repository registry credentials examples

(k8s-defaults-images-and-registry-credentials)=

### Defaults

**\<\<** {ref}`Return <helm-sep-image>` to section in k8s configuration
documentation.

The following are the image- and registry-related defaults. Do not place
unchanged values in customization files. Instead, follow best practices for
creating YAML files for customizing SEP:

```yaml
image:
  repository: "harbor.starburstdata.net/starburstdata/starburst-enterprise"
  tag: "434-e"
  pullPolicy: "IfNotPresent"

initImage:
  repository: "harbor.starburstdata.net/starburstdata/starburst-enterprise-init"
  tag: "434.0.0"
  pullPolicy: "IfNotPresent"

registryCredentials:
  enabled: false
  # Replace this with Docker Registry that you use
  registry:
  username:
  password:

imagePullSecrets:
```

(k8s-ex-registry-version-handling)=

### Controlling SEP releases

**\<\<** {ref}`Return <helm-sep-image>` to section in k8s configuration
documentation.

With normal usage you do not need to configure the image, since the chart
version updates automatically include the version update of the Docker images.
In rare cases, it can be useful to update the Docker image without changing the
overall chart version used. For example, you can choose to upgrade from
`3xx-e.1` to a newer  patch version `3xx-e.3` of SEP, which allows you to
keep the rest of the chart configuration unchanged:

```yaml
image:
  repository: "harbor.starburstdata.net/starburstdata"
  tag: "3xx-e.3"
  pullPolicy: "IfNotPresent"
```

(k8s-ex-private-registries)=

### Using private registries

**\<\<** Return to {ref}`Docker images <helm-sep-image>` or {ref}`Docker
registry access <helm-sep-registry>` section in k8s configuration
documentation.

In some organizations you need to {ref}`use private registries and repositories
instead of Starburst Harbor <k8s-private-registries>`. They are often hosted on
a private Harbor instance or in a repository manager. You can publish the Helm
charts and Docker containers to your private setup, or use a proxying setup.
Steps to set this up vary widely based on your tools, and require both Docker
and Helm expertise:

- Pull the Docker image from the Starburst Harbor registry with your credentials
- Tag the image as desired for your internal registry
- Push the image to your registry
- Download the Helm charts
- Publish the Helm charts to your Helm repository

You can use your private setup with the following steps:

- Add your private Helm chart repository using {ref}`these same
  steps <k8s-helm-repository>` as for adding the Starburst repository, but
  replacing the values with those of your private repository.
- Update your {ref}`registry credentials with
  details for your private Docker registry <k8s-ex-private-registries>`.

The following example overrides Docker registry to use your private registry:

```yaml
image:
  repository: "docker.example.com/thirdparty"
  tag: "434-e"
  pullPolicy: "IfNotPresent"
```

You also need to update your registry access configuration:

```yaml
registryCredentials:
  enabled: true
  registry: docker.example.com
  username: myusername
  password: mypassword
```

You can also use `imagePullSecrets:` with private registries instead of
`registryCredentials`.

If you changed the Docker image organization, name, or version tags, you also
need to override these details in your YAML configuration files. For example,
update {ref}`image and initImage for SEP <helm-sep-image>`. Similar steps are
necessary if you are using the HMS or Ranger charts.

(k8s-ex-imagepullsecrets)=

### Using `imagePullSecrets`

**\<\<** Return to {ref}`Docker images <helm-sep-image>` or {ref}`Docker
registry access <helm-sep-registry>` section in k8s configuration
documentation.

You can use `imagePullSecrets:` to authenticate with a Docker registry as an
alternative to using `registryCredentials:`. You can pass an array list of
Kubernetes secret names of type `kubernetes.io/dockerconfigjson` with the
following format:

```yaml
imagePullSecrets:
 - name: secret1
 - name: secret2
```

Detailed instructions for using private registries with pull secrets can be
found in the [Kubernetes documentation](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/).

(k8s-ex-internal-comms)=

## Internal communications configuration

(k8s-defaults-internal-comms)=

### Defaults

The following are the internal communications-related defaults in the
`values.yaml` file. Do not place unchanged values in customization files.
Instead, follow {ref}`best practices <k8s-customization-file-set>` for creating
YAML files for customizing SEP:

```yaml
sharedSecret:

environment:

internalTls: false

internal:
  ports:
    http:
      port: 8080
    https:
      port: 8443
```

(k8s-ex-internal-tls)=

### Using TLS for internal communication

**\<\<** {ref}`Return <helm-sep-internal-communication>` to section in k8s
configuration documentation.

You can optionally enable TLS for {doc}`internal communication
</security/internal-communication>`, if the cluster is deemed insecure, or TLS
is otherwise required and the performance overhead is acceptable.

(k8s-automatic-internal-tls)=

#### Configuring automatic internal TLS

Set `internalTls` to `true` to configure SEP to enable TLS for internal
communication. Certificates are automatically generated and used. You must
configure the `environment:` and `sharedSecret:` top level nodes, as well as
the `internal.ports.https.port:` to enable this feature.

```yaml
environment: production
sharedSecret: AN0Qhhw9PsZmEgEXAMPLEkIj3AJZ5/Mnyy5iRANDOMceM+SSV+APSTiSTRING
internalTls: true
internal:
  ports:
    http:
      port: 8080
    https:
      port: 8443
```

#### Manual TLS configuration

:::{warning}
We **very strongly suggest** that you use the {ref}`automatic internal TLS
<k8s-automatic-internal-tls>` as described in the preceding section. Manual
TLS configuration is **deprecated functionality**. Using and configuring TLS
for internal communication is very complex, requiring you to implement a
certificate manager and managing certificates within the cluster.
:::

All cluster nodes must have a fully qualified domain name (FQDN) that matches
the Kubernetes naming scheme. When `node.internal-address-source` is set to
`FQDN` (in both the the `coordinator.additionalProperties:` and
`worker.additionalProperties:` nodes), the chart manages the
`node.internal-address` property automatically, and the SAN field in TLS certs
must match.

The TLS certificates used must have `starburst`,
`coordinator.<namespace>.svc` and `*.worker.<namespace>.svc` in the Subject
Alternative Name (SAN) field. Replace `<namespace>` with a real value.

```yaml
coordinator:
  additionalProperties: |
    node.internal-address-source=FQDN

worker:
  additionalProperties: |
    node.internal-address-source=FQDN
```

(k8s-ex-nonstd-https)=

### Non-standard HTTPS port numbers

**\<\<** {ref}`Return <helm-sep-internal-communication>` to section in k8s
configuration documentation.

If a non-standard port (other than 8443) is used for HTTPS, the same value must
be set for both `internal.ports.https.port` in the YAML file and the
`http-server.https.port` property in `config.properties`. This is best
achieved by adding the setting to the additionalProperties for coordinator and
workers as detailed in {ref}`k8s-ex-internal-tls`.

```yaml
internal:
  ports:
    https:
      port: 8440
coordinator:
  additionalProperties: |
    http-server.https.port=8440
worker:
  additionalProperties: |
    http-server.https.port=8440
```

(k8s-ex-external-comms)=

## External communications configuration

(k8s-defaults-external-comms)=

### Defaults

The following are the external communications-related defaults in the
`values.yaml` file. Do not place unchanged values in customization files.
Instead, follow {ref}`best practices <k8s-customization-file-set>` for creating
YAML files for customizing SEP:

```yaml
expose:
  type: "clusterIp"
  clusterIp:
    name: "starburst"
    ports:
      http:
        port: 8080
  nodePort:
    name: "starburst"
    ports:
      http:
        port: 8080
        nodePort: 30080
    extraLabels: {}
  loadBalancer:
    name: "starburst"
    IP: ""
    ports:
      http:
        port: 8080
    annotations: {}
    sourceRanges: []
  ingress:
    ingressName: "coordinator-ingress"
    serviceName: "starburst"
    servicePort: 8080
    ingressClassName:
    tls:
      enabled: true
      secretName:
    host:
    path: "/"
    annotations: {}
```

(k8s-ex-expose-clusterip)=

### `clusterIp` type

**\<\<** {ref}`Return <helm-sep-expose>` to section in k8s configuration
documentation.

```yaml
expose:
  type: "clusterIp"
  clusterIp:
    name: "starburst"
    ports:
      http:
        port: 8080
```

(k8s-ex-expose-nodeport)=

### `nodePort` type

**\<\<** {ref}`Return <helm-sep-expose>` to section in k8s configuration
documentation.

```yaml
expose:
  type: "nodePort"
  nodePort:
    name: "starburst"
    ports:
      http:
        port: 8080
        nodePort: 30080
```

(k8s-ex-expose-lb)=

### `loadBalancer` type

**\<\<** {ref}`Return <helm-sep-expose>` to section in k8s configuration
documentation.

```yaml
expose:
  type: "loadBalancer"
  loadBalancer:
    name: "starburst"
    IP: ""
    ports:
      http:
        port: 8080
    annotations: {}
    sourceRanges: []
```

(k8s-ex-expose-ingress)=

### Basic `ingress` type

**\<\<** {ref}`Return <helm-sep-expose>` to section in k8s configuration
documentation.

```yaml
expose:
  type: "ingress"
  ingress:
    serviceName: "starburst"
    servicePort: 8080
    tls:
      enabled: true
      secretName:
    host:
    path: "/"
    annotations: {}
```

(k8s-ex-expose-nginxcert)=

### `ingress` with nginx and cert-manager

**\<\<** {ref}`Return <helm-sep-expose>` to section in k8s configuration
documentation.

[nginx](https://nginx.org/en/) is a powerful HTTP and proxy server, commonly
used as load balancer. You can combine using it with cert-manager backed by
[Let’s Encrypt](https://letsencrypt.org/).

As a first step you need to deploy an HTTPS ingress controller for your cluster.
You can follow a [tutorial from the cert-manager documentation](https://cert-manager.io/docs/tutorials/acme/nginx-ingress/).

With the setup done, and an A record in your DNS zone ready, you can expose the
{{sep_ui}}:

```yaml
expose:
  type: "ingress"
  ingress:
    serviceName: "starburst"
    servicePort: 8080
    tls:
      enabled: true
      secretName: "tls-secret-starburst"
    host: ""
    path: "/(.*)"
    annotations:
      kubernetes.io/ingress.class: "nginx"
      cert-manager.io/issuer: "letsencrypt-staging"
```

The `secretName` is used by the cert-manager to store the generated
certificate, and can be any value.

The `annotations` section uses the `nginx` default value for the single
ingress controller installation. It assumes certificate issuer with the name
`letsencrypt-staging` is used, and needs to exist.

The Ranger user interface can be exposed in exactly the same way:

```yaml
expose:
  type: "ingress"
  ingress:
    tls:
      enabled: true
      secretName: "tls-secret-ranger"
    host: ""
    path: "/(.*)"
    annotations:
      kubernetes.io/ingress.class: "nginx"
      cert-manager.io/issuer: "letsencrypt-staging"
```

(k8s-coordinator-yaml)=

## Coordinator configuration

(k8s-defaults-coordinator)=

### Defaults

**\<\<** {ref}`Return <helm-sep-coordinator>` to section in k8s configuration
documentation.

The following are the coordinator-related defaults in the `values.yaml` file.
Do not place unchanged values in customization files. Instead, follow {ref}`best
practices <k8s-customization-file-set>` for creating YAML files for customizing
SEP:

```yaml
coordinator:
  etcFiles:
    jvm.config: |
      -server
      --add-opens=java.base/sun.nio.ch=ALL-UNNAMED
      --add-opens=java.base/java.nio=ALL-UNNAMED
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
    properties:
      config.properties: |
        coordinator=true
        node-scheduler.include-coordinator=false
        http-server.http.port=8080
        discovery-server.enabled=true
        discovery.uri=http://localhost:8080
      node.properties: |
        node.environment={% raw %}{{include "starburst.environment" .}}{% endraw %}
        node.data-dir=/data/starburst
        plugin.dir=/usr/lib/starburst/plugin
        node.server-log-file=/var/log/starburst/server.log
        node.launcher-log-file=/var/log/starburst/launcher.log
      log.properties: |
        # Enable verbose logging from Starburst Enterprise
        #io.trino=DEBUG
        #com.starburstdata.presto=DEBUG
      password-authenticator.properties: |
        password-authenticator.name=file
        file.password-file=/usr/lib/starburst/etc/auth/password.db
      access-control.properties:
    other: {}
  resources:
    memory: "60Gi"
    cpu: 16
  nodeMemoryHeadroom: "2Gi"
  heapSizePercentage: 90
  heapHeadroomPercentage: 30
  additionalProperties: ""

  envFrom: []
  nodeSelector: {}
  affinity: {}
  tolerations: []
  deploymentAnnotations: {}
  podAnnotations: {}
  priorityClassName:
```

(k8s-ex-jvm-config)=

### JVM configuration

**\<\<** {ref}`Return <k8s-etcfiles>` to section in k8s configuration
documentation.

The JVM configuration is automatically included and includes the appropriate
memory settings based on the configured resources. In rare cases you might need
to add or modify some parameters. In this case you need to include the full
default JVM configuration and the modified values. The following example only
modified G1HeapRegionSize and ReservedCodeCacheSize, but all values are required
to be included.

```yaml
coordinator:
  etcFiles:
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

The same settings need to be applied to the worker configuration.

(k8s-ex-etcfiles-other)=

### Adding other files in `etc`

**\<\<** {ref}`Return <k8s-etcfiles>` to section in k8s configuration
documentation.

You can add any random other required configuration file in the `etc` folder,
by adding a node with the desired filename in the `other` section, and using
YAML multi-line sections to define the content. The following example adds the
two files `etc/resource-groups.json` and `etc/kafka/tpch.customer.json`:

```yaml
coordinator:
  etcFiles:
    other:
      resource-groups.json: |
          {
          <<json_here>
          }
      kafka/tpch.customer.json: |
          {
          <<json_here>
          }
```

(k8s-ex-etcfiles-log)=

### Logging

Writing log output to files is not enabled in k8s by default. The logs are
written to `stdout` and `stderr` streams. Kubernetes-native logging tools
should be used for log analysis and troubleshooting.

In rare cases you can temporarily enable logging by setting the `log.path`
property in `additionalProperties`, and configure {ref}`log-levels` in
`log.properties` to your desired settings. For more information, see
{doc}`/admin/properties-logging`.

The package structure of the source code determines the package to use for
logging specific plugins or functionality. The following example shows how to
debug the plugin codebase of the Iceberg connector.

```yaml
coordinator:
  additionalProperties: |
    log.path=/var/log/starburst/server.log
  etcFiles:
    properties:
      log.properties: |
        io.trino.plugin.iceberg=DEBUG
```

(k8s-ex-coord-resources)=

### CPU and memory allocation

**\<\<** {ref}`Return <helm-sep-coordinator>` to section in k8s configuration
documentation.

You can configure the desired CPU and memory allocation to use for coordinator
and worker pods. The parameters affect the requested pod size via k8s [resource
management settings](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/).
The settings are also used to determine the memory settings for the JVM running
SEP.

```yaml
coordinator:
  resources:
    memory: "256Gi"
    cpu: 32
  nodeMemoryHeadroom: "4Gi"
```

By default, SEP use the same values for resource requests and limits. You can
specify different settings for requests and limits using the [same syntax that
k8s uses for pods](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#example-1).

For example, you can make sure CPU limits are not specified, in order to bypass
a known issue with [k8s throttling](https://medium.com/omio-engineering/cpu-limits-and-aggressive-throttling-in-kubernetes-c5b20bd8a718):

```yaml
coordinator:
  resources:
    requests:
      memory: "256Gi"
      cpu: 32
    limits:
      memory: "256Gi"
```

The `query.max-memory` property is set to `1PB`. This setting overrides the
{doc}`low default value </admin/properties-resource-management>`.

(k8s-ex-additional-properties)=

### Adding properties to `config.properties`

**\<\<** {ref}`Return <helm-sep-coordinator>` to section in k8s configuration
documentation.

You can set additional properties to change to the {ref}`default configuration
file <config-properties>` and override the default values. An example usage is
to set client and query time-out values:

```yaml
additionalProperties: |
  query.client.timeout=5m
  query.min-expire-age=30m
```

(helm-sep-coordinator-envfrom)=

### Using environment variables for secrets

**\<\<** {ref}`Return <helm-sep-coordinator>` to section in k8s configuration
documentation.

```yaml
envFrom:
  - secretRef:
    name: <<secret_name>>
```

To use secrets for sensitive credential information to use in a catalog
properties file:

1\. Create a secret holding variables. You can statically create secrets using
`base64` encoded values of your configuration. Make sure your secret key,
which is used to define the environment variable name, follows this regex
pattern `[a-zA-Z][a-zA-Z0-9_]*` - only alphanumerics and underscore allowed.
Convention is to use all caps and underscores such as `PSQL_USERNAME`.

```shell
$ echo -n user | base64
$ echo -n pass | base64
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: variables-secret
type: Opaque
data:
  PSQL_USERNAME: <base64_encoded_user>
  PSQL_PASSWORD: <base64_encoded_pass>
```

2. Add the secret reference in `envFrom` for both `coordinator` and
   `worker` to make it accessible on all nodes:

```yaml
envFrom:
  - secretRef:
      name: variables-secret
```

3\. Reference variables in properties files using built-in placeholder pattern as
enabled by the {doc}`secrets support </security/secrets>`.

```yaml
catalogs:
  postgresql: |
    connector.name=postgresql
    connection-url=jdbc:postgresql://postgresql:5432/postgres
    connection-password=${ENV:PSQL_PASSWORD}
    connection-user=${ENV:PSQL_USERNAME}
```

(k8s-ex-insights)=

### Enabling the SEP backend service and {{insights}}

**\<\<** {ref}`Return <helm-sep-coordinator>` to section in k8s configuration
documentation.

:::{note}
The `insights.jdbc.url`, `insights.jdbc.user` and
`insights.jdbc.password` configuration properties are part of the backend
service and are **required**. Together, they enable  a number of features
besides {{insights}}. Read more about it in our {doc}`backend service topic
</admin/backend-service>`.
:::

{doc}`Starburst Insights </insights/index>` provides a visual overview of
important metrics about your cluster as well as a query editor feature to write
and run SQL queries. We strongly suggest reading about the options and
capabilities of this tool.

Enable and configure {{insights}} in the
`additionalProperties` section of the `coordinator` configuration:

```yaml
coordinator:
  additionalProperties: |
    insights.persistence-enabled=true
    insights.metrics-persistence-enabled=true
    insights.jdbc.url=jdbc:postgresql://<database hostname>:5432/starburstenterprise
    insights.jdbc.user=<user>
    insights.jdbc.password=<password>
    insights.authorized-users=<superuser>
```

This example activates persistence for the query history feature.

Our {doc}`Insights documentation </insights/index>` contains a complete
description of all {{insights}} configuration properties.

(k8s-ex-access-control)=

### Enabling access control

**\<\<** {ref}`Return <helm-sep-coordinator>` to section in k8s configuration
documentation.

SEP has several options for implementing {doc}`access control
<../security>`. You can add any desired content of the properties
file inside the YAML file. For example, you can use the read-only
{doc}`/security/built-in-system-access-control`:

```yaml
coordinator:
  etcFiles:
    properties:
      access-control.properties: |
        access-control.name=read-only
```

To implement Ranger, use the {doc}`Apache Ranger Helm chart
<./ranger-configuration>`.

Other access control choices, such as {ref}`file-based user access control
<helm-sep-user-db>` require configurations in one or more nodes in the SEP
Helm chart.

SEP also supports Privacera. Please refer to your [Privacera user
documentation](https://docs.privacera.com/latest/platform/) to learn how to
connect that service to your k8s cluster.

Also see: file-based user authentication {ref}`example <k8s-ex-sep-user-db>`.

(k8s-worker-yaml)=

## Worker configuration

(k8s-defaults-worker)=

### Defaults

**\<\<** {ref}`Return <helm-sep-workers>` to section in k8s configuration
documentation.

The following are the worker-related defaults in the `values.yaml` file. Do
not place unchanged values in customization files. Instead, follow {ref}`best
practices <k8s-customization-file-set>` for creating YAML files for customizing
SEP:

```yaml
worker:
  etcFiles:
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
    properties:
      config.properties: |
        coordinator=false
        http-server.http.port=8080
        discovery.uri=http://{% raw %}{{include "starburst.service.name" .}}{% endraw %}
      node.properties: |
        node.environment={% raw %}{{include "starburst.environment" .}}{% endraw %}
        node.data-dir=/data/starburst
        plugin.dir=/usr/lib/starburst/plugin
        node.server-log-file=/var/log/starburst/server.log
        node.launcher-log-file=/var/log/starburst/launcher.log
      log.properties: |
        # Enable verbose logging from Starburst Enterprise
        #io.trino=DEBUG
        #com.starburstdata.presto=DEBUG
    other: {}
  replicas: 2
  autoscaling:
    enabled: false
    minReplicas: 1
    maxReplicas: 100
    targetCPUUtilizationPercentage: 80
  deploymentTerminationGracePeriodSeconds: 300 # 5 minutes
  starburstWorkerShutdownGracePeriodSeconds: 120 # 2 minutes
  resources:
    memory: "100Gi"
    cpu: 16
  nodeMemoryHeadroom: "2Gi"
  heapSizePercentage: 90
  heapHeadroomPercentage: 30
  additionalProperties: ""
  envFrom: []
  nodeSelector: {}
  affinity: {}
  tolerations: []
  deploymentAnnotations: {}
  podAnnotations: {}
  priorityClassName:
```

## Startup script

(k8s-defaults-initfile)=

### Defaults

The following are the startup script-related defaults in the `values.yaml`
file. Do not place unchanged values in customization files. Instead, follow
{ref}`best practices <k8s-customization-file-set>` for creating YAML files for
customizing SEP:

```yaml
initFile: ""
extraArguments: []
```

(k8s-ex-startup-script)=

### Using initFile

**\<\<** {ref}`Return <helm-sep-nodes>` to section in k8s configuration
documentation.

```yaml
initFile:
extraArguments:
extraSecret:
  name:
  file:
```

The following example shows how you can use `initFile` to run a custom init
script on the coordinator and workers:

```yaml
initFile: |
  #!/bin/bash
  echo "Custom init for $1 $2"
  exec /usr/lib/starburst/bin/run-starburst
extraArguments:
  - TEST_ARG
```

% todo: need something less vague for <<starburst_logs>>

Output on the coordinator:

```text
Custom init for coordinator TEST_ARG
<<starburst_logs>>
```

Output on a worker:

```text
Custom init for worker TEST_ARG
<<starburst_logs>>
```

Use `initFile:` to retrieve and load drivers and other large binaries with
`curl` at startup:

```yaml
initFile: |
  #!/bin/bash
  echo "Custom init for $1 $2"
  curl https://gdaadmins.blob.core.windows.net/prestoteradatadriver/terajdbc4.jar -o /usr/lib/presto/plugin/teradata/terajdbc4.jar
  exec /usr/lib/starburst/bin/run-starburst
extraArguments:
  - TEST_ARG
```

(k8s-ex-security-considerations)=

## Security considerations

(k8s-defaults-security-considerations)=

### Defaults

The following are the security-related defaults in the `values.yaml` file. Do
not place unchanged values in customization files. Instead, follow {ref}`best
practices <k8s-customization-file-set>` for creating YAML files for customizing
SEP:

```yaml
externalSecrets:
  enabled: false
  type: goDaddy
  secretPrefix: external/
  goDaddy:
    backendType: secretsManager

userDatabase:
  enabled: false
  users:
    - username: admin
      password: thepassword

securityContext: {}

extraSecret:
  name:
  file:
```

(k8s-ex-secretref)=

### External secret reference

**\<\<** {ref}`Return <helm-sep-secretRef>` to section in k8s configuration
documentation.

To configure SEP to work with LDAP as an external secret reference, first
create a k8s secret holding the file:

```shell
$ kubectl create secret generic ldap-ca --from-file=ca.crt
```

When the file is created, you can configure the secret reference usage for the
above configuration as:

```yaml
coordinator:
  etcFiles:
    properties:
      password-authenticator.properties: |
        ldap.url=ldaps://ldap-server:636
        ldap.user-bind-pattern=uid=${USER},OU=America,DC=corp,DC=example,DC=com
        ldap.ssl-trust-certificate=secretRef:ldap-ca:ca.crt
```

This mounts the secret named `ldap-ca` in the path `/mnt/secretsRef/ldap-ca`
and replaces `secretRef:ldap-ca` occurrences into the absolute path, resulting
in the following configuration property setting:

```text
ldap.ssl-trust-certificate=/mnt/secretRef/ldap-ca/ca.crt
```

:::{note}
Specific secret values, such as passwords, can be passed into properties files
{ref}`using the envFrom parameters <helm-sep-coordinator-envfrom>` available
for `coordinator` and `worker`.
:::

(k8s-ex-external-secrets)=

### Defining external secrets

**\<\<** {ref}`Return <k8s-external-secrets>` to section in k8s configuration
documentation.

You can automatically mount external secrets, for example from the AWS Secrets
Manager, using the `secretRef` or `secretEnv` notation.

```yaml
externalSecrets:
  enabled: true # disabled by default
  type: goDaddy
  secretPrefix: <<secret_name_prefix>>
  goDaddy:
    backendType: <<string>>
```

An example of this configuration:

1. Create AWS Secrets Manager secret:

```shell
$ aws secretsmanager create-secret --name external.starburst-http-server-port --secret-string 8888
```

2. Reference it from your configuration section in `config.properties`:

```yaml
coordinator:
  etcFiles:
      config.properties: |
      http-server.http.port=secretEnv:external/starburst-http-server-port
```

3. Configure the external secrets:

```yaml
externalSecrets:
  enabled: true
  type: goDaddy
  secretPrefix: external/
  goDaddy:
    backendType: secretsManager
```

This creates the following external secret manifest:

```yaml
apiVersion: kubernetes-client.io/v1
kind: ExternalSecret
metadata:
  name: external.starburst-http-server-port
spec:
  backendType: secretsManager
  data:
    - key: external/starburst-http-server-port
      name: EXTERNAL_STARBURST_HTTP_SERVER_PORT
```

Additionally, the external secrets provider fetches secrets from AWS and creates
a k8s secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: external.starburst-http-server-port
type: Opaque
data:
  EXTERNAL_STARBURST_HTTP_SERVER_PORT: 8888
```

The k8s secret is now bound to the container as the
`EXTERNAL_STARBURST_HTTP_SERVER_PORT` environment variable. SEP
`config.properties` is resolved to:

```shell
http-server.http.port=${ENV:EXTERNAL_STARBURST_HTTP_SERVER_PORT}
```

If you have a secret with multiple values, such as a JSON-formatted secret, you
can reference the secret values independently.

For example, you may have a secret named `external-starburst-creds-mysql` that
is structured like this in the AWS Secrets Manager:

```shell
{
  "MYSQL_USER": "user",
  "MYSQL_PASSWORD": "password"
}
```

The `MYSQL_USER` and `MYSQL_PASSWORD` keys can be referenced in the
`values.yaml` file:

```yaml
externalSecrets:
  enabled: true
  type: goDaddy
  secretPrefix: external/
  goDaddy:
    backendType: secretsManager

catalogs:
   mysqldb: |-
    connector.name=mysql
    connection-url=jdbc:mysql://<<dns>>:3306
    connection-user=secretEnv:external-starburst-creds-mysql:MYSQL_USER
    connection-password=secretEnv:external-starburst-creds-mysql:MYSQL_PASSWORD
```

(k8s-ex-sep-user-db)=

### File-based authentication

**\<\<** {ref}`Return <helm-sep-user-db>` to section in k8s configuration
documentation.

Using the htpasswd-generated file:

```yaml
userDatabase:
  enabled: true
  name: password.db
  users:
    - username: admin
      password: thepassword
```

Using an externally-created user database:

% todo: need full example here

```yaml
userDatabase:
  enabled: false
```

(k8s-ex-rbac-enabled-clusters)=

### RBAC-enabled clusters

**\<\<** {ref}`Return <k8s-rbac-clusters>` to section in k8s requirements
documentation.

In the following example, a user `steve` is configured to work with
SEP in a namespace called `dev-sandbox`:

1. Create *RoleBinding* `steve-edit` to bind the edit *ClusterRole* to the
   user in the specific namespace:

> ```shell
> kubectl create rolebinding steve-edit \
>   --clusterrole edit \
>   --user steve \
>   --namespace dev-sandbox
> ```

2. If `externalSecrets:` are in use, then a role with the following
   permissions must be additionally bound to the user:

> ```yaml
> kind: ClusterRole
> apiVersion: rbac.authorization.k8s.io/v1
> metadata:
>   name: external-secrets-edit
> rules:
> - apiGroups:
>   - kubernetes-client.io
>   resources:
>   - externalsecrets
>   - externalsecrets/status
>   verbs:
>     - create
>     - get
>     - list
>     - watch
>     - update
>     - patch
>     - delete
> ```
>
> ```shell
> kubectl create rolebinding steve-external-secrets-edit \
>   --clusterrole external-secrets-edit \
>   --user steve \
>   --namespace dev-sandbox
> ```

The Starburst Enterprise Helm chart does not provide functionality to use a
service account for deployments. None of the containers deployed to the cluster
needs access to the Kubernetes API.

(k8s-ex-performance-considerations)=

## Performance considerations

(k8s-defaults-performance-query)=

### Concurrent query defaults

The following is the `query:` default in the `values.yaml` file. Do not
place unchanged values in customization files. Instead, follow {ref}`best
practices <k8s-customization-file-set>` for creating YAML files for customizing
SEP:

```yaml
query:
  maxConcurrentQueries: 3
```

(k8s-ex-sep-concurrent-queries)=

### Concurrent query increase example

**\<\<** {ref}`Return <helm-sep-concurrent-queries>` to section in k8s
configuration documentation.

```yaml
query:
  maxConcurrentQueries: 5
```

(k8s-defaults-performance-spilling)=

### Spilling defaults

The following are the spilling-related defaults in the `values.yaml` file. Do
not place unchanged values in customization files. Instead, follow {ref}`best
practices <k8s-customization-file-set>` for creating YAML files for customizing
SEP:

```yaml
spilling:
  enabled: false
  volume:
    emptyDir: {}
```

(k8s-ex-sep-spilling)=

### Spilling example

**\<\<** {ref}`Return <helm-sep-spilling>` to section in k8s configuration
documentation.

% todo: need an example with emptyDir braces having content. Super vague as is.

```yaml
spilling:
  enabled: true:
  volume:
    emptyDir: {}
```

(k8s-defaults-performance-cache)=

### Hive connector storage caching defaults

The following are the storage caching-related defaults in the `values.yaml`
file. Do not place unchanged values in customization files. Instead, follow
{ref}`best practices <k8s-customization-file-set>` for creating YAML files for
customizing SEP:

```yaml
cache:
  enabled: false
  diskUsagePercentage: 80
  ttl: "7d"
  volume:
    emptyDir: {}
```

(k8s-ex-sep-hive-caching)=

### Hive connector storage caching example

**\<\<** {ref}`Return <helm-sep-hive-caching>` to section in k8s configuration
documentation.

```yaml
cache:
  enabled: true
  diskUsagePercentage: 75
  ttl: "5d"
```

## Catalogs

(k8s-defaults-catalogs)=

### Default

The following is the default catalog entry in the `values.yaml` file. Do not
place unchanged values in customization files. Instead, follow {ref}`best
practices <k8s-customization-file-set>` for creating YAML files for customizing
SEP:

```yaml
catalogs:
  tpch: |-
    connector.name=tpch
    tpch.splits-per-node=4
```

(k8s-ex-sep-catalogs)=

### Catalog examples

**\<\<** {ref}`Return <helm-sep-catalogs>` to section in k8s configuration
documentation.

The following snippet adds the `tpcds-testdata` catalog. It uses
the {doc}`/connector/tpcds` and only specifies the connector name.

```yaml
catalogs:
  tpcds-testdata: |
  connector.name=tpcds
```

Multiple catalogs are configured one after the other:

```yaml
catalogs:
  tpch-testdata: |
    connector.name=tpch
  tpcds-testdata: |
    connector.name=tpcds
  tmpmemory: |
    connector.name=memory
  metrics: |
    connector.name=jmx
  devnull: |
    connector.name=blackhole
  datalake: |
    connector.name=hive
    hive.metastore.uri=thrift://hive:9083
  s3: |
    connector.name=hive
    hive.metastore=glue
```

The name of each catalog is defined by the chosen name for the node within
`catalogs`. The above examples results in catalog names such as
`tpch-testdata`, `tpcds-testdata`, `tmpmemory`, `metrics` and others.
These names are visible for CLI and other tool users with `SHOW CATALOGS` and
potentially in the user interface.

Each catalog properties file can use the configuration options supported by the
{doc}`connector <../connector/starburst-connectors>` designated by the
configured `connector.name`.

(k8s-ex-teradata-direct-connector)=

### Teradata Direct connector

**\<\<** {ref}`Return <helm-sep-catalogs>` to section in k8s configuration
documentation.

The {ref}`Starburst Teradata Direct connector
<starburst-teradata-direct-connector>` is supported for Kubernetes deployments
in AWS EKS and in Azure AKS. Follow {doc}`the detailed instructions to configure
the necessary networking and components <./sep-config-teradata-direct>`.

:::{warning}
The configuration to use the Starburst Teradata Direct connector on Kubernetes
is complex. You need significant Kubernetes and networking knowledge. Contact
our {{support}} team for assistance.
:::

## Additional volumes

(k8s-defaults-additionalvolumes)=

### Default

The following is the `additionalVolumes:` default in the `values.yaml`
file. Do not place unchanged values in customization files:

```yaml
additionalVolumes: []
```

(k8s-ex-sep-volumes)=

### Adding volumes examples

**\<\<** {ref}`Return <helm-sep-volumes>` to section in k8s configuration
documentation.

```yaml
additionalVolumes:
  - path: /mnt/InContainer
    volume:
      emptyDir: {}
  - path: /var/lib/starburst/cache1
    volume:
      hostPath:
        path: /media/nv1/starburst-cache
  - path: /var/lib/starburst/cache2
    volume:
      hostPath:
        path: /media/nv2/starburst-cache
```

(k8s-ex-sep-adding-files)=

### Adding files examples

**\<\<** {ref}`Return <helm-sep-adding-files>` to section in k8s configuration
documentation.

% todo: subpath is not mentioned in the values file, only hostpath. This is
% either a bug or in need of clarification; its not covered or differentiated in
% the linked sep-configuration section

As an example, if you want to copy a file to an already existing location like
`/usr/lib/starburst/plugin` you can mount the file to a Kubernetes volume like
a [ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/),
and add the file as a `subPath` to the `path`:

```yaml
additionalVolumes:
- path: /usr/lib/starburst/plugin/x.jar
  subPath: x.jar
  volume:
    configMap:
      name: "configmap-in-volume"
```

In this case, the key named `x.jar` from the ConfigMap is mounted as that file
in the location provided in `path`.

Large binaries, such as drivers, are added at cluster start time in the
`initFile:` {ref}`top level node <k8s-ex-startup-script>`.

(k8s-ex-sep-prometheus)=

## Prometheus

(k8s-defaults-prometheus)=

### Default

**\<\<** {ref}`Return <helm-sep-prometheus>` to section in k8s configuration
documentation.

The following are the `prometheus:` defaults in the `values.yaml` file. Do
not place unchanged values in customization files. Instead, follow {ref}`best
practices <k8s-customization-file-set>` for creating YAML files for customizing
SEP:

```yaml
prometheus:
  enabled: true
  agent:
    version: "0.20.0"
    port: 8081
    config: "/etc/starburst/telemetry/prometheus.yaml"
  rules:
    - pattern: trino.execution<name=QueryManager><>(running_queries|queued_queries)
      name: $1
      attrNameSnakeCase: true
      type: GAUGE
    - pattern: 'trino.execution<name=QueryManager><>FailedQueries\.TotalCount'
      name: 'failed_queries'
      type: COUNTER
```

(k8s-ex-kubernetes-configs)=

### Kubernetes management and monitoring

(k8s-defaults-k8s-monitoring)=

### Default

The following are the Kubernetes-related defaults in the `values.yaml` file.
Do not place unchanged values in customization files. Instead, follow {ref}`best
practices <k8s-customization-file-set>` for creating YAML files for customizing
SEP:

```yaml
readinessProbe:
  exec:
    command:
      - /bin/sh
      - -c
      - curl --max-time 5 -s http://localhost:8080/v1/info | grep \"starting\":false
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 30

livenessProbe:
  exec:
    command:
      - /bin/sh
      - -c
      - curl --max-time 5 -s http://localhost:8080/v1/info | grep \"starting\":false
  initialDelaySeconds: 300
  periodSeconds: 300
  timeoutSeconds: 30
  failureThreshold: 1

commonLabels: {}
```
