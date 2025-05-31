# A simple but Ready-to-test AWS EKS PrivateBin Deployment

A simple but Ready-to-test AWS EKS PrivateBin App


PrivateBin is a minimalist PHP-Based, open source online pastebin which provides a secure way to share text documents, code samples, logs etc  where the server has zero knowledge of pasted data. Data is encrypted and decrypted in the browser using 256-bit AES Encryption. Also it used HTTPS to ensure security and help share passwords other important Data privately and securely.

Main Features:

1: Zero-Knowledge Encryption: All data is encrypted and decrypted in the browser, ensuring the server has no knowledge of the content.
2: Password Protection: Users can set a password to further protect their pastes, preventing unauthorized access.
3: Expiration Options: Choose from various expiration times, including "forever" and "burn after reading" options.
4: File Uploads: Upload files with image, media, and PDF previews (disabled by default, size limit adjustable).


There is is the all-in-one image (Docker Hub / GitHub) that can be used with any storage backend 

https://github.com/PrivateBin/docker-nginx-fpm-alpine

Behind the scenes, There is a Pod which runs Nginx and php-fpm contaieners with DNS challenbeg + AWS ACM  + Route53 configuration +AWS ALB controller 80 443 + AWS CSI-EBS controlerr PVC as Stroage backend

---

## Directory Structure

/root/bin-reborncloud/
├── README.md
├── alb-csi-policy.json
├── bin-privatebin-deployment.yaml
├── bin-privatebin-ebs-storageclass.yaml
├── bin-privatebin-eks-cluster-config.yaml
├── bin-privatebin-ingress.yaml
├── bin-privatebin-namespace.yaml
├── bin-privatebin-pvc.yaml
└── bin-privatebin-service.yaml

---

## Prerequisites

- AWS CLI, kubectl, eksctl, and helm installed and configured
- AWS account with permissions to create EKS/IAM resources
- ACM certificate for your domain (e.g., bin.reborncloud.online)
- Route53 hosted zone for your domain

---

## 1. Cluster Creation

eksctl create cluster -f bin-privatebin-eks-cluster-config.yaml

---

Find your VPC ID:

aws eks describe-cluster \
  --name "<CLUSTER_NAME>" \
  --query "cluster.resourcesVpcConfig.vpcId" \
  --output text

List public subnets (with IGW):

aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=<VPC_ID>" \
  --output json | jq -r '
    .RouteTables[]
    | select(.Routes[]? | (.GatewayId? | type == "string" and startswith("igw-")))
    | .Associations[]?.SubnetId' | sort | uniq

Tag these as public:

aws ec2 create-tags --resources <public-subnet-ids> \
  --tags Key=kubernetes.io/cluster/<CLUSTER_NAME>,Value=owned \
         Key=kubernetes.io/role/elb,Value=1

List private subnets (no IGW):

aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=<VPC_ID>" \
  --output json | jq -r '
    .RouteTables[]
    | select(all(.Routes[]?; (.GatewayId? | type != "string" or (startswith("igw-") | not))))
    | .Associations[]?.SubnetId' | sort | uniq

Tag these as private:

aws ec2 create-tags --resources <private-subnet-ids> \
  --tags Key=kubernetes.io/cluster/<CLUSTER_NAME>,Value=owned \
         Key=kubernetes.io/role/internal-elb,Value=1


Open Security Group Ports 80/443 for ALB

Find security groups:

aws ec2 describe-security-groups \
  --filters "Name=tag:kubernetes.io/cluster/<CLUSTER_NAME>,Values=owned" \
  --query "SecurityGroups[*].GroupId" \
  --output text


Allow inbound HTTP/HTTPS:

aws ec2 authorize-security-group-ingress \
  --group-id <sg-id> \
  --protocol tcp --port 80 --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
  --group-id <sg-id> \
  --protocol tcp --port 443 --cidr 0.0.0.0/0

---

## 2. Enable OIDC for IRSA

eksctl utils associate-iam-oidc-provider --cluster privatebin-cluster --approve

---

## 3. IAM Policies

a. Download & Create ALB Policy

curl -o policies/alb-csi-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.6.1/docs/install/iam_policy.json

aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://policies/alb-csi-policy.json

b. EBS CSI Policy

Use AWS managed policy: arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy

---

## 4. Create IAM Service Accounts

a. ALB Ingress Controller

eksctl create iamserviceaccount \
  --cluster=privatebin-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve

b. EBS CSI Driver

eksctl create iamserviceaccount \
  --cluster=privatebin-cluster \
  --namespace=kube-system \
  --name=ebs-csi-controller-sa \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve

---

## 5. Install Controllers

a. AWS Load Balancer Controller

kubectl apply -f https://github.com/aws/eks-charts/raw/master/stable/aws-load-balancer-controller/crds/crds.yaml

helm repo add eks https://aws.github.io/eks-charts

helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=<CLUSTER_NAME> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=<REGION> \
  --set vpcId=$(aws eks describe-cluster --name <CLUSTER_NAME> --query "cluster.resourcesVpcConfig.vpcId" --output text) \
  --set image.repository=public.ecr.aws/eks/aws-load-balancer-controller

b. EBS CSI Driver

kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/ecr/?ref=release-1.30"

---

## 6. Deploy PrivateBin Resources

kubectl apply -f bin-privatebin-namespace.yaml
kubectl apply -f bin-privatebin-ebs-storageclass.yaml
kubectl apply -f bin-privatebin-pvc.yaml
kubectl apply -f bin-privatebin-deployment.yaml
kubectl apply -f bin-privatebin-service.yaml
kubectl apply -f bin-privatebin-ingress.yaml

---

## 7. Validation & Troubleshooting

kubectl get nodes -o wide

kubectl logs -n privatebin -l app=privatebin

kubectl get ingress privatebin-ingress -n privatebin -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'

curl -I https://bin.reborncloud.online


## 8. Configure DNS

In Route53, create a CNAME or Alias record for your domain (e.g., bin.reborncloud.online) pointing to the ALB DNS name.

Now you have a production-ready, automated PrivateBin deployment on AWS EKS with ALB, EBS, ACM, and Route53 services integrated.

Resolve your URL on a Browser : https://bin.reborncloud.online
---

## References for Troublshooting Common Pitfalls

- AWS Load Balancer Controller: https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.6/deploy/installation/
- Amazon EBS CSI Driver: https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html
- PrivateBin: https://privatebin.info/
- eksctl IAM Policies: https://eksctl.io/usage/iam-policies/

---

## License

MIT
---
