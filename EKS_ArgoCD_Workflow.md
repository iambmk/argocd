# EKS + ArgoCD Setup Guide

## Overview

This guide describes how to provision an Amazon EKS cluster, install ArgoCD, deploy a GitOps application, and clean up the environment.

It is written for a Linux EC2 instance running Amazon Linux with `t3.medium` instance type.

---

## Prerequisites

- Launch an EC2 instance using the Amazon Linux AMI.
- Use instance type `t3.medium`.
- Allow all traffic during initial setup (security groups may be tightened later).
- Log in to the EC2 instance before starting the steps below.
- Ensure the EC2 instance has an IAM role attached with sufficient permissions for EKS, EC2, IAM, CloudFormation, VPC, and related AWS resources.

Example permissive IAM policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "*",
      "Resource": "*"
    }
  ]
}
```

> Note: In production, use least-privilege IAM policies rather than this wide-open example.

---

## 1. Install kubectl

Follow the AWS EKS documentation for installing `kubectl` on Linux.

```bash
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.35.3/2026-04-08/bin/linux/amd64/kubectl
chmod +x ./kubectl
mkdir -p "$HOME/bin"
cp ./kubectl "$HOME/bin/kubectl"
export PATH="$HOME/bin:$PATH"
```

Verify the installation:

```bash
kubectl version --client
```

---

## 2. Install eksctl

Install the latest `eksctl` binary for `amd64` platforms.

```bash
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"
```

(Optional) verify the checksum:

```bash
curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_checksums.txt" | grep "$PLATFORM" | sha256sum --check
```

Install `eksctl`:

```bash
tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz
sudo install -m 0755 /tmp/eksctl /usr/local/bin && rm /tmp/eksctl
```

Confirm the tool is installed:

```bash
eksctl version
```

---

## 3. Create the EKS cluster

Provision the EKS cluster and managed node group.

```bash
eksctl create cluster --name argocd --region us-east-1 --nodegroup-name standard-nodes --node-type t3.medium --nodes 2
```

### Explanation

- `--name argocd`: names the cluster `argocd`.
- `--region us-east-1`: deploys the cluster in US East (N. Virginia).
- `--nodegroup-name standard-nodes`: names the worker node group.
- `--node-type t3.medium`: selects `t3.medium` EC2 instances.
- `--nodes 2`: creates two worker nodes.

Verify the cluster nodes:

```bash
kubectl get nodes
```

### Explanation

This command checks the cluster API and confirms the worker nodes are active and in the `Ready` state.

---

## 4. Install ArgoCD

Create the dedicated namespace and install ArgoCD manifests.

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.4.7/manifests/install.yaml
```

Expose the ArgoCD server via a LoadBalancer service:

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

### Explanation

- `kubectl create namespace argocd`: creates an isolated namespace for ArgoCD.
- `kubectl apply -n argocd -f ...`: installs the ArgoCD control plane resources.
- `kubectl patch svc argocd-server`: converts the ArgoCD server service into a LoadBalancer so it is reachable externally.

---

## 5. Get ArgoCD access details

Retrieve the public LoadBalancer hostname and initial admin password.

```bash
export ARGOCD_SERVER=$(kubectl get svc argocd-server -n argocd -o json | jq --raw-output '.status.loadBalancer.ingress[0].hostname')
echo "$ARGOCD_SERVER"

export ARGO_PWD=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
echo "$ARGO_PWD"
```

### Explanation

- `ARGOCD_SERVER`: stores the public DNS hostname of the ArgoCD LoadBalancer.
- `ARGO_PWD`: retrieves the initial admin password from the ArgoCD secret and decodes it.

> Note: If `jq` is not installed, install it first with `sudo yum install -y jq`.

---

## 6. Deploy your repository in ArgoCD

Open the ArgoCD UI in a browser using `ARGOCD_SERVER` and log in with the username `admin` and the displayed password.

Then:

1. Go to `Settings` → `Repositories` and connect your Git repository.
2. Create a new application in the ArgoCD UI.
3. Point it to the repository path containing the deployment and service manifests.
4. Sync the application.

Once deployed, ArgoCD will create the Deployment, Service, ReplicaSet, and Pods defined in the repo.

---

## 7. Verify GitOps deployments

Check that the application resources are deployed successfully.

```bash
kubectl get pods
kubectl get all
```

### Explanation

- `kubectl get pods`: shows pod status for the current namespace.
- `kubectl get all`: lists all key resources such as pods, services, deployments, and replica sets.

---

## 8. Optional: Inspect ArgoCD applications

```bash
kubectl get applications -n argocd
```

### Explanation

This shows ArgoCD-managed application objects and their sync status.

---

## 9. Cleanup and teardown

When you are done, remove the ArgoCD application and cluster resources.

```bash
kubectl delete application saidemy-web-app -n argocd
kubectl delete svc argocd-server -n argocd
kubectl get namespaces
kubectl delete namespace argocd
kubectl delete deployment web-deployment
kubectl delete service web-service
eksctl delete cluster argocd --region us-east-1
```

### Explanation

- `kubectl delete application`: removes the ArgoCD-managed application and its deployed resources.
- `kubectl delete svc argocd-server`: removes the LoadBalancer service.
- `kubectl get namespaces`: verifies what namespaces remain.
- `kubectl delete namespace argocd`: deletes the ArgoCD namespace and contained resources.
- `kubectl delete deployment/service`: cleans up any remaining app resources.
- `eksctl delete cluster`: destroys the EKS cluster and related AWS infrastructure.

---

## Notes

- This guide assumes the AWS CLI is configured with credentials and a default region.
- Use a more restrictive IAM policy in production environments.
- Verify the correct `kubectl` and `eksctl` versions before running the cluster creation command.

---

## Reference

- ArgoCD getting started: https://argo-cd.readthedocs.io/en/stable/getting_started/
