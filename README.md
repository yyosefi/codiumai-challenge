# EKS Cluster with Karpenter Terraform Setup

This repository provides a Terraform configuration to set up an AWS EKS cluster with Karpenter. Karpenter is configured with node pools to allow deployment of both x86_64(amd64) and arm64 (Graviton) instances.

## Prerequisites

Before getting started, ensure you have the following installed:

- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
- [Terraform](https://www.terraform.io/downloads.html)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [helm](https://helm.sh/docs/intro/install/)

You should also have an AWS account with the necessary permissions to create EKS clusters, IAM roles, and other AWS resources.

1. An EKS managed node group with a taint and label for the Karpenter controller. We use a nodeSelector to ensure controller pods run only on these nodes, not on those created by Karpenter. The taint keeps other pods off these nodes, reserving them for controller pods. CoreDNS has a toleration to run on these nodes, ensuring it can start and allow Karpenter to manage additional cluster resources. Without this, CoreDNS would fail to deploy, causing a "deadlock" as the controllers wait for resources.

2. The `eks-pod-identity-agent` addon will has be deployed to enable the Karpenter controller to use EKS Pod Identity for AWS permissions via an IAM role

3. The VPC subnets and node security group are tagged with `"karpenter.sh/discovery" = local.name` for discoverability. This allows the Karpenter controller to identify and use these resources to provision EC2 instances for the cluster.

4. An IAM role for the Karpenter controller will be created with a trust policy that trusts the EKS Pod Identity service principal. This allows the EKS Pod Identity service to provide AWS credentials to the Karpenter controller pods in order to call AWS APIs.

5. An IAM role for the nodes that Karpenter will create along with a cluster access entry which allows the nodes to acquire permissions to join the cluster. Karpenter will create and manage the instance profile that utilizes this IAM role.
6. An SQS queue that is subscribed to certain EC2 CloudWatch events will be created. This queue is used by Karpenter, allowing it to respond to certain EC2 lifecycle events and gracefully migrate pods off the instance before it is terminated.

## Getting Started
Clone the repository
```sh
git clone https://github.com/yyosefi/codiumai-challenge.git
cd codiumai-challenge
```

## Terraform
``` sh
terraform init
terraform plan
terraform apply -auto-aprove
```

After Terraform finishes creating the resources it will output important values of the Cluster, IAM Roles, ARNs Karpenter etc.
Note: The cluster is ready for deploying x86_64 CPU based deployments and includes `inflate` (wait image) to scale up.
Upon scale of the `inflate`, you should expect Karpenter to create node(s) to be able to run the new pods.

Following the below will demonstrate how to connect to the cluster and deploy ARM64 based pods.

## Set Environment variables
In order to create a deployment that will run on ARM64 CPU follow the following:
copy the value of 'karpenter_node_iam_role_name' Terraform output and use it as environment variable, for example:

```sh
export KARPENTER_NODE_IAM_ROLE_NAME="Karpenter-codiumai-challenge-20240706201736603900000018"

```

Copy the value of 'cluster_name' to an environment, for example:
```sh
export CLUSTER_NAME="codiumai-challenge"
```

## Connect
run the following command to create KUBE_CONFIG file for `kubectl` to be able to connect to the cluster, replace `eu-west-1` with the appropriate region and replace `codiumai-challenge` with the value of `cluster_name` used above:
:
```sh
aws eks --region eu-west-1 update-kubeconfig --name codiumai-challenge
```

## Validate
1. Test by listing the nodes in the cluster. You should see four Fargate nodes in the kubecluster:

    ```sh
    kubectl get nodes

    NAME                                       STATUS   ROLES    AGE     VERSION
    ip-10-0-26-82.eu-west-1.compute.internal   Ready    <none>   9m36s   v1.30.0-eks-036c24b
    ip-10-0-3-215.eu-west-1.compute.internal   Ready    <none>   9m35s   v1.30.0-eks-036c24b


    kubectl get deployment

    NAME      READY   UP-TO-DATE   AVAILABLE   AGE
    inflate   0/0     0            0           27m    
    ```

2. Provision the Karpenter `EC2NodeClass` and `NodePool` resources which provide Karpenter the necessary configurations to provision EC2 resources:

    ```sh
    cat arm64-ec2nodeclass.yaml | envsubst | kubectl apply -f -
    kubectl apply -f arm64-nodepool.yaml
    ```
Note that the `envsubst` is used to replace the following environment variable placeholders in the file with their respctive values set above: `${KARPENTER_NODE_IAM_ROLE_NAME}` and `${CLUSTER_NAME}`

## Deploy
3. Once the Karpenter resources are in place, Karpenter will provision the necessary EC2 resources to satisfy any pending pods in the scheduler's queue. You can demonstrate this with the example deployment provided. First deploy the example deployment which has the initial number replicas set to 0:

    ```sh
    kubectl apply -f deployment-arm64.yaml
    ```

4. When you scale the example deployment, you should see Karpenter respond by quickly provisioning EC2 resources to satisfy those pending pod requests:

    ```sh
    kubectl scale deployment arm64inflate --replicas=5
    ```

5. Listing the nodes should now show some EC2 compute that Karpenter has created for the example deployment:

    ```sh
    kubectl get nodes

    NAME                                        STATUS   ROLES    AGE     VERSION
    ip-10-0-26-82.eu-west-1.compute.internal   Ready      <none>   47m   v1.30.0-eks-036c24b
    ip-10-0-3-215.eu-west-1.compute.internal   Ready      <none>   47m   v1.30.0-eks-036c24b
    ip-10-0-3-61.eu-west-1.compute.internal    NotReady   <none>   6s    v1.30.0-eks-036c24b # <== EC2 created by Karpenter   
    ```


run the following command to check Karpenter controller logs
```sh
kubectl logs -f -n "kube-system" -l app.kubernetes.io/name=karpenter -c controller
```
press Ctrl-C to stop watching the log




## Destroy

Scale down the deployment to de-provision Karpenter created resources first:

```sh
kubectl delete deployment arm64inflate
terraform destroy
```
answer `yes` as input to Terraform to clean up AWS resources

## Developer Process

```mermaid
graph TD;
    A[Developer Process] --> B[Clone the Repository];
    B --> C[Run Terraform];
    C --> D[Connect to the Cluster] --> E[Run Shell commands] --> F[Destroy environment];