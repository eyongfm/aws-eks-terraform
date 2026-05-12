# Network Flow Diagrams

## Overview

This document explains the network data flows in the AWS EKS infrastructure.

---

## 1. Outbound Traffic Flow (Pod → Internet)

```
┌─────────────────────────────────────────────────────────────┐
│                     Pod in Container                         │
│                    IP: 10.1.x.x/32                           │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────┐
│              VPC CNI (Container Network Interface)           │
│          Routes pod traffic via worker node networking       │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────┐
│              Worker Node (EC2 Instance)                      │
│              IP: 10.0.11.x / 10.0.12.x                      │
│          (Primary Elastic Network Interface)                 │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────┐
│              Private Subnet Route Table                       │
│         Route: 0.0.0.0/0 → NAT Gateway                       │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────┐
│                  NAT Gateway                                  │
│  (Translates private IPs to Elastic IP)                     │
│  IP: <Elastic IP>                                            │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────┐
│              Public Subnet Route Table                        │
│         Route: 0.0.0.0/0 → Internet Gateway                 │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────┐
│             Internet Gateway                                  │
│    (Translates AWS IPs to internet-routable addresses)       │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────┐
│                   Internet                                    │
│                 0.0.0.0/0                                    │
└─────────────────────────────────────────────────────────────┘
```

### Key Points:

- ✅ NAT Gateway provides security by masking private IPs
- ✅ Enables outbound access without exposing worker nodes
- ✅ Multiple NAT Gateways (one per AZ) for HA
- ✅ Data transfer charges apply

---

## 2. Inbound Traffic Flow (Internet → Pod)

```
┌─────────────────────────────────────────────────────────────┐
│                   Internet                                    │
│                 0.0.0.0/0                                    │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────┐
│             Internet Gateway                                  │
│   (Routes to EKS Load Balancer or Ingress Controller)        │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────┐
│              Public Subnet (Load Balancer)                    │
│              IP: 10.0.1.x / 10.0.2.x                         │
│         (AWS Network Load Balancer or ALB)                   │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────┐
│          EKS Cluster Security Group (Inbound)                │
│    Allows traffic from load balancer to worker nodes         │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────┐
│          Private Subnet (Worker Node)                         │
│              IP: 10.0.11.x / 10.0.12.x                       │
│                Kube-proxy (iptables)                          │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────┐
│                 Service (ClusterIP)                           │
│                   Virtual IP                                  │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ↓
┌────────────────────────────��────────────────────────────────┐
│                   Pod Container                               │
│                 Running Application                           │
└─────────────────────────────────────────────────────────────┘
```

### Key Components:

1. **Internet Gateway**: Routes internet traffic into VPC
2. **Load Balancer**: Distributes traffic across service endpoints
3. **Security Groups**: Control inbound rules
4. **Service**: Kubernetes abstraction (ClusterIP/NodePort/LoadBalancer)
5. **Pod**: Application container

---

## 3. Pod-to-Pod Communication (Same Cluster)

### Same Node

```
Pod A (10.1.0.5)  ──→  Worker Node (10.0.11.10)  ←──  Pod B (10.1.0.6)
                        │
                   VPC CNI Plugin
                   (Direct bridge)
                        │
                   Linux veth pairs
                   (Virtual Ethernet)
```

### Different Nodes (Same AZ)

```
Pod A (10.1.0.5)                         Pod B (10.1.0.15)
      │                                        │
      ↓                                        ↓
Worker Node 1                           Worker Node 2
(10.0.11.10)                            (10.0.11.20)
      │                                        │
      └────────────────────┬────────────────┘
                           │
                      VPC Network
                   (AWS Managed)
                           │
                   Route: 10.1.0.0/16
                      (Direct)
```

### Different Nodes (Different AZ)

```
Pod A (10.1.0.5)                         Pod B (10.2.0.15)
      │                                        │
      ↓                                        ↓
Worker Node 1                           Worker Node 3
(10.0.11.10)                            (10.0.12.20)
AZ: us-east-1a                          AZ: us-east-1b
      │                                        │
      └────────────────────┬────────────────┘
                           │
                      VPC Network
                   (Cross-AZ traffic)
                           │
                   Route: 10.2.0.0/16
                   (AWS backbone)
```

### Key Points:

- ✅ VPC CNI handles IP assignment per pod
- ✅ Direct AWS routing (no NAT for pod-to-pod)
- ✅ Cross-AZ traffic charged but low latency
- ✅ Network policies can restrict traffic

---

## 4. Pod-to-AWS Services

```
┌──────────────────────────────────────────────────────────────┐
│                    EKS Pod                                    │
│                 (Application)                                 │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ↓ (DynamoDB, S3, etc.)
         ┌─────────────────────────────┐
         │  VPC Endpoint (Optional)    │
         │  PrivateLink Gateway        │
         └──────────┬──────────────────┘
                    │
        ┌───────────┴───────────┐
        │                       │
        ↓                       ↓
    Option 1:             Option 2:
  NAT Gateway         VPC Endpoint
  (Via Internet)      (Direct AWS)  ← Recommended
        │                    │
        └────────┬───────────┘
                 │
                 ↓
      AWS Service (S3, DynamoDB, etc.)
```

### VPC Endpoint Setup

```hcl
# S3 Gateway Endpoint (no charge)
resource "aws_vpc_endpoint" "s3" {
  vpc_id       = aws_vpc.main.id
  service_name = "com.amazonaws.us-east-1.s3"
  route_table_ids = [aws_route_table.private[*].id]
}

# DynamoDB Interface Endpoint (charged)
resource "aws_vpc_endpoint" "dynamodb" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.us-east-1.dynamodb"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
}
```

---

## 5. Pod-to-External Database

