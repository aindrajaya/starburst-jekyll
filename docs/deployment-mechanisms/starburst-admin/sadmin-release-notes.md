---
layout: default
title: Release Notes
parent: Deploy with Starburst Admin
grand_parent: SEP Deployment Mechanisms
nav_order: 6
permalink: /docs/deployment-mechanisms/starburst-admin/release-notes/
---

# Release notes

New releases of Starburst Admin are created whenever significant new
features or bug fixes are available.

## 1.8.0 (18 Jan 2024)

* Enhance the `push-config.yml` playbook to upload the
  `access-control.properties` file to coordinators and workers. This facilitates
  graceful shutdown configuration for SEP versions 414 to 426.
* Improve the `collect-logs.yml` playbook to ignore missing files while
  downloading logs.
* Resolve issues in the `check-status.yml` playbook for clusters installed using
  RPM with systemD.
* Remove support for {{oss}} and SEP versions 354-e and earlier.
* Implement autoconfigure and ulimit verification for the number of processes in
  installations based on the `launcher` script.

## 1.7.1 (15 Dec 2023)

* Add support for systemV in the `stop.yml` and `uninstall.yml` playbooks. This
  enables users to stop instances of SEP initiated by older versions of
  Starburst Admin.
* Improve playbook output by eliminating misleading warnings.

## 1.7.0 (5 Dec 2023)

* Add support for Starburst Enterprise 429-e and higher.
* Remove the Python dependency for SEP 427-e and higher.
* Replace systemV with systemD in start scripts.
* Remove `jvmkill` from starburst-admin (added in 1.6.1). This functionality was
  moved to the SEP code.

## 1.6.1 (12 Sep 2023)

* Use `jvmkill` for memory management.

## 1.6.0 (17 Oct 2022)

* Add automated memory configuration feature to `push-configs` playbook.

## 1.5.1 (17 Aug 2022)

* Adjust Java version check to support Java 17 for release version 390-e and
  higher.
* Update default JVM configuration for Java 17 for release versions 390-e
  and higher.

## 1.5.0 (9 Aug 2022)

* Add the ability to install Starburst Admin as a non-root user.

## 1.4.0 (27 Apr 2022)

* Added a new playbook `collect-jstacks.yml` that captures a
  {ref}`Java thread dump <pb-collect-jstacks>`
  on all SEP cluster nodes and saves each to a local file.

## 1.3.1 (25 Mar 2022)

* Allow user to specify a {ref}`custom vars.yml <pb-install>`
  file for playbooks to use. Allows users to specify the location of custom
  application configuration files.
* Remove the deprecated `discovery-server.enabled` setting from
  `config.properties`.

## 1.3.0 (17 Mar 2022)

* Remove any empty plugin directories left over from a previous installation in
  the {ref}`install playbook <pb-install>`.
* Remove any old installation directories left over  from a previous
  tarball-based installation in the {ref}`install playbook <pb-install>`.

## 1.2.0 (6 Aug 2021)

* Add condition to status printing to detect what preliminary task was skipped
* Add `OmitStackTraceInFastThrow` JVM flag
* Setup limits in `/etc/security/limits.d` for tarball installation

## 1.1.0 (11 Jun 2021)

* Fix `installation_group` name issue
* Remove `query.*` config property values in files to use automatic defaults
* Move user documentation to {doc}`docs.starburst.io
  </starburst-admin/get-started>`
* Remove developer documentation from package

## 1.0.0 (24 Mar 2021)

* Initial supported release with full feature set to run production systems
