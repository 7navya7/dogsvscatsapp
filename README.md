# Dogs vs Cats Voting App Demo

A complete Kubernetes-native voting application demo using KRO (Kubernetes Resource Orchestrator) and ACK (AWS Controllers for Kubernetes) that deploys on existing EKS clusters.

## Overview

This demo showcases how to deploy a full-stack voting application with AWS infrastructure using only Kubernetes manifests - no Terraform or shell scripts required!

### What Gets Deployed

- **Vote App**: Frontend for casting votes (`vote.dogsvscats.us`)
- **Result App**: Real-time results display (`result.dogsvscats.us`) 
- **Worker App**: Background vote processor
- **PostgreSQL**: RDS database for vote storage
- **Redis**: ElastiCache for vote queuing
- **SSL Certificate**: ACM certificate with automatic validation
- **DNS Records**: Route53 records for vote and result domains
- **Load Balancer**: ALB with SSL termination

## Architecture

```
Internet
    │
    ├── vote.dogsvscats.us ──┐
    └── result.dogsvscats.us ─┼── ALB (SSL) ──┐
                              │               │
                              │               ├── Vote Service
                              │               └── Result Service
                              │                       │
                              └── Worker ─────────────┤
                                     │                │
                              ┌──────┴──────┐        │
                              │    Redis    │        │
                              │ (ElastiCache)│        │
                              └─────────────┘        │
                                     │                │
                              ┌──────┴──────┐        │
                              │ PostgreSQL  │────────┘
                              │    (RDS)    │
                              └─────────────┘
```

## Prerequisites

- ✅ Existing EKS cluster (1.28+)
- ✅ kubectl configured for your cluster
- ✅ Route53 hosted zone for `dogsvscats.us`
- ✅ Required ACK controllers installed:
  - ACM Controller
  - Route53 Controller  
  - EC2 Controller
  - RDS Controller
  - ElastiCache Controller
- ✅ KRO (Kubernetes Resource Orchestrator) installed
- ✅ AWS Load Balancer Controller installed

## Quick Start

### Step 1: Configure Your Infrastructure

Edit `voting-app-config.yaml` with your AWS infrastructure details:

```yaml
data:
  # Your Route53 hosted zone ID
  hostedZoneId: "Z0362187GFK6MIGB8GOB"
  
  # Your domain (must match your hosted zone)
  baseDomain: "dogsvscats.us"
  
  # Your EKS cluster's VPC ID
  vpcId: "vpc-0123456789abcdef0"
  
  # Your EKS cluster's subnet IDs (comma-separated)
  subnetIds: "subnet-0123456789abcdef0,subnet-0fedcba9876543210"
  
  # AWS region
  awsRegion: "us-west-2"
  
  # Your EKS cluster name
  clusterName: "my-cluster"
```

### Step 2: Deploy the Application

```bash
# Apply the configuration
kubectl apply -f voting-app-config.yaml

# Deploy the ResourceGraph definition
kubectl apply -f voting-app-rg.yaml

# Deploy the VotingApp instance
kubectl apply -f voting-app-instance.yaml
```

### Step 3: Monitor Deployment

```bash
# Watch the VotingApp status
kubectl get votingapp dogsvscats-voting-app -w

# Check all created resources
kubectl get certificate,recordset,dbinstance,securitygroup,deployment,service,ingress

# Monitor KRO controller logs
kubectl logs -l app.kubernetes.io/name=kro-controller-manager -n kro-system -f
```

### Step 4: Access the Application

Once deployed (5-10 minutes), access your voting app:

- **Vote**: https://vote.dogsvscats.us
- **Results**: https://result.dogsvscats.us

## Files Description

- `README.md` - This documentation
- `voting-app-config.yaml` - Customer configuration (edit this!)
- `voting-app-rg.yaml` - ResourceGraph definition (KRO template)
- `voting-app-instance.yaml` - VotingApp instance (triggers deployment)

## Configuration Details

### Required Configuration

Update these values in `voting-app-config.yaml`:

| Field | Description | Example |
|-------|-------------|---------|
| `hostedZoneId` | Route53 hosted zone ID | `Z0362187GFK6MIGB8GOB` |
| `baseDomain` | Your domain name | `dogsvscats.us` |
| `vpcId` | EKS cluster VPC ID | `vpc-0123456789abcdef0` |
| `subnetIds` | Comma-separated subnet IDs | `subnet-abc,subnet-def` |
| `awsRegion` | AWS region | `us-west-2` |
| `clusterName` | EKS cluster name | `my-cluster` |

### Optional Configuration

These have sensible defaults but can be customized:

