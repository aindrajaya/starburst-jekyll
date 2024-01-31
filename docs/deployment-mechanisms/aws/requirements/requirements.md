---
layout: default
title: Starburst Enterprise with CFT requirements
parent: Deploy with ECT AWS
grand_parent: SEP Deployment Mechanisms
nav_order: 1
permalink: /docs/aws/requirements
---

# Starburst Enterprise with CFT requirements

The following sections detail networking and security requirements, best
practices, and troubleshooting tips for the Starburst Enterprise platform (SEP) CloudFormation template
in AWS.

(aws-ssh)=

## SSH keys

Amazon EC2 uses public–key cryptography to encrypt and decrypt login
information, so to access your SEP cluster you need to set up a private/public
key pair. To log into your instance where SEP is installed, do the
following:

1. Create a key pair.
2. Specify the name of the key pair when you launch the instance or invoke the
   CloudFormation template.
3. Provide the private key when you connect to the instance. This enables you to
   securely access your instance using the private key pair instead of a password.

:::{note}
Amazon EC2 stores only the public key and you are responsible for storing the
private key. Take action to safeguard your keys, as anyone who possesses your
private key can decrypt your login information.
:::

You can find more information about [key pair usage with EC2 in the AWS
documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html).

(aws-vpc)=

## VPC and VPC subnets

When using SEP’s CloudFormation template, you must specify which existing VPC
and subnets to deploy to.

