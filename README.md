# AWS EKS PrivateBin Deployment (Ready-to-Test)

This repository provides a ready-to-deploy configuration of [PrivateBin](https://privatebin.info/) on AWS EKS using the following AWS services:

- AWS ALB Ingress Controller
- Amazon EBS CSI Driver
- Route 53
- ACM (SSL Certificates)

---

## What is PrivateBin?

**PrivateBin** is a minimalist, open-source pastebin written in PHP. It allows users to share code, logs, or text securely with **zero server-side knowledge** of the content. All data is **end-to-end encrypted** in the browser using **256-bit AES**.

### Key Features

- **Zero-Knowledge Encryption**: Data is encrypted/decrypted in the browser; the server cannot read it.
- **Password Protection**: Optional password to restrict access.
- **Expiration Options**: Choose when pastes expire (e.g., burn after reading).
- **Optional File Uploads**: Supports image/media/PDF preview (disabled by default).

Docker Image:

```sh
https://github.com/PrivateBin/docker-nginx-fpm-alpine
```

This setup runs a Pod with **Nginx** and **PHP-FPM**, integrates DNS challenge, ALB (ports 80/443), and uses Amazon EBS CSI driver for persistent storage.

---

## Project Structure

```sh
/root/bin-reborncloud/
├── README.md
├── alb-csi-policy.json
├── bin-privatebin-deployment.yaml
├── bin-privatebin-ebs-storageclass.yaml
├── bin-privatebin-eks-cluster-config.yaml
├── bin-privatebin-ingress.yaml
├── bin-privatebin-namespace.yaml
├── bin-privatebin-pvc.yaml
├── bin-privatebin-service.yaml
```
---

## Prerequisites

Ensure you have the following tools installed and configured:

- AWS CLI
- \`kubectl\`, \`eksctl\`, and \`helm\`
- AWS account with EKS/IAM permissions
- ACM Certificate for your domain (e.g., \`bin.reborncloud.online\`)
- Route 53 Hosted Zone for your domain

---

## Step 1: Create the EKS Cluster

```sh
eksctl create cluster -f bin-privatebin-eks-cluster-config.yaml
```

---

## Step 2: Configure VPC Subnets for Load Balancers

### Get VPC ID:

```sh
aws eks describe-cluster \
  --name "<CLUSTER_NAME>" \
  --query "cluster.resourcesVpcConfig.vpcId" \
  --output text
```

### Get and Tag Public Subnets:

```sh
aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=<VPC_ID>" \
  --output json | jq -r \
  '.RouteTables[] | select(.Routes[]?.GatewayId? | startswith("igw-")) | .Associations[]?.SubnetId' | sort | uniq

aws ec2 create-tags --resources <SUBNET_IDS> \
  --tags Key=kubernetes.io/cluster/<CLUSTER_NAME>,Value=owned \
         Key=kubernetes.io/role/elb,Value=1
```

### Get and Tag Private Subnets:

```sh
aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=<VPC_ID>" \
  --output json | jq -r \
  '.RouteTables[] | select(all(.Routes[]?; (.GatewayId? | type != "string" or (startswith("igw-") | not)))) | .Associations[]?.SubnetId' | sort | uniq

aws ec2 create-tags --resources <SUBNET_IDS> \
  --tags Key=kubernetes.io/cluster/<CLUSTER_NAME>,Value=owned \
         Key=kubernetes.io/role/internal-elb,Value=1
```

---

## Step 3: Open Ports 80/443 on Security Groups

### Get Security Group ID:

```sh
aws ec2 describe-security-groups \
  --filters "Name=tag:kubernetes.io/cluster/<CLUSTER_NAME>,Values=owned" \
  --query "SecurityGroups[*].GroupId" \
  --output text
```

### Authorize HTTP/HTTPS Access:

```sh
aws ec2 authorize-security-group-ingress \
  --group-id <GROUP_ID> \
  --protocol tcp --port 80 --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
  --group-id <GROUP_ID> \
  --protocol tcp --port 443 --cidr 0.0.0.0/0
```

---

## Step 4: Enable OIDC for IAM Roles

```sh
eksctl utils associate-iam-oidc-provider \
  --cluster privatebin-cluster \
  --approve
```

---

## Step 5: Create IAM Policies

### a. ALB Policy

```sh
curl -o policies/alb-csi-policy.json \
  https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.6.1/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://policies/alb-csi-policy.json
```

### b. EBS CSI Policy

Use the following AWS-managed policy:

```sh
arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy
```

---

## Step 6: Create IAM Service Accounts

### a. ALB Ingress Controller

```sh
eksctl create iamserviceaccount \
  --cluster=privatebin-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve
```

### b. EBS CSI Driver

```sh
eksctl create iamserviceaccount \
  --cluster=privatebin-cluster \
  --namespace=kube-system \
  --name=ebs-csi-controller-sa \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve
```

---

## Step 7: Install Controllers

### a. AWS Load Balancer Controller

```sh
kubectl apply -f \
  https://github.com/aws/eks-charts/raw/master/stable/aws-load-balancer-controller/crds/crds.yaml

helm repo add eks https://aws.github.io/eks-charts

helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=<CLUSTER_NAME> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=<AWS_REGION> \
  --set vpcId=$(aws eks describe-cluster --name <CLUSTER_NAME> --query "cluster.resourcesVpcConfig.vpcId" --output text) \
  --set image.repository=public.ecr.aws/eks/aws-load-balancer-controller
```

### b. EBS CSI Driver

```sh
kubectl apply -k \
  "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/ecr/?ref=release-1.30"
```
---

## Step 8: Deploy PrivateBin Resources

```sh
kubectl apply -f bin-privatebin-namespace.yaml
kubectl apply -f bin-privatebin-ebs-storageclass.yaml
kubectl apply -f bin-privatebin-pvc.yaml
kubectl apply -f bin-privatebin-deployment.yaml
kubectl apply -f bin-privatebin-service.yaml
kubectl apply -f bin-privatebin-ingress.yaml
```

---

## Step 9: Validate the Deployment

```sh
kubectl get nodes -o wide
kubectl logs -n privatebin -l app=privatebin
kubectl get ingress privatebin-ingress -n privatebin -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
curl -I https://bin.reborncloud.online
```
---

## Step 10: Configure DNS

In **Route 53**, create a **CNAME** or **Alias Record** pointing your domain (e.g., \`bin.reborncloud.online\`) to the ALB DNS name.

---

## Final Result

Your PrivateBin deployment should now be accessible at:

```sh
https://bin.reborncloud.online
```
---

## References & Troubleshooting

- AWS Load Balancer Controller: https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.6/deploy/installation/
- Amazon EBS CSI Driver: https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html
- PrivateBin: https://privatebin.info/
- eksctl IAM Policies: https://eksctl.io/usage/iam-policies/

---
