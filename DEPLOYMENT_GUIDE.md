# AWS EKS Deployment Guide

## Complete Step-by-Step Deployment Instructions

### Prerequisites Checklist

- [ ] AWS Account with billing enabled
- [ ] IAM User with EC2, VPC, EKS, and IAM permissions
- [ ] AWS CLI installed and configured
- [ ] Terraform installed (>= 1.0)
- [ ] kubectl installed
- [ ] Git installed

---

## Step 1: Install Required Tools

### AWS CLI

```bash
# macOS
brew install awscli

# Linux
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Windows
msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2.msi
```

### Terraform

```bash
# macOS
brew install terraform

# Linux
wget https://releases.hashicorp.com/terraform/1.6.0/terraform_1.6.0_linux_amd64.zip
unzip terraform_1.6.0_linux_amd64.zip
sudo mv terraform /usr/local/bin/

# Windows
choco install terraform
```

### kubectl

```bash
# macOS
brew install kubectl

# Linux
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Windows
choco install kubernetes-cli
```

### eksctl (Optional but Recommended)

```bash
# macOS
brew install eksctl

# Linux
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz
sudo mv eksctl /usr/local/bin

# Windows
choco install eksctl
```

---

## Step 2: Configure AWS Credentials

### Option A: AWS CLI Configuration (Interactive)

```bash
aws configure

# Enter:
# AWS Access Key ID: [your-key-id]
# AWS Secret Access Key: [your-secret-key]
# Default region name: us-east-1
# Default output format: json
```

### Option B: Credentials File

```bash
cat ~/.aws/credentials

# Should contain:
# [default]
# aws_access_key_id = YOUR_KEY_ID
# aws_secret_access_key = YOUR_SECRET_KEY
```

### Option C: Environment Variables

```bash
export AWS_ACCESS_KEY_ID="your-key-id"
export AWS_SECRET_ACCESS_KEY="your-secret-key"
export AWS_DEFAULT_REGION="us-east-1"
```

### Verify Configuration

```bash
aws sts get-caller-identity

# Expected output:
# {
#     "UserId": "AIDAI...",
#     "Account": "123456789012",
#     "Arn": "arn:aws:iam::123456789012:user/your-user"
# }
```

---

## Step 3: Clone and Prepare Repository

```bash
# Clone the repository
git clone https://github.com/eyongfm/aws-eks-terraform.git
cd aws-eks-terraform

# Review the structure
ls -la
```

---

## Step 4: Customize Configuration

### Edit terraform.tfvars

```bash
vim terraform.tfvars
```

Modify values as needed:

```hcl
# AWS Region
aws_region = "us-east-1"

# Environment tag
environment = "prod"

# VPC CIDR - adjust if needed
vpc_cidr = "10.0.0.0/16"

# Public subnet CIDRs
public_subnet_cidrs = ["10.0.1.0/24", "10.0.2.0/24"]

# Private subnet CIDRs
private_subnet_cidrs = ["10.0.11.0/24", "10.0.12.0/24"]

# Cluster name - must be unique
cluster_name = "my-eks-cluster"

# Kubernetes version
kubernetes_version = "1.29"

# Node instance type (options: t3.small, t3.medium, t3.large, t3.xlarge)
instance_type = "t3.medium"

# Auto-scaling configuration
min_size = 1
desired_size = 2
max_size = 4
```

---

## Step 5: Initialize Terraform

```bash
# Initialize Terraform working directory
terraform init

# Expected output:
# Terraform has been successfully configured!
# You may now begin working with Terraform.
```

### Verify Initialization

```bash
ls -la .terraform/
# Should contain: modules, plugins, etc.
```

---

## Step 6: Validate Configuration

```bash
# Validate Terraform configuration
terraform validate

# Expected output:
# Success! The configuration is valid.
```

### Check for Issues

```bash
# Format check
terraform fmt -check

# If needed, auto-format
terraform fmt -recursive
```

---

## Step 7: Plan Deployment

```bash
# Create execution plan
terraform plan -out=tfplan

# Expected to create:
# - 1 VPC
# - 4 Subnets (2 public, 2 private)
# - 1 Internet Gateway
# - 2 NAT Gateways
# - 1 EKS Cluster
# - 1 Node Group
# - IAM roles and policies
# - Security groups
```

### Review Plan Output

```bash
# Display the plan
terraform show tfplan

# Count resources to be created
grep "Plan:" tfplan

# Should show something like:
# Plan: 27 to add, 0 to change, 0 to destroy.
```

---

## Step 8: Apply Configuration

⚠️ **Important**: This will incur AWS costs!

```bash
# Apply the Terraform configuration
terraform apply tfplan

# Wait for completion (typically 15-20 minutes)
```

### Monitor Progress

In another terminal:

```bash
# Watch CloudFormation stack creation
aws cloudformation describe-stacks \
  --stack-name eksctl-my-eks-cluster-cluster \
  --region us-east-1 \
  --query 'Stacks[0].StackStatus'
```

### After Successful Deployment

```bash
# Display outputs
terraform output

# Typical output:
# eks_cluster_endpoint = "https://xyz.eks.us-east-1.amazonaws.com"
# eks_cluster_name = "my-eks-cluster"
# vpc_id = "vpc-12345678"
```

---

## Step 9: Configure kubectl

```bash
# Update kubeconfig with new cluster
aws eks update-kubeconfig \
  --region us-east-1 \
  --name my-eks-cluster

# Verify kubeconfig
cat ~/.kube/config | grep current-context

# Should show:
# current-context: arn:aws:eks:us-east-1:123456789012:cluster/my-eks-cluster
```

