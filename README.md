# EKS Cluster with Karpenter Terraform Setup

This repository provides a Terraform configuration to set up an AWS EKS cluster with Karpenter. Karpenter is configured with node pools to allow deployment of both x86 and arm64 (Graviton) instances.

## Prerequisites

Before getting started, ensure you have the following installed:

- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
- [Terraform](https://www.terraform.io/downloads.html)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [helm](https://helm.sh/docs/intro/install/)

You should also have an AWS account with the necessary permissions to create EKS clusters, IAM roles, and other AWS resources.

## Developer Process

```mermaid
graph TD;
    A[Developer Process] --> B[Clone the Repository];
    B --> C[Run Terraform];
    C --> D[Run Shell Scripts];