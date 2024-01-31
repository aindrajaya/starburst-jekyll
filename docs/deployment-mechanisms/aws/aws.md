---
layout: default
title: Deploy with ECT AWS
parent: SEP Deployment Mechanisms
nav_order: 2
permalink: /docs/aws
has_children: true
---


# Deploy with CFT on AWS

Starburst Enterprise platform (SEP) can be run on Amazon Web Services (AWS) as provider
for all aspects of the required infrastructure. This includes using [AWS
CloudFormation](https://docs.aws.amazon.com/cloudformation/index.html) for
provisioning, [Amazon Simple Storage Service (S3)](https://docs.aws.amazon.com/s3/) for storage, Amazon Machine Images (AMI) and
[Amazon Elastic Compute Cloud (EC2)](https://docs.aws.amazon.com/ec2/) for
computes, Amazon Glue as metadata catalog, and others.

This chapter covers all the above.

Usage of Amazon Elastic Kubernetes Service (EKS) is covered in the {doc}`general
Kubernetes section </k8s>`.

```{toctree}
:maxdepth: 1

aws/requirements
aws/installation
aws/configuration
aws/metastore
aws/cloudwatch
aws/release-notes
```
