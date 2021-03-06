---

copyright:
  years: 2014, 2020
lastupdated: "2020-02-03"

keywords: kubernetes, iks, vpc

subcollection: containers

---

{:codeblock: .codeblock}
{:deprecated: .deprecated}
{:download: .download}
{:external: target="_blank" .external}
{:faq: data-hd-content-type='faq'}
{:gif: data-image-type='gif'}
{:help: data-hd-content-type='help'}
{:important: .important}
{:new_window: target="_blank"}
{:note: .note}
{:pre: .pre}
{:preview: .preview}
{:screen: .screen}
{:shortdesc: .shortdesc}
{:support: data-reuse='support'}
{:table: .aria-labeledby="caption"}
{:tip: .tip}
{:troubleshoot: data-hd-content-type='troubleshoot'}
{:tsCauses: .tsCauses}
{:tsResolve: .tsResolve}
{:tsSymptoms: .tsSymptoms}


# Troubleshooting VPC clusters
{: #vpc_troubleshoot}

Review some known issues or common error messages that you might encounter when you use VPC clusters.
{: shortdesc}

## Cannot connect to an app via load balancer
{: #vpc_ts_lb}

{: tsSymptoms}
You publicly exposed your app by creating a Kubernetes `LoadBalancer` service in your VPC cluster. When you try to connect to your app by using the hostname that is assigned to the Kubernetes `LoadBalancer`, the connection fails or times out.

When you run `kubectl describe svc <kubernetes_lb_service_name>`, you see a warning message similar to one of the following in the **Events** section:
```
The VPC load balancer that routes requests to this Kubernetes `LoadBalancer` service is offline.
```
{: screen}
```
The VPC load balancer that routes requests to this Kubernetes `LoadBalancer` service was deleted from your VPC.
```
{: screen}

{: tsCauses}
When you create a Kubernetes `LoadBalancer` service in your cluster, a VPC load balancer is automatically created in your VPC. The VPC load balancer routes requests only to the app that the Kubernetes `LoadBalancer` service exposes. Requests cannot be routed to your app in the following situations:
* The VPC load balancer is offline, such as due to load balancer provisioning errors or VSI connection errors.
* The VPC load balancer is deleted through the VPC console or the CLI.
* The VPC load balancer's DNS entry is still registering.

{: tsResolve}
Verify that the VPC load balancer for the Kubernetes `LoadBalancer` service exists. In the output, look for the VPC load balancer that is formatted `kube-<cluster_ID>-<kubernetes_lb_service_UID>`. You can get the Kubernetes `LoadBalancer` service UID by running `kubectl get svc <service_name> -o yaml`.
  ```
  ibmcloud is load-balancers
  ```
  {: pre}

* If the VPC load balancer is not listed, it was deleted through the VPC console or the CLI. To re-create the VPC load balancer for your Kubernetes `LoadBalancer` service, restart the Kubernetes master by running `ibmcloud ks cluster master refresh --cluster <cluster_name_or_id>`. <p class="tip">If you want to remove the load-balancing setup for an app in your VPC cluster, delete the Kubernetes `LoadBalancer` service by running `kubectl delete svc <kubernetes_lb_service_name>`. The VPC load balancer that is associated with the Kubernetes `LoadBalancer` service is automatically deleted from your VPC.</p>
* If the VPC load balancer is listed, its DNS entry might still be registering. When a VPC load balancer is created, the hostname is registered through a public DNS. In some cases, it can take several minutes for this DNS entry to be replicated to the specific DNS that your client is using. You can either wait for the hostname to be registered in your DNS, or access the VPC load balancer directly by using one of its IP addresses. To find the VPC load balancer IP addresses, look for the **Public IP** column in the output of `ibmcloud is load-balancers`. If after several minutes you cannot reach the load balancer, it might be offline due to provisioning or connection issues. [Open an {{site.data.keyword.cloud_notm}} support case](https://cloud.ibm.com/unifiedsupport/cases/add). For the type, select **Technical**. For the category, select **Network** in the VPC section. In the description, include your cluster ID and the VPC load balancer ID.

<br />


## Kubernetes `LoadBalancer` service fails because no IPs are available
{: #vpc_no_lb}

{: tsSymptoms}
You publicly exposed your app by creating a Kubernetes `LoadBalancer` service in your VPC cluster. When you run `kubectl describe svc <kubernetes_lb_service_name>`, you see a warning message in the **Events** section similar to one of the following:
```
The subnet with ID(s) '<subnet_id>' has insufficient available ipv4 addresses.
```
{: screen}

{: tsCauses}
When you create a Kubernetes `LoadBalancer` service in your cluster, a VPC load balancer is automatically created in your VPC. The VPC load balancer puts a floating IP address for your Kubernetes `LoadBalancer` service behind a hostname that you can access your app through.

In VPC clusters, both worker nodes and services are assigned IP addresses from the same subnets. Traffic routing is enabled between subnets, so when all IP addresses in a subnet for a zone are used by worker nodes or services, you can still create new worker nodes or services in that zone because they use IP addresses from subnets in other zones. However, if all IP addresses on all subnets are in use, a new Kubernetes `LoadBalancer` service cannot be successfully provisioned.

{: tsResolve}
After you create a VPC subnet, you cannot resize it or change its IP range. Instead, you must create a larger VPC subnet in one or more zones where you have worker nodes. Then you create a new worker pool using the larger subnets.

1. [Create a new VPC subnet](https://cloud.ibm.com/vpc/provision/network){: external} in the same VPC and in one or more zones where your cluster has worker nodes. Make sure that you create a subnet that can support both the number of worker nodes and services that you plan to create in your cluster. The default CIDR size of each VPC subnet is `/24`, which can support up to 253 worker nodes and services. To check your cluster's VPC and zones, run `ibmcloud ks cluster get --cluster <cluster_name_or_ID>`.

2. Create a new worker pool in your cluster.
     ```
     ibmcloud ks worker-pool create vpc-classic --name <name> --cluster <cluster_name_or_ID> --flavor <flavor> --size-per-zone <number_of_worker_nodes> --label <key>=<value>
     ```
     {: pre}

3. Using the ID for the larger subnets that you created in step 1, add the zones to the worker pool. Repeat the following command for each zone and subnet.
    ```
    ibmcloud ks zone add vpc-classic --zone <zone> --subnet-id <subnet_id> --cluster <cluster_name_or_ID> --worker-pool <worker_pool_name>
    ```
    {: pre}

4. After a few minutes, verify that your `LoadBalancer` service is successfully provisioned onto one of the new subnets. If the service is provisioned successfully, no `Warning` or `Error` events are displayed.
  ```
  kubectl describe svc <kubernetes_lb_service_name>
  ```
  {: pre}

<br />


## Cannot connect to an app via Ingress
{: #vpc_ts_alb}

{: tsSymptoms}
You publicly exposed your app by creating an Ingress resource for your app in your cluster. When you tried to connect to your app by using the subdomain of the Ingress application load balancer (ALB), the connection failed or timed out.

{: tsCauses}
When you create a VPC cluster, one public and one private VPC load balancer is automatically created outside of your cluster in your VPC. The VPC load balancer routes requests to the apps that the ALBs expose. Requests cannot be routed to your app in the following situations:
* The VPC load balancer is offline, such as due to load balancer provisioning errors or VSI connection errors.
* The VPC load balancer is deleted through the VPC console or the CLI.
* The VPC load balancer's DNS entry is still registering.

{: tsResolve}
Verify that the VPC load balancer for your ALBs exists. In the output, look for the VPC load balancer that has the same **Host Name** as your public or private ALBs. You can see the hostnames for your public and private ALBs by running `ibmcloud ks alb ls --cluster <cluster_name_or_ID>` and looking for the **Load Balancer Hostname** field.
  ```
  ibmcloud is load-balancers
  ```
  {: pre}

* If the VPC load balancer is not listed, it was deleted through the VPC console or the CLI. To re-create the VPC load balancer for your ALBs, disable all of the public or private ALBs that are assigned that VPC load balancer's hostname by running `ibmcloud ks alb configure vpc-classic --alb-id <ALB_ID> --disable` for each ALB. Then, re-enable those ALBs by running `ibmcloud ks alb configure vpc-classic --alb-id <ALB_ID> --enable` for each ALB. A new VPC load balancer for the ALBs takes a few minutes to provision in your VPC. You cannot access your app until the VPC load balancer for your ALBs is fully provisioned.
* If the VPC load balancer is listed, its DNS entry might still be registering. When a VPC load balancer is created, the hostname is registered through a public DNS. In some cases, it can take several minutes for this DNS entry to be replicated to the specific DNS that your client is using. You can either wait for the hostname to be registered in your DNS, or access the VPC load balancer directly by using one of its IP addresses. To find the VPC load balancer IP addresses, look for the **Public IP** column in the output of `ibmcloud is load-balancers`. If after several minutes you cannot reach the load balancer, it might be offline due to provisioning or connection issues. [Open an {{site.data.keyword.cloud_notm}} support case](https://cloud.ibm.com/unifiedsupport/cases/add). For the type, select **Technical**. For the category, select **Network** in the VPC section. In the description, include your cluster ID and the VPC load balancer ID.

## Feedback, questions, and support
{: #getting_help}

Still having issues with your cluster? Review different ways to get help and support for your {{site.data.keyword.containerlong_notm}} clusters. For any questions or feedback, post in Slack.
{: shortdesc}

**General ways to resolve issues**<br>
1. Keep your cluster environment up to date.
   * Check monthly for available security and operating system patches to [update your worker nodes](/docs/containers?topic=containers-update#worker_node).
   * [Update your cluster](/docs/containers?topic=containers-update#master) to the latest default version for [{{site.data.keyword.containershort}}](/docs/containers?topic=containers-cs_versions).
2. Make sure that your command line tools are up to date.
   * In the terminal, you are notified when updates to the `ibmcloud` CLI and plug-ins are available. Be sure to keep your CLI up-to-date so that you can use all available commands and flags.
   * Make sure that [your `kubectl` CLI](/docs/containers?topic=containers-cs_cli_install#kubectl) client matches the same Kubernetes version as your cluster server. [Kubernetes does not support](https://kubernetes.io/docs/setup/release/version-skew-policy/){: external} `kubectl` client versions that are 2 or more versions apart from the server version (n +/- 2).
<br>

**Reviewing issues and status**<br>
1. To see whether {{site.data.keyword.cloud_notm}} is available, [check the {{site.data.keyword.cloud_notm}} status page](https://cloud.ibm.com/status?selected=status){: external}.
2. Filter for the **Kubernetes Service** component.
<br>

**Feedback and questions**<br>
1. Post in the {{site.data.keyword.containershort}} Slack.
   * If you are an external user, post in the [#general](https://ibm-container-service.slack.com/archives/C4G6362ER){: external} channel.
   * If you are an IBMer, use the [#armada-users](https://ibm-argonauts.slack.com/archives/C4S4NUCB1) channel.<p class="tip">If you do not use an IBMid for your {{site.data.keyword.cloud_notm}} account, [request an invitation](https://cloud.ibm.com/kubernetes/slack){:external} to this Slack.</p>
2. Review forums such as {{site.data.keyword.containershort}} help, Stack Overflow, and IBM Developer to see whether other users ran into the same issue. When you use the forums to ask a question, tag your question so that it is seen by the {{site.data.keyword.cloud_notm}} development teams.
   * If you have technical questions about developing or deploying clusters or apps with {{site.data.keyword.containerlong_notm}}, post your question on [Stack Overflow](https://stackoverflow.com/questions/tagged/ibm-cloud+containers){: external} and tag your question with `ibm-cloud`, `containers`, and `openshift`.
   * For questions about the service and getting started instructions, use the [IBM Developer Answers](https://developer.ibm.com/answers/topics/containers/?smartspace=bluemix){: external} forum. Include the `ibm-cloud` and `containers` tags.
   * See [Getting help](/docs/get-support?topic=get-support-getting-customer-support#using-avatar) for more details about using the forums.
<br>

**Getting help**<br>
1.  Contact IBM Support by opening a case. To learn about opening an IBM support case, or about support levels and case severities, see [Contacting support](/docs/get-support?topic=get-support-getting-customer-support).
2.  In your support case, for **Category**, select **Containers**.
3.  For the **Offering**, select your {{site.data.keyword.containershort}} cluster.<p class="tip">When you report an issue, include your cluster ID. To get your cluster ID, run `ibmcloud ks cluster ls`. You can also use the [{{site.data.keyword.containerlong_notm}} Diagnostics and Debug Tool](/docs/containers?topic=containers-cs_troubleshoot#debug_utility) to gather and export pertinent information from your cluster to share with IBM Support.</p>


