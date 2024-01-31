---
layout: default
title: Starburst License & Agreement
parent: SEP Deployment Mechanisms
nav_order: 1
---

# Starburst Enterprise license

As mentioned in the {doc}`overview </overview>`, a license for Starburst Enterprise platform (SEP)
includes support and enables numerous features.

To purchase a license or obtain a free trial, [contact us](https://www.starburstdata.com/contact/) or email us at
[hello@starburstdata.com](mailto:hello@starburstdata.com).

(license-usage)=

## License usage

After receiving a signed license file from Starburst Enterprise, it needs to be stored
on all SEP nodes in your clusters.

The license file needs to be named `starburstdata.license` and located within
the SEP installation directory as `etc/starburstdata.license`, or whatever
directory is configured as the etc directory with the launcher script. In case
of the {doc}`rpm archive </installation/rpm>` this path is
`/etc/starburst/starburstdata.license`.

Users of the CFT for SEP can provide the {ref}`license information as part of
the configuration <cft-sep-configuration>`.

Kubernetes users need to follow the {ref}`available instructions
<helm-sep-license>`, which are included in the {doc}`complete documentation for
Kubernetes usage with Helm </k8s>`.

(sep-elite-license)=

## Starburst Enterprise Elite license

The Starburst Enterprise Elite license unlocks premium features, such as
{doc}`/connector/starburst-stargate` and {doc}`/connector/starburst-warp-speed`,
in addition to the support and features included in the Starburst Enterprise license.

Managing and using the licenses is {ref}`identical <license-usage>`.