```
┌────────────────────────────────────────────────────────────┐
│                   EKS Pod                                   │
│              Connection to RDS DB                           │
└─────────────────────────┬────────────────────────────────┘
                          │
                          ↓
            ┌──────────────────────────┐
            │  Private Subnet          │
            │  Worker Node Security    │
            │  Group (Egress)          │
            └──────────────┬───────────┘
                          │
                          ↓
        ┌─────────────────────────────┐
        │  RDS Security Group         │
        │  (Inbound: MySQL 3306)      │
        │  Source: Worker Node SG     │
        └──────────────┬──────────────┘
                       │
                       ↓
           ┌───────────────────────┐
           │  RDS Database         │
           │  (Managed by AWS)      │
           │  Multi-AZ (Optional)   │
           └───────────────────────┘
```

### Configuration Example

```hcl
# RDS Security Group
resource "aws_security_group" "rds" {
  name   = "rds-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [aws_security_group.eks_cluster.id]
  }
}
```

---

## 6. Kubernetes Control Plane Communication

```
┌──────────────────────────────────────────────────────────┐
│           Worker Node (EC2)                              │
│  ┌─────────────────────────────────────────────────┐    │
│  │  kubelet (Node Agent)                           │    │
│  │  kube-proxy (Service Networking)                │    │
│  │  Container Runtime (Docker/containerd)          │    │
│  └────────────────────┬────────────────────────────┘    │
│                       │                                  │
│                       ↓ (HTTPS :443)                      │
│  ┌─────────────────────────────────────────────────┐    │
│  │  Security Group (Outbound Rule)                 │    │
│  │  Allow: 0.0.0.0/0 :443                          │    │
│  └────────────────────┬────────────────────────────┘    │
└───────────────────────┼──────────────────────────────────┘
                        │
                        ↓
    ┌───────────────────────────────────────────┐
    │  EKS Control Plane (AWS Managed)          │
    │                                            │
    │  API Server (Port 443)                    │
    │  ├─ Kubernetes API                        │
    │  ├─ Resource Management                   │
    │  └─ Cluster Operations                    │
    │                                            │
    │  Controller Manager                       │
    │  ├─ Node Controller                       │
    │  ├─ Pod Controller                        │
    │  └─ Service Controller                    │
    │                                            │
    │  Scheduler                                │
    │  └─ Pod Placement                         │
    │                                            │
    │  etcd Database                            │
    │  └─ Cluster State                         │
    └───────────────────────────────────────────┘
```

### Communication Details

- **Protocol**: HTTPS/TLS
- **Port**: 443 (Standard)
- **Frequency**: Continuous (heartbeat, events)
- **Throughput**: Low to medium
- **Security**: Certificate-based authentication

---

## 7. Local Development to Cluster

```
┌──────────────────────────────────────────────────────────┐
│  Developer Laptop (Local)                                │
│  ├─ kubectl CLI                                          │
│  ├─ AWS CLI                                              │
│  └─ kubeconfig (~/.kube/config)                          │
└──────────────┬───────────────────────────────────────────┘
               │
               ↓ (AWS SigV4 + TLS)
       ┌───────────────────┐
       │  Internet         │
       │  (Encrypted)      │
       └───────┬───────────┘
               │
               ↓
       ┌───────────────────────────────┐
       │  EKS Control Plane Endpoint   │
       │  (Public HTTPS Endpoint)      │
       │  https://xyz.eks.amazonaws.com│
       └───────┬───────────────────────┘
               │
               ↓
       ┌───────────────────────────────┐
       │  API Server                   │
       │  └─ Processes kubectl requests│
       └───────────────────────────────┘
```

### Endpoint Options

```hcl
# Public Endpoint (Default)
vpc_config {
  endpoint_public_access  = true   # Accessible from internet
  endpoint_private_access = false  # Not accessible from VPC
}

# Private Endpoint (Secure)
vpc_config {
  endpoint_public_access  = false  # Not accessible from internet
  endpoint_private_access = true   # Only accessible from VPC
}

# Both (Recommended)
vpc_config {
  endpoint_public_access  = true   # Public for remote access
  endpoint_private_access = true   # Private for VPC access
}
```

---

## Summary: Network Flow Decision Tree

```
Which traffic flow?
│
├─→ Pod → Internet
│   └─→ VPC CNI → Worker Node → Private Route Table
│       → NAT Gateway → IGW → Internet
│
├─→ Internet → Pod
│   └─→ IGW → Load Balancer → Service → Pod
│
├─→ Pod → Pod (Same Cluster)
│   └─→ VPC CNI → Direct routing (no NAT)
│
├─→ Pod → AWS Service
│   ├─→ Option 1: Via NAT → IGW → Service
│   └─→ Option 2: Via VPC Endpoint (Recommended)
│
├─→ Pod → External Database
│   └─→ Security Group Rules → Database
│
└─→ kubectl → Cluster
    └─→ HTTPS → EKS Endpoint → API Server
```

---

## Troubleshooting Network Issues

### Pod cannot reach internet

```bash
# Check NAT Gateway status
aws ec2 describe-nat-gateways --region us-east-1

# Check route tables
aws ec2 describe-route-tables --region us-east-1

# Check security group egress rules
aws ec2 describe-security-groups --region us-east-1
```

### Pod cannot reach other pod

```bash
# Check VPC CNI
kubectl get pods -n kube-system -l k8s-app=aws-node

# Check network policies
kubectl get networkpolicies -A

# Test connectivity
kubectl exec -it <pod> -- ping <other-pod-ip>
```

### Slow internet access

```bash
# Check NAT Gateway utilization
aws cloudwatch get-metric-statistics \
  --namespace AWS/NatGateway \
  --metric-name ProcessedBytes \
  --region us-east-1

# Consider adding more NAT Gateways or upgrading node size
```

---
