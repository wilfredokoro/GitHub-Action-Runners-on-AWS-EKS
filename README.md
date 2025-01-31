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

Apply the NodePool and EC2NodeClass configurations, create a nodepoo.yaml file

```
kubectl apply -f nodepool.yaml
```

## Install Action Controller:
Install the GitHub Actions Controller using Helm:

```
helm install arc \
    --namespace "arc-systems" \
    --create-namespace \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller
```
Install the runner set:
Create a Personal Access Token (PAT) for GitHub.

```
helm install "arc-runner-set" \
    --namespace "arc-runners" \
    --create-namespace \
    --set githubConfigUrl="https://github.com/yourusername/yourreponame" \
    --set githubConfigSecret.github_token="ghp_somepattokenhere342342234" \
    --set "template.spec.containers[0].resources.requests.cpu=7" \
    --set "template.spec.containers[0].resources.requests.memory=14Gi" \
    --set "template.spec.containers[0].resources.limits.cpu=7" \
    --set "template.spec.containers[0].resources.limits.memory=14Gi" \
    --set "template.spec.containers[0].name=runner" \
    --set "template.spec.containers[0].image=ghcr.io/actions/actions-runner:latest" \
    --set "template.spec.containers[0].command[0]=/home/runner/run.sh" \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```
## Create a GitHub Actions Workflow:

```
name: CI
on:
  workflow_dispatch:
jobs:
  build:
    runs-on: arc-runner-set
    steps:
      - uses: actions/checkout@v4
      - name: Run a one-line script
        run: echo Hello, world!

      - name: Run a multi-line script
        run: |
          cat README.md
```

## Run the Workflow

Trigger the workflow from the GitHub Actions tab in your repository. The runner will be provisioned by Karpenter, and the job will execute on the newly created instance. After the job completes, Karpenter will terminate the instance to save costs.