| Field | Default | Description |
|-------|---------|-------------|
| `voteImage` | `dockersamples/examplevotingapp_vote` | Vote app container image |
| `resultImage` | `dockersamples/examplevotingapp_result` | Result app container image |
| `workerImage` | `dockersamples/examplevotingapp_worker` | Worker container image |
| `dbInstanceClass` | `db.t3.micro` | RDS instance type |
| `cacheNodeType` | `cache.t3.micro` | ElastiCache node type |

## How It Works

### 1. ConfigMap-Driven Configuration
Instead of hardcoded values, all infrastructure details are read from a ConfigMap that customers edit.

### 2. KRO Resource Orchestration
KRO reads the ResourceGraph definition and creates all AWS resources in the correct order with proper dependencies.

### 3. ACK Controllers
AWS Controllers for Kubernetes manage the actual AWS resources (RDS, ElastiCache, ACM, Route53) as Kubernetes resources.

### 4. Automatic DNS & SSL
- ACM certificate is automatically created and validated
- Route53 records point to the ALB
- SSL termination at the load balancer

## Troubleshooting

### Common Issues

**VotingApp stuck in pending state:**
```bash
# Check KRO controller logs
kubectl logs -l app.kubernetes.io/name=kro-controller-manager -n kro-system

# Check individual resource status
kubectl describe certificate dogsvscats-certificate
kubectl describe dbinstance dogsvscats-postgres
```

**DNS not resolving:**
```bash
# Check Route53 records were created
kubectl get recordset

# Test DNS resolution
dig vote.dogsvscats.us
dig result.dogsvscats.us
```

**SSL certificate issues:**
```bash
# Check certificate status
kubectl get certificate -o yaml

# Check validation records
kubectl get recordset -o yaml | grep validation
```

**Database connection issues:**
```bash
# Check RDS instance status
kubectl get dbinstance dogsvscats-postgres -o yaml

# Check security group rules
kubectl get securitygroup -o yaml
```

### Useful Commands

```bash
# Get overall status
kubectl get votingapp dogsvscats-voting-app -o yaml

# List all created AWS resources
kubectl get certificate,recordset,dbinstance,cachecluster,securitygroup

# Check application pods
kubectl get pods -l app.kubernetes.io/part-of=dogsvscats-voting-app

# View application logs
kubectl logs -l app=vote
kubectl logs -l app=result
kubectl logs -l app=worker

# Check ingress status
kubectl get ingress -o wide

# Monitor ALB creation
kubectl describe ingress vote-ingress
kubectl describe ingress result-ingress
```

## Cleanup

To remove all resources:

```bash
# Delete the VotingApp instance (this triggers cleanup of all AWS resources)
kubectl delete votingapp dogsvscats-voting-app

# Wait for cleanup to complete (may take 5-10 minutes)
kubectl get certificate,recordset,dbinstance,cachecluster,securitygroup

# Optional: Remove the ResourceGraph definition
kubectl delete resourcegraphdefinition voting-app.kro.run

# Optional: Remove the ConfigMap
kubectl delete configmap voting-app-config
```

## Demo Script

For presenting this demo:

### Setup (2 minutes)
1. Show the ConfigMap with customer values
2. Explain the three-file approach

### Deploy (1 minute)
```bash
kubectl apply -f voting-app-config.yaml
kubectl apply -f voting-app-rg.yaml  
kubectl apply -f voting-app-instance.yaml
```

### Monitor (5-8 minutes)
```bash
# Watch resources being created
kubectl get votingapp dogsvscats-voting-app -w

# Show AWS resources appearing
watch "kubectl get certificate,recordset,dbinstance,securitygroup"
```

### Demo (5 minutes)
1. Visit https://vote.dogsvscats.us
2. Cast some votes
3. Visit https://result.dogsvscats.us
4. Show real-time results

### Explain (10 minutes)
1. Show created AWS resources in console
2. Explain KRO orchestration
3. Show ACK controller management
4. Discuss benefits vs Terraform/CloudFormation

### Cleanup (1 minute)
```bash
kubectl delete votingapp dogsvscats-voting-app
```

## Benefits of This Approach

- ✅ **Kubernetes-Native**: Everything managed through kubectl
- ✅ **No Scripts**: Pure YAML configuration
- ✅ **Declarative**: Desired state management
- ✅ **Portable**: Works on any EKS cluster
- ✅ **GitOps Ready**: All configuration in version control
- ✅ **Clean Separation**: Infrastructure config vs application logic
- ✅ **Dependency Management**: KRO handles resource ordering
- ✅ **Automatic Cleanup**: Delete one resource, cleanup everything

## Support

- **KRO Documentation**: https://kro.run
- **ACK Documentation**: https://aws-controllers-k8s.github.io/community/
- **AWS Load Balancer Controller**: https://kubernetes-sigs.github.io/aws-load-balancer-controller/