---

## Step 10: Verify Cluster

### Check Cluster Info

```bash
kubectl cluster-info

# Expected output:
# Kubernetes control plane is running at https://xyz.eks.us-east-1.amazonaws.com
# CoreDNS is running at https://xyz.eks.us-east-1.amazonaws.com/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

### Check Nodes

```bash
kubectl get nodes

# Expected output:
# NAME                                         STATUS   ROLES    AGE   VERSION
# ip-10-0-11-xx.ec2.internal                  Ready    <none>   5m    v1.29.x
# ip-10-0-12-xx.ec2.internal                  Ready    <none>   5m    v1.29.x
```

### Check System Pods

```bash
kubectl get pods -n kube-system

# Expected output:
# NAME                              READY   STATUS    RESTARTS   AGE
# aws-node-xxxxx                    1/1     Running   0          3m
# aws-node-yyyyy                    1/1     Running   0          3m
# coredns-xxxxx                     1/1     Running   0          5m
# coredns-yyyyy                     1/1     Running   0          5m
# kube-proxy-xxxxx                  1/1     Running   0          3m
# kube-proxy-yyyyy                  1/1     Running   0          3m
```

### Verify Node Connectivity

```bash
# Check node resources
kubectl top nodes

# Check node labels
kubectl get nodes -L topology.kubernetes.io/zone

# Should show nodes across 2 availability zones
```

---

## Step 11: Deploy Sample Application (Optional)

### Create Namespace

```bash
kubectl create namespace demo
```

### Deploy nginx

```bash
kubectl create deployment nginx-demo \
  --image=nginx:latest \
  --replicas=2 \
  -n demo

# Wait for deployment
kubectl rollout status deployment/nginx-demo -n demo
```

### Expose Service

```bash
kubectl expose deployment nginx-demo \
  --type=LoadBalancer \
  --port=80 \
  -n demo

# Get load balancer URL
kubectl get svc -n demo
```

### Access Application

```bash
# Get external IP
EXTERNAL_IP=$(kubectl get svc nginx-demo -n demo -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# Access in browser
curl http://$EXTERNAL_IP
```

---

## Step 12: Setup Monitoring (Optional)

### Enable Container Insights

```bash
# Create IAM policy
aws iam create-policy \
  --policy-name CloudWatchContainerInsightsPolicy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": [
        "cloudwatch:PutMetricData",
        "ec2:DescribeVolumes",
        "ec2:DescribeTags",
        "logs:PutLogEvents",
        "logs:CreateLogStream",
        "logs:CreateLogGroup"
      ],
      "Resource": "*"
    }]
  }'
```

### Install CloudWatch Agent

```bash
kubectl apply -f https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluentd-quickstart.yaml
```

---

## Troubleshooting

### Common Issues

#### 1. Cluster Not Ready

```bash
# Check cluster status
aws eks describe-cluster \
  --name my-eks-cluster \
  --region us-east-1 \
  --query 'cluster.status'

# Wait for ACTIVE status
```

#### 2. Nodes Not Joining

```bash
# Check node group status
aws eks describe-nodegroup \
  --cluster-name my-eks-cluster \
  --nodegroup-name my-eks-cluster-node-group \
  --region us-east-1

# Check auto-scaling group
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names eks-my-eks-cluster-node-group-*
```

#### 3. kubectl Connection Failed

```bash
# Update kubeconfig again
aws eks update-kubeconfig \
  --region us-east-1 \
  --name my-eks-cluster \
  --kubeconfig ~/.kube/config

# Verify credentials
aws sts get-caller-identity
```

#### 4. Terraform Destroy Issues

```bash
# Force destroy with all dependencies
terraform destroy -auto-approve

# Check for remaining resources
aws ec2 describe-vpcs --filters "Name=tag:Name,Values=prod-vpc"
aws eks list-clusters --region us-east-1
```

---

## Cost Optimization

### Reduce Cluster Size

Edit `terraform.tfvars`:

```hcl
desired_size = 1  # Reduced from 2
instance_type = "t3.small"  # Smaller instance
```

Apply changes:

```bash
terraform plan
terraform apply
```

### Use Spot Instances (Future Enhancement)

Add to `main.tf`:

```hcl
capacity_type = "SPOT"  # Use spot instances
```

---

## Cleanup

### Delete Sample Application

```bash
kubectl delete namespace demo
```

### Destroy EKS Cluster

```bash
# Remove kubeconfig context
kubectl config delete-context arn:aws:eks:us-east-1:YOUR_ACCOUNT_ID:cluster/my-eks-cluster

# Destroy Terraform resources
terraform destroy -auto-approve

# Verify deletion
aws eks list-clusters --region us-east-1
```

---

## Next Steps

1. **Configure RBAC**: Set up role-based access control
2. **Install Helm**: Deploy applications using Helm charts
3. **Setup Ingress**: Configure ingress controller
4. **Enable Logging**: Configure CloudWatch logs
5. **Setup Monitoring**: Install Prometheus and Grafana
6. **Configure Auto-Scaling**: Setup Cluster Autoscaler and HPA
7. **Security Hardening**: Implement network policies and pod security policies

---

## Support

For issues:

1. Check [AWS EKS Documentation](https://docs.aws.amazon.com/eks/)
2. Review CloudWatch logs in AWS Console
3. Check security group rules
4. Verify IAM permissions

---

**Deployment completed successfully!** 🎉
