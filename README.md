# GitHub-Action-Runners-on-AWS-EKS

This guide demonstrates how to set up GitHub Action Runners on AWS EKS (Elastic Kubernetes Service) using the GitHub Actions Controller and Karpenter for just-in-time node provisioning. The setup ensures cost efficiency by dynamically scaling nodes based on workload demands.

## Prerequisites
- An existing AWS EKS cluster. If you don't have one, you can use the provided Terraform script to create a simple EKS cluster.
- knowledge of Kubernetes, Helm, and GitHub Actions.

## Access the EKS Cluster
To access your cluster, update your kubeconfig:

```Bash
aws eks update-kubeconfig --name <cluster-name>--region <Region>
```

Verify access to the cluster:

```bash
kubectl get pods -A
```

## Install Karpenter

Karpenter will manage the just-in-time provisioning of nodes. Install it using Helm:

```
helm registry logout public.ecr.aws

helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter --version "0.36.2" --namespace kube-system --create-namespace \
  --set "settings.clusterName=<cluster-name>" \
  --set controller.resources.requests.cpu=0.5 \
  --set controller.resources.requests.memory=1Gi \
  --set controller.resources.limits.cpu=0.5  \
  --set controller.resources.limits.memory=1Gi \
  --set controller.ttlSecondsAfterEmpty=300

```

