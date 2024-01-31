# Teradata Direct connector in Kubernetes

**\<\<** {ref}`Return <k8s-ex-teradata-direct-connector>` to section in k8s
configuration documentation.

The {ref}`Starburst Teradata Direct connector
<starburst-teradata-direct-connector>` is supported for Kubernetes deployments
in AWS EKS and in Azure AKS.

:::{warning}
The configuration to use the {{sb}} Teradata Direct connector on Kubernetes is
complex. You need significant Kubernetes and networking knowledge. Contact our
{{support}} team for assistance.
:::

It relies on the Teradata cluster nodes, and the deployed table operator, to be
able to address the SEP cluster nodes. Each worker node and the coordinator
need an assigned IP address that is exposed outside the Kubernetes SEP
cluster and is reachable from Teradata. This need translates into the following technical requirements:

- A CNI (Container Network Interface) must be configured using [this interface](https://github.com/aws/amazon-vpc-cni-k8s) for AWS with SNAT disabled so
  that it is performed outside of Kubernetes, allowing pods to be accessed from
  outside the cluster by their assigned IP addresses.
- Teradata must be deployed into the same Virtual Network via a VPC/VNet peering
  connection or VPN.
- An adequate number of node instance network interfaces and IP address are
  available, taking autoscaling needs into account.

Thoroughly read the {ref}`Teradata Direct connector documentation
<starburst-teradata-direct-connector>` before you proceed to setup your
environment, and create and verify catalogs.

## AWS EKS setup

AWS provides the `vpc-cni` addon to automate configuraiton of your EKS
cluster. Follow the [EKS Addons instructions](https://eksctl.io/usage/addons/)
for guidance.

After the EKS cluster is deployed, you must update its default security group
with an inbound rule allowing the Teradata host to get back into the EKS
cluster.

## Azure AKS setup

Ensure that [Azure VNet CNI](https://docs.microsoft.com/en-us/azure/aks/configure-azure-cni) is applied to
the cluster from your preferred management interface, such as Azure Portal or
Terraform.

Only if you are using Terraform, the following is an example section:

```text
resource "azurerm_kubernetes_cluster" "main" {
  network_profile {
    network_plugin    = "azure"
    network_mode      = "transparent"
  }
}
```

## Catalog setup and verification

1. Install the {ref}`native table operator
   <native-table-operator>` of the Teradata Direct connector on Teradata.
2. Add a catalog to your `sep-prod-catalogs.yaml` file, following this
   example:

```yaml
catalogs:
  myteradatadirect: |-
    connector.name=teradata-direct
    connection-url=jdbc:teradata://<routable_ip_address>
    connection-user=test
    connection-password=test
    teradata-direct.table-operator.name=test.starburst_table_operator
    teradata-direct.http.port=9000
    teradata-direct.table-operator.logging-level=DEBUG
```

3. Verify all pods have IP assigned addresses in a shared network, and the
   following command does not return an error indicating that no IP addresses
   are available:

   ```shell
   $ kubectl describe pod <my-deployment-xxxxxxxxxx-xxxxx> -n <my-namespace>

   Failed to create pod sandbox: rpc error: code = Unknown desc = failed to
   set up sandbox container "<LongHash> network for pod
   "<my-deployment-59f5f68b58-c89wx>": networkPlugin cni failed to set up
   pod "<my-deployment-59f5f68b58-c89wx_my-namespace>" network: add cmd:
   failed to assign an IP address to container
   ```

4. Run the following Teradata Direct connector query through your preferred
   client:

   ```sql
   select count(*) from myteradatadirect.sys_calendar.calendar;
   ```

5. Verify that the receivers are responding. The connector should return the
   result of the above query immediately. A brief check of the
   `/tmp/starburstdata_table_operator*.log` file should not display any
   `sleep:` entries, indicating that receivers could not respond or were not
   found:

   ```text
   2021-04-15T11:18:32 DEBUG src/main/cpp/util.cpp:211 sleep: 244
   2021-04-15T11:18:32 DEBUG src/main/cpp/util.cpp:211 sleep: 500
   2021-04-15T11:18:32 DEBUG src/main/cpp/util.cpp:211 sleep: 1000
   ```
