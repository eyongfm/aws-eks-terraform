# AWS EKS Terraform Infrastructure

## Overview

This Terraform configuration creates a production-ready AWS EKS (Elastic Kubernetes Service) cluster with:

- ✅ **VPC** with public and private subnets across 2 availability zones
- ✅ **NAT Gateways** for secure outbound internet access
- ✅ **Internet Gateway** for inbound connectivity
- ✅ **EKS Cluster** with managed node group (auto-scaling 1-4 nodes)
- ✅ **Security Groups** and **IAM Roles** with least privilege access
- ✅ **Multi-AZ** deployment for high availability

## Architecture

### Network Topology

```
Internet
    ↓
Internet Gateway (IGW)
    ↓
┌─────────────────────────────────────┐
│       Public Subnets (2)             │
│  10.0.1.0/24 | 10.0.2.0/24          │
│  NAT Gateway 1 | NAT Gateway 2       │
└─────────────────────────────────────┘
    ↓                  ↓
┌──────────────┐  ┌──────────────┐
│ Private AZ1a │  │ Private AZ1b │
│ 10.0.11.0/24 │  │ 10.0.12.0/24 │
│   Worker Nodes  │  │  Worker Nodes  │
│   (t3.medium)   │  │  (t3.medium)   │
└──────────────┘  └──────────────┘
    ↓                  ↓
└─────────────────────────────────────┐
│   EKS Control Plane (AWS Managed)   │
│   Kubernetes 1.29                   │
└─────────────────────────────────────┘
```

## Prerequisites

1. **AWS Account** with appropriate permissions
2. **Terraform** (>= 1.0)
3. **AWS CLI** configured with credentials
4. **kubectl** (for managing the cluster)

### Install Terraform

```bash
# macOS
brew install terraform

# Linux
wget https://releases.hashicorp.com/terraform/1.6.0/terraform_1.6.0_linux_amd64.zip
unzip terraform_1.6.0_linux_amd64.zip
mv terraform /usr/local/bin/

# Windows
choco install terraform
```

## Quick Start

### 1. Initialize Terraform

```bash
terraform init
```

### 2. Review Configuration

Edit `terraform.tfvars` to customize:
- Cluster name
- Node instance type
- Number of worker nodes
- VPC CIDR blocks

```hcl
aws_region         = "us-east-1"
cluster_name       = "my-eks-cluster"
instance_type      = "t3.medium"
desired_size       = 2
min_size           = 1
max_size           = 4
```

### 3. Plan Deployment

```bash
terraform plan
```

This shows all resources that will be created.

### 4. Apply Configuration

```bash
terraform apply
```

Confirm with `yes` when prompted.

### 5. Configure kubectl

```bash
aws eks update-kubeconfig \
  --region us-east-1 \
  --name my-eks-cluster
```

### 6. Verify Cluster

```bash
# Check nodes
kubectl get nodes

# Check cluster info
kubectl cluster-info

# Check pods
kubectl get pods -A
```

## File Structure

```
.
├── main.tf                      # Main Terraform configuration
├── variables.tf                 # Variable definitions
├── outputs.tf                   # Output values
├── terraform.tfvars             # Terraform variables (customize here)
├── architecture-diagram.drawio  # Draw.io architecture diagram
└── README.md                    # This file
```

## Configuration Details

### VPC Configuration

| Parameter | Value | Description |
|-----------|-------|-------------|
| VPC CIDR | 10.0.0.0/16 | Main VPC network block |
| Public Subnet 1 | 10.0.1.0/24 | AZ: us-east-1a |
| Public Subnet 2 | 10.0.2.0/24 | AZ: us-east-1b |
| Private Subnet 1 | 10.0.11.0/24 | AZ: us-east-1a |
| Private Subnet 2 | 10.0.12.0/24 | AZ: us-east-1b |

### EKS Configuration

| Parameter | Value | Description |
|-----------|-------|-------------|
| Kubernetes Version | 1.29 | Latest stable version |
| Instance Type | t3.medium | 1 vCPU, 1 GB RAM per node |
| Min Nodes | 1 | Minimum cluster size |
| Desired Nodes | 2 | Default cluster size |
| Max Nodes | 4 | Maximum cluster size (auto-scaling) |
| Availability Zones | 2 | Multi-AZ for HA |

### Security Configuration

- **IAM Roles**: Least privilege access
- **Security Groups**: Control inbound/outbound traffic
- **Private Subnets**: Worker nodes in private subnets
- **NAT Gateways**: Secure outbound internet access
- **Public Endpoints**: API server accessible (configurable)

## IAM Roles

### EKS Cluster Role

```json
{
  "Service": "eks.amazonaws.com",
  "Policies": ["AmazonEKSClusterPolicy"]
}
```

### EKS Node Role

```json
{
  "Service": "ec2.amazonaws.com",
  "Policies": [
    "AmazonEKSWorkerNodePolicy",
    "AmazonEKS_CNI_Policy",
    "AmazonEC2ContainerRegistryReadOnly"
  ]
}
```

