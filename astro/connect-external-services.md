---
sidebar_label: "Connect to external services"
title: "Connect Astro to external data services"
id: connect-external-services
description: Learn how to connect your Astro data plane to different types of external data services.
---

Before you can run pipelines on Astro with real data, you first need to make your data services accessible to your data plane and the Deployments running within it. This guide explains how to securely connect Astro to external data services using the following methods:

- Public endpoints
- Virtual Private Cloud (VPC)
- Virtual Network (VNet) peering
- Amazon Web Services (AWS) Transit Gateway
- AWS PrivateLink
- AWS IAM roles
- Workload Identity (_Google Cloud Platform (GCP) only_)

If you need to connect to a type of data service that requires a connectivity method that is not documented here, reach out to [Astronomer support](https://support.astronomer.io).

## Public endpoints

The fastest and easiest way to connect Astro to an external data service is by using the data service's publicly accessible endpoints.

These endpoints can be configured by:

- Setting [environment variables](environment-variables.md) on Astro with your endpoint information.
- Creating an [Airflow connection](https://airflow.apache.org/docs/apache-airflow/stable/howto/connection.html) with your endpoint information.

Public connection traffic moves directly between your data plane and the external data service's API endpoint. This means that the data in this traffic never reaches the control plane, which is managed by Astronomer.

### Allowlist Astro IPs

Some data services, including Snowflake and Databricks, require that you identify and allowlist any IP addresses responsible for outbound traffic before you can access their services from an external network. This provides an additional layer of security.

On Astro, each cluster has two unique external IP addresses that persist throughout the lifetime of the cluster. These IP addresses are assigned for network address translation, which means that they are responsible for all outbound traffic from your Astro cluster to the internet.

To retrieve the IP addresses of an Astro cluster, contact [Astronomer support](https://support.astronomer.io) and request them. Then, allowlist these IP addresses in your data service provider. If you have more than one cluster, you will need to allowlist each cluster individually.

## VPC/VNet peering

Each cluster on Astro runs in a dedicated Virtual Private Network (VPC/VNet). To set up private connectivity between an Astro VPC/VNet and another VPC/VNet, you can set up a VPC peering connection. VPC peering ensures private and secure connectivity, reduced network transit costs, and simplified network layouts.

To create a VPC peering connection between an Astro VPC and a target VPC, reach out to [Astronomer support](https://support.astronomer.io) and provide the following information:

- Astro cluster ID and name
- AWS Account ID or Google Cloud project ID or Azure TenantID and Subscription ID of the target VPC/VNet
- Region of the target VPC/VNet (_AWS and Azure only_)
- VPC/VNet ID of the target VPC/VNet
- CIDR of the target VPC/VNet

From there, Astronomer initiates a peering request. To connect successfully, this peering request must be accepted by the owner of the target VPC/VNet in your organization.

Once the VPC/VNet peering connection is established, the owner of the target VPC will continue to work with our team to update the routing tables of both VPCs to direct traffic to each other.

### DNS considerations for VPC peering (_AWS only_)

To resolve DNS hostnames from your target VPC, every Astro VPC has **DNS Hostnames**, **DNS Resolutions**, and **Requester DNS Resolution** enabled. See AWS [Peering Connection settings](https://docs.aws.amazon.com/vpc/latest/peering/modify-peering-connections.html).

If your target VPC resolves DNS hostnames using **DNS Hostnames** and **DNS Resolution**, you must also enable the **Accepter DNS Resolution** setting on AWS. This allows the data plane to resolve the public DNS hostnames of the target VPC to its private IP addresses. To configure this option, see [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/peering/modify-peering-connections.html).

If your target VPC resolves DNS hostnames using [private hosted zones](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/hosted-zones-private.html), then you must associate your Route53 private hosted zone with the Astro VPC using instructions provided in [AWS Documentation](https://aws.amazon.com/premiumsupport/knowledge-center/route53-private-hosted-zone/).

To retrieve the ID of any Astro VPC, contact [Astronomer support](https://support.astronomer.io). If you have more than one Astro cluster, request the VPC ID of each cluster.

## AWS Transit Gateway

Use AWS Transit Gateway to connect one or more Astro clusters to other VPCs, AWS accounts, and on-premises networks supported by your organization.

AWS Transit Gateway is an alternative to VPC Peering on AWS. Instead of having to establish a direct connection between two VPCs, you can attach over 5,000 networks to a central transit gateway that has a single VPN connection to your corporate network.

While it can be more costly, AWS Transit Gateway requires less configuration and is often recommended for organizations connecting a larger number of VPCs. For more information, see [AWS Transit Gateway](https://aws.amazon.com/transit-gateway/).

AWS Transit Gateway doesn't provide built-in support for DNS resolution. If you need DNS integration, Astronomer recommends that you use the Route 53 Resolver service. For assistance integrating the Route 53 Resolver service with your Astronomer VPC, contact [Astronomer support](https://support.astronomer.io).

:::info

If your transit gateway is in a different region than your Astro cluster, contact [Astronomer support](https://support.astronomer.io). Astronomer support can create a new transit gateway in your AWS account for Astro and set up a cross-region peering connection with your existing transit gateway.

If Astronomer creates a new transit gateway in your AWS account for Astro, keep in mind that your organization will incur additional AWS charges for the new transit gateway as well as the inter-region transfer costs.

:::

### Prerequisites

- An Astro cluster
- An existing transit gateway in the same region as your Astro cluster
- Permission to share resources using AWS Resource Access Manager (RAM)

### Setup

1. In the Cloud UI, click the **Clusters** tab and copy the **Account ID** for your Astro cluster. This is an AWS account ID.
2. Create a resource share in AWS RAM with the account ID from step 1. See [Creating a resource share in AWS RAM](https://docs.aws.amazon.com/ram/latest/userguide/working-with-sharing-create.html).
3. Contact [Astronomer support](https://support.astronomer.io) and provide the CIDR block of the target VPC or on-premises network that you want to connect your Astro cluster with. From here, Astronomer approves the resource sharing request and creates a transit gateway peering attachment request to your network.
4. Accept the transit gateway peering attachment request from your network. See [Accept or reject a peering attachment request](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-peering.html#tgw-peering-accept-reject).
5. Create a static route from your CIDR block to the transit gateway. See [Add a route to the transit gateway route table](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-peering.html#tgw-peering-add-route).
6. Contact [Astronomer support](https://support.astronomer.io) to confirm that you have created the static route. Astronomer support will update the Astro VPC routing table to send traffic from your CIDR block through the transit gateway.
7. Optional. Repeat the steps for each Astro cluster that you want to connect to your transit gateway.

## AWS PrivateLink

Use AWS PrivateLink to create private connections from Astro to your AWS services without exposing your data to the public internet.

Astro clusters are pre-configured with the following AWS PrivateLink endpoint services:

- Amazon S3 - Gateway Endpoint
- Amazon Elastic Compute Cloud (Amazon EC2) Autoscaling - Interface Endpoint
- Amazon Elastic Container Registry (ECR) - Interface Endpoints for ECR API and Docker Registry API
- Elastic Load Balancing (ELB)  - Interface Endpoint
- AWS Security Token Service (AWS STS) - Interface Endpoint

To request additional endpoints, or assistance connecting to other AWS services, contact [Astronomer support](https://support.astronomer.io).

By default, Astronomer support activates the **Enable DNS Name** option on supported AWS PrivateLink endpoint services.  With this option enabled, you can make requests to the default public DNS service name instead of the public DNS name that is automatically generated by the VPC endpoint service. For example, `*.notebook.us-east-1.sagemaker.aws` instead of `vpce-xxx.notebook.us-east-1.vpce.sagemaker.aws`.

You'll incur additional AWS infrastructure costs for every AWS PrivateLink endpoint service that you use.  See [AWS PrivateLink pricing](https://aws.amazon.com/privatelink/pricing/).

## AWS IAM roles

To grant an Astro cluster access to a service that is running in an AWS account not managed by Astronomer, use AWS IAM roles. IAM roles on AWS are often used to manage the level of access a specific user, object, or group of users has to a resource. This includes an Amazon S3 bucket, Redshift instance, or secrets backend.

1. In the Cloud UI, click **Clusters** and then copy the value displayed in the **Cluster ID** column for the Astro cluster that needs access to AWS service resources.
2. Create an IAM role in the AWS account that contains your AWS service. See [Creating a role to delegate permissions to an AWS service](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-service.html).
3. In the AWS Management Console, go to the Identity and Access Management (IAM) dashboard.
4. Click **Roles** and in the **Role name** column, select the role you created in step 2.
5. Click the **Trust relationships** tab.
6. Click **Edit trust policy** and update the `arn` value:

```text {8}
    {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": [
                    "arn:aws:iam::<dataplane-AWS-account-ID>:role/AirflowS3Logs-<cluster-ID>"
                ]
            },
            "Action": "sts:AssumeRole"
        },
    ]
}
```

    Your Astro cluster's data plane account includes the `AirflowLogsS3-<clusterid>` role. When you configure an Airflow connection for a Deployment, specify this role in an [AWS Airflow Connection](https://airflow.apache.org/docs/apache-airflow-providers-amazon/stable/connections/aws.html) to allow your Deployment access your service.

7. Click **Update policy**.
8. In the Airflow UI or as an environment variable on Astro, create an Airflow connection to AWS for each Deployment that requires the resources you connected. See [Managing connections to Apache Airflow](https://airflow.apache.org/docs/apache-airflow-providers-amazon/stable/connections/aws.html).
8. Optional. Repeat these steps for each Astro cluster that requires access to external data services on AWS.

## Workload Identity (_GCP only_)

[Workload Identity](https://cloud.google.com/kubernetes-engine/docs/concepts/workload-identity) is recommended by Google as the best way for data pipelines running on GCP to access Google Cloud services in a secure and manageable way. All Astro clusters on GCP have Workload Identity enabled by default. Each Astro Deployment is associated with a Google service account that's created by Astronomer and is bound to an identity from your Google Cloud project's fixed workload identity pool.

To grant a Deployment on Astro access to external data services on GCP, such as BigQuery:

1. In the Cloud UI, select a Deployment and then copy the value in the **Namespace** field.
2. Use the Deployment namespace value and the name of your Google Cloud project to identify the Google service account for your Deployment.

    Google service accounts for Astro Deployments are formatted as follows:

    ```text
    astro-<deployment-namespace>@<gcp-project-name>.iam.gserviceaccount.com
    ```

    For a Google Cloud project named `astronomer-prod` and a Deployment namespace defined as `nuclear-science-2730`, for example, the service account for the Deployment would be:

    ```text
    astro-nuclear-science-2730@astronomer-prod.iam.gserviceaccount.com
    ```
4. Grant the Google service account for your Astro Deployment an IAM role that has access to your external data service. With the Google Cloud CLI, run:

    ```text
    gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT --member=serviceAccount:astro-<deployment-namespace>@<gcp-project-name>.iam.gserviceaccount.com --role=roles/viewer
    ```

    For instructions on how to grant your service account an IAM role in the Google Cloud console, see [Grant an IAM role](https://cloud.google.com/iam/docs/grant-role-console#grant_an_iam_role).

5. Optional. Repeat these steps for every Astro Deployment that requires access to external data services on GCP.

:::info

GCP has a 30-character limit for service account names. For Deployment namespaces which are longer than 24 characters, use only the first 24 characters when determining your service account name.

For example, if your Google Cloud project is named `astronomer-prod` and your Deployment namespace is `nuclear-scintillation-2730`, the service account name is:

```text
astro-nuclear-scintillation-27@astronomer-prod.iam.gserviceaccount.com
```

To determine your Deployment namespace that meets requirements without having to manually count characters, run:

```
echo astro-$(echo <deployment-namespace> | cut -c -24)
```

:::
