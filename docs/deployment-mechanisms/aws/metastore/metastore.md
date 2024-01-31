---
layout: default
title: Configuring the Hive Metastore Service with CFT
parent: Deploy with ECT AWS
grand_parent: SEP Deployment Mechanisms
nav_order: 4
permalink: /docs/aws/metastore
---

# Configuring the Hive Metastore Service with CFT

The SEP CloudFormation Template (CFT) supports several different methods for
configuring a Hive Metastore Service (HMS), Hive metastore, and optionally a
backing database in the cluster. The following SEP {doc}`connectors
<../connector/starburst-connectors>` and features require a HMS:

- {doc}`/connector/starburst-hive`
- {doc}`/connector/starburst-delta-lake`
- {doc}`/connector/iceberg`
- and others

You can set the `MetastoreType` to the following values.

:::{list-table} Supported Hive metastore types
:widths: 45, 55
:header-rows: 1

* - `MetastoreType`
  - Description
* - `None`
  - *(Default)* {ref}`No Hive metastore <aws-metastore-none>` is configured.
* - `Standalone (ephemeral)`
  - Use a {ref}`standalone Hive metastore instance in EC2
    <aws-metastore-standalone>`
* - `AWS Glue Data Catalog`
  - Use the {ref}`AWS Glue data catalog<aws-metastore-glue>` as an HMS
* - `External MySQL RDBMS`
  - Use an external {ref}`MySQL RBDMS<aws-metastore-mysql>` as a Hive
    metastore
* - `External PostgreSQL RDBMS`
  - Use an external {ref}`PostgresSQL RBDMS<aws-metastore-psql>` as a Hive
    metastore
* - `External Hive Metastore Service`
  - Connect to an existing, {ref}`external Hive Metastore Service
    <aws-metastore-hive>`.
:::

(aws-metastore-none)=

## No metastore

The default configuration sets to `MetastoreType` to `None` and does not configure a Hive metastore.

(aws-metastore-standalone)=

## Standalone (ephemeral) metastore

By setting `MetastoreType` to `Standalone (ephemeral)`, a separate EC2
instance is created by the CFT. It contains both the Hive metastore and its
underlying RDBMS.

Note that information stored in such a metastore only lives as long as the SEP
cluster. Because of that such configuration should be avoided on production
system, while it is the best option to test {{oss}} and the Hive connector.

(aws-metastore-glue)=

## AWS Glue data catalog

By setting `MetastoreType` to `AWS Glue Data Catalog`, the Hive catalog
uses the AWS Glue Data Catalog as its metastore service.

(aws-metastore-mysql)=

## External MySQL RDBMS

By setting `MetastoreType` to `External MySQL RDBMS`, a separate EC2
instance is created by the CFT. It runs a Hive Metastore Service that leverages
an external **MySQL** RDBMS as its underlying storage.

This new instance requires network access to the external MySQL system. You must
configure your networking and security groups accordingly. We recommend using
AWS RDS, but you can use your own MySQL instance.

This configuration requires the following properties to be set:

- `ExternalMetastoreHost` the host address of the **MySQL** service.
- `ExternalMetastorePort` the port number of **MySQL** service. If `0` is
  set then `3306` (default **MySQL** port) is used.
- `ExternalRdbmsMetastoreUserName` the **MySQL** user name
- `ExternalRdbmsMetastorePassword` the **MySQL** user password
- `ExternalRdbmsMetastoreDatabaseName` the **MySQL** database name that
  is used for storing Hive Metastore data.

RDBMS does not require any schema initialization other than database creation.
It is well suited for **MySQL** provisioned with AWS RDS service.

(aws-metastore-psql)=

## External PostgreSQL RDBMS

By setting the `MetastoreType` to `External PostgreSQL RDBMS`, a separate
EC2 instance is created by CFT which runs a Hive Metastore Service. It leverages
an external **PostgreSQL** RDBMS as its underlying storage.

This new instance needs network access to the external PostgreSQL system. You
must configure your networking and security groups accordingly. We recommend
using AWS RDS, but you can use your own PostgreSQL instance.

This configuration requires the following properties to be set:

- `ExternalMetastoreHost` the host address of the **PostgreSQL** service.
- `ExternalMetastorePort` the port number of **PostgreSQL** service. If
  `0` is set then `5432` (default **PostgreSQL** port) is used.
- `ExternalRdbmsMetastoreUserName` the **PostgreSQL** user name
- `ExternalRdbmsMetastorePassword` the **PostgreSQL** user password
- `ExternalRdbmsMetastoreDatabaseName` the **PostgreSQL** database name
  that is used for storing Hive Metastore data.

RDBMS does not require any schema initialization other than database creation.
It is well suited for **PostgreSQL** provisioned with AWS RDS service.

(aws-metastore-hive)=

## External Hive metastore service

By setting `MetastoreType` to `External Hive Metastore Service`, the Hive
connector uses an existing Hive Metastore Service.

This configuration requires the below properties to be set:

- `ExternalMetastoreHost` the host address of the Hive Metastore Service.
- `ExternalMetastorePort` the port number of the Hive Metastore Service. If
  `0` is set then `9083` (default **Hive Metastore Service** port) is used.