Read more about {doc}`best practices when selecting VPCs for SEP
</ecosystems/aws/eks/networking>`. You can
also find more information about [VPCs and subnets in the AWS documentation](https://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Subnets.html).

(aws-security-groups)=

## Security groups

It’s recommended that ports 8080 and 8088 are accessible in order to access the
{{sep_ui}}, submit queries from outside the cluster, and access Apache Superset.
Additionally, it’s recommended that port 22 is accessible for SSH access.

You can find more information about [security groups in the AWS documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-network-security.html).

(aws-security-prereq)=

## IAM requirements

[AWS Identity and Access Management (IAM)](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html)
is a web service that helps you securely control access to your AWS resources.
The following sections detail the IAM roles used by the SEP CloudFormation
template.

(aws-security-prereq-nodes)=

### IAM role permissions for cluster nodes

Below is a JSON segment for the IAM role automatically created by the SEP
CloudFormation template. The role is used by the cluster nodes that are created
on the stack. The permissions set below are a minimal subset of IAM permissions:

```json
{
    "Statement": [
        {
            "Action": [
                "autoscaling:CompleteLifecycleAction",
                "autoscaling:RecordLifecycleActionHeartbeat",
                "cloudformation:SignalResource",
                "ec2:DescribeInstances",
                "glue:BatchGetPartition",
                "glue:BatchCreatePartition",
                "glue:CreateDatabase",
                "glue:CreateTable",
                "glue:DeleteDatabase",
                "glue:DeletePartition",
                "glue:DeleteTable",
                "glue:GetDatabase",
                "glue:GetDatabases",
                "glue:GetPartition",
                "glue:GetPartitions",
                "glue:GetTable",
                "glue:GetTables",
                "glue:UpdateTable",
                "glue:UpdatePartition",
                "s3:GetObject",
                "s3:ListBucket",
                "s3:PutObject",
                "sqs:ChangeMessageVisibility",
                "sqs:DeleteMessage",
                "sqs:GetQueueUrl",
                "sqs:ReceiveMessage",
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents",
                "logs:PutRetentionPolicy",
                "logs:DescribeLogGroups",
                "logs:DescribeLogStreams",
                "cloudwatch:PutMetricData",
                "secretsmanager:GetSecretValue",
                "secretsmanager:ListSecrets"
            ],
            "Effect": "Allow",
            "Resource": [
                "*"
            ]
        }
    ],
    "Version": "2012-10-17"
}
```

In the above:

- *AutoScaling*, *EC2* and *SQS* permissions are needed for {ref}`graceful
  scaledown <cft-graceful-scaledown>` to work, when autoscaling kicks in, or
  reshaping the cluster via the stack modification.
- *CloudWatch Logs* permissions are needed to enable logging to AWS CloudWatch
  Logs.
- *CloudFormation* permission is needed to report startup progress back to
  CloudFormation.
- *CloudWatch* permissions are needed to run the coordinator high availability
  features.
- *Glue* access is needed to leverage AWS Glue data catalogs.
- *S3* permissions are needed for the {doc}`Hive connector
  </connector/starburst-hive>` to actually access (read/write) the data on S3.

If a user wants to have a more precise control over the permissions, and for
example limit S3 access to a specific bucket, then a custom IAM Instance Profile
can be provided and passed to the CloudFormation template during stack creation
(`IamInstanceProfile` field in the stack creation form). The custom profile is
then used instead and the IAM Role and Instance Profile are not created.

:::{warning}
You must give all other permissions regarding *AutoScaling*, *SQS*,
*EC2*, *CloudFormation*, *CloudWatch*, and *Glue* to this custom
role. Otherwise, the cluster either fails to start or does not function as
documented.
:::

Additionally the role needs to have a trust relationship to EC2 established, so
that EC2 can use this role on the user's behalf. Below is a minimal trust
relationship document:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

(aws-security-prereq-cft)=

### IAM role permissions for CloudFormation

The user who launches the CloudFormation template needs to have specific
permissions to issue all necessary actions in AWS during stack creation. In
particular, before proceeding make sure your IAM user/role has a policy with
permissions to manage all of the following:

```json
{
    "Statement": [
        {
            "Action": [
                "apigateway:DELETE",
                "apigateway:GET",
                "apigateway:POST",
                "apigateway:PUT",
                "apigateway:UpdateRestApiPolicy",
                "autoscaling:CreateAutoScalingGroup",
                "autoscaling:CreateLaunchTemplate",
                "autoscaling:DeleteAutoScalingGroup",
                "autoscaling:DeleteLaunchTemplate",
                "autoscaling:DeleteLifecycleHook",
                "autoscaling:DescribeAutoScalingGroups",
                "autoscaling:DescribeAutoScalingInstances",
                "autoscaling:DescribeLaunchTemplate",
                "autoscaling:DescribeLifecycleHooks",
                "autoscaling:DescribeScalingActivities",
                "autoscaling:DisableMetricsCollection",
                "autoscaling:EnableMetricsCollection",
                "autoscaling:PutLifecycleHook",
                "autoscaling:UpdateAutoScalingGroup",
                "cloudformation:CreateStack",
                "cloudformation:DeleteStack",
                "cloudformation:DescribeStackEvents",
                "cloudformation:DescribeStacks",
                "cloudwatch:DeleteAlarms",
                "cloudwatch:DeleteDashboards",
                "cloudwatch:DescribeAlarms",
                "cloudwatch:GetDashboard",
                "cloudwatch:ListDashboards",
                "cloudwatch:PutDashboard",
                "cloudwatch:PutMetricAlarm",
                "ec2:AuthorizeSecurityGroupEgress",
                "ec2:AuthorizeSecurityGroupIngress",
                "ec2:CreatePlacementGroup",
                "ec2:CreateNetworkInterface",
                "ec2:CreateSecurityGroup",
                "ec2:CreateTags",
                "ec2:CreateVpcEndpoint",
                "ec2:DeletePlacementGroup",
                "ec2:DeleteNetworkInterface",
                "ec2:DeleteSecurityGroup",
                "ec2:DeleteVpcEndpoints",
                "ec2:DescribeInstances",
                "ec2:DescribeKeyPairs",
                "ec2:DescribePlacementGroups",
                "ec2:DescribeNetworkInterfaces",
                "ec2:DescribeSecurityGroups",
                "ec2:DescribeSubnets",
                "ec2:DescribeVpcEndpoints",
                "ec2:DescribeVpcs",
                "ec2:ModifyNetworkInterfaceAttribute",
                "ec2:RevokeSecurityGroupEgress",
                "ec2:RevokeSecurityGroupIngress",
                "ec2:RunInstances",
                "ec2:TerminateInstances",
                "events:DeleteRule",
                "events:DescribeRule",
                "events:PutRule",
                "events:PutTargets",
                "events:RemoveTargets",
                "iam:AddRoleToInstanceProfile",
                "iam:AttachRolePolicy",
                "iam:CreateInstanceProfile",
                "iam:CreateRole",
                "iam:DeleteInstanceProfile",
                "iam:DeleteRole",
                "iam:DeleteRolePolicy",
                "iam:DetachRolePolicy",
                "iam:GetInstanceProfile",
                "iam:GetRole",
                "iam:PassRole",
                "iam:PutRolePolicy",
                "iam:RemoveRoleFromInstanceProfile",
                "lambda:AddPermission",
                "lambda:CreateFunction",
                "lambda:DeleteFunction",
                "lambda:GetFunction",
                "lambda:GetFunctionConfiguration",
                "lambda:InvokeFunction",
                "lambda:PutFunctionConcurrency",
                "lambda:RemovePermission",
                "sqs:CreateQueue",
                "sqs:DeleteQueue",
                "sqs:GetQueueAttributes",
                "sqs:TagQueue"
                "secretsmanager:CreateSecret",
                "secretsmanager:DeleteResourcePolicy",
                "secretsmanager:DeleteSecret",
                "secretsmanager:DescribeSecret",
                "secretsmanager:PutResourcePolicy",
                "secretsmanager:PutSecretValue",
                "secretsmanager:TagResource",
                "secretsmanager:UpdateSecret",
            ],
            "Effect": "Allow",
            "Resource": [
                "*"
            ]
        }
    ],
    "Version": "2012-10-17"
}
```

The role needs to have a trust relationship to CloudFormation established, so
that CloudFormation can use this role on the user's behalf. Below is a minimal
trust relationship document:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudformation.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

(aws-instance-profiles)=

## Instance profiles

When using SEP’s CloudFormation template, specify the instance profile
to associate with the EC2 instances. For example, this may be useful to provide
the EC2 instance the appropriate privileges to access data in S3.

You can find more information about [instance profiles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-ec2_instance-profiles.html).

## AWS CFT Troubleshooting tips

The following sections describe procedures and other information to troubleshoot
your SEP clusters.

### Accessing an SEP cluster node to troubleshoot

You can connect to your SEP cluster via SSH. To do so, you must first
obtain the IP address of the coordinator and the file name of your `.pem`
file.

To locate the coordinator IP address:

- **CloudFormation Console:** Navigate to the CloudFormation Console under
  “Management Tools” within the Services menu.
- **Outputs:** Select your "Stack Name" and click the associated tab labeled
  “Outputs”.
- **SSH Access:** Find and copy the “StarburstSSH” key's value.

After you have gathered that information, type the ssh command in a terminal
window to establish a connection to the coordinator:

```shell
ssh ec2-user@coordinator-ip -i ~/.ssh/mykeypair.pem
```

Using the example above, replace `coordinator-ip` with the IP address of your
coordinator and replace `mykeypair.pem` with the location and file name of
your `.pem` file.

### Custom SEP configuration

When using a {ref}`custom-configuration`, the CloudFormation update stack may not
update the Starburst Enterprise platform (SEP) configuration after you've made changes to content in
your configuration zip. This usually occurs if the updated configuration zip
file name is the same as previously used. Try renaming the package and rerunning
the CloudFormation update stack. We recommend including a version name within
the file name to avoid any complications when updating your configurations.

When using a {ref}`custom-configuration`, the CloudFormation update stack may
fail. This is often because the configuration zip file has an invalid directory
structure. Be sure to double check the content and structure of your
configuration package. When troubleshooting, try adding incremental updates to
the configuration package until it fails.

Read more about {ref}`HA considerations <cft-high-availability>` for
non-standard issues related to custom security configurations and coordinator
HA.

### Placement groups capacity

When using the cluster placement group type, you can encounter insufficient
capacity errors. To avoid such a setback, consider the following suggestions:

#### Single launch request

It is recommended that you launch the number of instances that you need in the
placement group in a single launch request, and that you use the same instance
type for all instances in the placement group. If you try to add more instances
to the placement group later, or if you try to launch more than one instance
type in the placement group, you increase your chances of getting an
insufficient capacity error.

#### Start. Stop. Relaunch.

If you receive a capacity error when launching an instance in a placement group
that already has running instances, stop and start all of the instances in the
placement group, and try the launch again. Restarting the instances may migrate
them to hardware that has the capacity for all the requested instances.

Refer to the AWS documentation for more information on [placement groups](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/placement-groups.html#concepts-placement-group).
