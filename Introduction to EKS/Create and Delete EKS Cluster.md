# Create and Delete EKS Cluster

## Overview

This guide covers the complete lifecycle of an Amazon EKS cluster using `eksctl`:

- create control plane
- associate IAM OIDC provider
- create managed node group
- verify cluster resources
- safely delete node groups and cluster

The flow is written for macOS terminal usage and is suitable for repeatable hands-on practice.

---

## EKS Core Objects You Should Know

Before creating a cluster, understand these building blocks:

### 1. Control Plane (Managed by AWS)

EKS runs and manages Kubernetes API server, etcd, scheduler, and controller manager.

### 2. Worker Nodes and Node Groups (Managed by You)

Your application Pods run on worker nodes. In this guide, we use an EKS managed node group.

### 3. Fargate Profiles (Optional)

Used when you want serverless Pod execution without managing EC2 worker nodes.

### 4. VPC Networking

EKS control plane and worker nodes depend on VPC subnets, route tables, and security groups.

### 5. IAM OIDC Provider

Required for IAM Roles for Service Accounts (IRSA), which enables Pods to access AWS services securely.

---

## Prerequisites

- AWS account with required permissions

- `aws`, `kubectl`, and `eksctl` installed

- AWS CLI configured (`aws configure`)

- Region decided (example: `us-east-1`)

Verify tools:

```bash
aws --version
kubectl version --client
eksctl version
aws sts get-caller-identity
```

---

## Step-01: Create EKS Cluster (Control Plane Only)

Create cluster without node group first:

```bash
eksctl create cluster --name=eksdemo \
                      --region=us-east-1 \
                      --zones=us-east-1a,us-east-1b \
                      --without-nodegroup
```

Notes:

- cluster creation usually takes 15 to 20 minutes
- this creates control plane and required networking stacks

Check cluster list:

```bash
eksctl get cluster
```

---

## Step-02: Associate IAM OIDC Provider

To use IRSA, associate OIDC provider with your cluster:

```bash
eksctl utils associate-iam-oidc-provider \
    --region us-east-1 \
    --cluster eksdemo \
    --approve
```

Template form:

```bash
eksctl utils associate-iam-oidc-provider \
    --region <region-code> \
    --cluster <cluster-name> \
    --approve
```

---

## Step-03: Create EC2 Key Pair (for Node SSH)

Create a key pair named `kube-demo` in AWS EC2 console, then download `kube-demo.pem`.

On macOS, secure key permissions:

```bash
chmod 400 kube-demo.pem
```

This key will be used to SSH into worker nodes.

---

## Step-04: Create Managed Node Group (Public Subnets)

```bash
eksctl create nodegroup --cluster=eksdemo \
                       --region=us-east-1 \
                       --name=eksdemo-ng-public \
                       --node-type=t3.medium \
                       --nodes=2 \
                       --nodes-min=2 \
                       --nodes-max=4 \
                       --node-volume-size=20 \
                       --ssh-access \
                       --ssh-public-key=kube-demo \
                       --managed \
                       --asg-access \
                       --external-dns-access \
                       --full-ecr-access \
                       --appmesh-access \
                       --alb-ingress-access
```

This creates worker nodes and attaches additional IAM permissions for common add-ons.

---

## Step-05: Verify Cluster, Node Group, and Nodes

### Verify with eksctl and kubectl

```bash
# List EKS clusters
eksctl get cluster

# List node groups for a cluster
eksctl get nodegroup --cluster=eksdemo --region=us-east-1

# Check kubectl context
kubectl config view --minify

# List worker nodes
kubectl get nodes -o wide
```

### Verify AWS-side resources

- EKS console: cluster status should be `Active`
- Node group status should be `Active`
- EC2 instances should be running
- Worker node IAM role should have expected policies
- Node group subnet route table should include internet route (`0.0.0.0/0 -> igw-...`) if public

---

## Step-06: SSH into Worker Node (macOS)

Find the EC2 public IP from AWS console, then connect:

```bash
ssh -i kube-demo.pem ec2-user@<worker-node-public-ip>
```

If connection fails, verify:

- key pair name used in node group matches downloaded `.pem`
- security group allows inbound SSH (`22`) from your IP
- `.pem` file permissions are `400`

---

## Step-07: Update Worker Node Security Group (If Needed)

For lab scenarios, you may temporarily allow broader access for testing.

Recommended approach:

- allow only required ports
- restrict source CIDR to your own IP wherever possible

Avoid leaving open inbound rules after testing.

---

## Deletion Workflow (Cluster Cleanup)

Always clean up after practice to avoid AWS charges.

### Option A: Delete Entire Cluster (Recommended)

```bash
eksctl delete cluster --name=eksdemo --region=us-east-1
```

This removes:

- EKS control plane
- managed node groups
- related CloudFormation stacks created by `eksctl`

### Option B: Delete Node Group First, Then Cluster

```bash
# Delete specific node group
eksctl delete nodegroup \
  --cluster=eksdemo \
  --name=eksdemo-ng-public1 \
  --region=us-east-1

# Delete cluster
eksctl delete cluster --name=eksdemo --region=us-east-1
```

### Validate Deletion

```bash
eksctl get cluster
aws cloudformation list-stacks --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE
```

Also verify in AWS console:

- cluster no longer appears in EKS
- worker instances terminated in EC2
- load balancers and EBS volumes are cleaned up

---

## Common Issues and Fixes

| Issue | Likely Cause | Quick Fix |
|---|---|---|
| Cluster creation fails | IAM permission gaps or unsupported AZs | verify IAM rights, try different AZ pair |
| `kubectl get nodes` shows none | node group not created or failed | create/check node group status |
| OIDC association fails | wrong region/cluster name | re-run associate command with correct values |
| SSH fails | SG, key pair, or pem permissions | open port 22 for your IP, verify key pair, run `chmod 400` |
| Delete stuck in CloudFormation | dependent resources still attached | identify dependent resources and retry delete |

---

## Best Practices

- Create cluster without node group first for clearer staged provisioning.

- Use IRSA (OIDC + service accounts) instead of static AWS keys in Pods.

- Keep node group size minimal for learning to control cost.

- Tag resources clearly for easy cleanup.

- Delete unused clusters quickly.

---

## Interview Questions

### 1. Why do we associate an IAM OIDC provider with EKS?

**Answer:**
It enables IAM Roles for Service Accounts (IRSA), so Pods can access AWS services using short-lived credentials tied to Kubernetes service accounts.

---

### 2. What is the difference between EKS control plane and node group?

**Answer:**
Control plane runs Kubernetes management components and is managed by AWS; node groups are worker compute resources where application Pods run.

---

### 3. Which command is used to delete an EKS cluster created with eksctl?

**Answer:**
`eksctl delete cluster --name=<cluster-name> --region=<region>`.

---

### 4. Why should EKS clusters be deleted after lab sessions?

**Answer:**
To prevent ongoing charges from control plane, EC2 nodes, load balancers, and storage resources.

---

## Summary

- EKS setup flow: create cluster -> associate OIDC -> create node group -> verify nodes -> optional SSH validation.

- Cluster deletion is a mandatory part of the lifecycle for cost control.

- `eksctl` provides a reliable and repeatable CLI workflow for both creation and cleanup.

---