## Network Flows

### Outbound (Pod → Internet)

1. Pod sends traffic
2. VPC CNI routes to worker node
3. Worker node routes to private subnet route table
4. Route table sends to NAT Gateway
5. NAT Gateway sends to Internet Gateway
6. IGW forwards to internet

### Inbound (Internet → Service)

1. External client connects to service
2. Traffic reaches Internet Gateway
3. IGW routes to service (via load balancer or ingress)
4. Traffic reaches EKS pods

### Pod-to-Pod Communication

1. Pod A → VPC CNI assigns IP
2. VPC CNI routes via worker node networking
3. Reaches Pod B (same or different node)

## Outputs

After successful deployment, Terraform outputs:

```bash
terraform output
```

Key outputs include:

- `vpc_id`: VPC resource ID
- `public_subnet_ids`: Public subnet IDs
- `private_subnet_ids`: Private subnet IDs
- `eks_cluster_name`: EKS cluster name
- `eks_cluster_endpoint`: Kubernetes API endpoint
- `eks_cluster_version`: Kubernetes version
- `configure_kubectl`: Command to configure kubectl

## Cost Estimation

### Monthly Costs (Production Setup)

| Resource | Cost |
|----------|------|
| EKS Cluster | $73.00 |
| NAT Gateways (2x @ $32/month) | $64.00 |
| EC2 Nodes (2x t3.medium @ $15/month) | $30.00 |
| Data Transfer | ~$5.00 |
| **Total** | **~$170-180/month** |

*Costs vary by region and data transfer amounts*

## Scaling the Cluster

### Add More Worker Nodes

Edit `terraform.tfvars`:

```hcl
desired_size = 3  # Changed from 2
max_size     = 6  # Increased from 4
```

Apply changes:

```bash
terraform apply
```

### Change Instance Type

Edit `terraform.tfvars`:

```hcl
instance_type = "t3.large"  # Larger nodes
```

## Monitoring & Logging

### Enable CloudWatch Logs

Add to `main.tf`:

```hcl
enable_cluster_autoscaling = true
cloudwatch_log_group_name   = "/aws/eks/my-eks-cluster"
```

### Check Cluster Status

```bash
aws eks describe-cluster \
  --name my-eks-cluster \
  --region us-east-1
```

## Troubleshooting

### Cannot connect to cluster

```bash
# Update kubeconfig
aws eks update-kubeconfig \
  --region us-east-1 \
  --name my-eks-cluster

# Verify connection
kubectl auth can-i get pods --all-namespaces
```

### Nodes not joining cluster

```bash
# Check node status
kubectl get nodes -o wide

# Describe node issues
kubectl describe node <node-name>
```

### Pod networking issues

```bash
# Check CNI plugin
kubectl get pods -n kube-system | grep aws-node

# Check security groups
aws ec2 describe-security-groups \
  --filters "Name=tag:Name,Values=my-eks-cluster*"
```

## Cleanup

### Destroy All Resources

```bash
terraform destroy
```

Confirm with `yes`. This will:
- Terminate EKS cluster
- Delete worker nodes
- Remove VPC and subnets
- Delete NAT gateways and EIPs
- Remove IAM roles

## Variables Reference

### Required Variables

None - all have defaults, but you should customize:

### Optional Variables

```hcl
aws_region              = "us-east-1"
environment             = "prod"
vpc_cidr                = "10.0.0.0/16"
public_subnet_cidrs     = ["10.0.1.0/24", "10.0.2.0/24"]
private_subnet_cidrs    = ["10.0.11.0/24", "10.0.12.0/24"]
cluster_name            = "my-eks-cluster"
kubernetes_version      = "1.29"
instance_type           = "t3.medium"
desired_size            = 2
min_size                = 1
max_size                = 4
```

## Best Practices

✅ **Do**:
- Use `terraform.tfvars` for customization
- Review `terraform plan` before applying
- Use separate Terraform workspaces for dev/staging/prod
- Enable CloudWatch logs for debugging
- Implement pod security policies
- Use network policies for traffic control
- Keep Kubernetes version updated
- Monitor cluster resources

❌ **Don't**:
- Modify AWS resources outside Terraform
- Use default VPC CIDR blocks in production
- Run all pods on public subnets
- Disable security groups
- Use overly permissive IAM policies
- Ignore node auto-scaling requirements

## Additional Resources

- [AWS EKS Documentation](https://docs.aws.amazon.com/eks/)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [EKS Best Practices Guide](https://aws.github.io/aws-eks-best-practices/)

## Support

For issues or questions:

1. Check [AWS EKS Troubleshooting Guide](https://docs.aws.amazon.com/eks/latest/userguide/troubleshooting.html)
2. Review Terraform logs: `TF_LOG=DEBUG terraform apply`
3. Check CloudWatch logs in AWS Console
4. Review security group rules

## License

MIT License - Feel free to use and modify

## Author

Moses EYONG (AWS Solution Architect)
Created with ☸️ and Terraform
