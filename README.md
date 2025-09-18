# Dogs vs Cats Voting App - KRO Implementation

A cloud-native voting application built with Kubernetes Resource Orchestrator (KRO) that demonstrates modern application deployment patterns on Amazon EKS with AWS Load Balancer Controller (ALB) and AWS Controllers for Kubernetes (ACK).

## Architecture Overview

This application consists of:
- **Vote App**: Frontend for casting votes (Python/Flask)
- **Result App**: Real-time results display (Node.js)
- **Worker App**: Vote processor (Java)
- **Database**: PostgreSQL for vote storage and Redis for caching
- **Infrastructure**: ALB ingresses, ACM certificates, Route53 DNS

## Repository Structure

```
dogsvscatsapp/
├── app-instance.yaml                  # Main application instance
└── resourcegraphdefinitions/
    ├── nested-app-rg.yaml             # Top-level ResourceGraphDefinition
    └── components/
        ├── database-rg.yaml           # PostgreSQL + Redis
        ├── vote-app-rg.yaml           # Vote frontend
        ├── result-app-rg.yaml         # Results display
        ├── worker-app-rg.yaml         # Vote processor
        └── ssl-domain-rg.yaml         # ALB + ACM + Route53
```

## Prerequisites

### Required AWS Resources
- EKS cluster with OIDC provider
- VPC with public/private subnets
- Route53 hosted zone (for HTTPS/domain setup)

### Required Kubernetes Components
- [KRO (Kubernetes Resource Orchestrator)](https://kro.run)
- [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [AWS Controllers for Kubernetes (ACK)](https://aws-controllers-k8s.github.io/community/)
  - EC2 Controller
  - EKS Controller
  - IAM Controller
  - RDS Controller
  - ElastiCache Controller
  - Route53 Controller
  - ACM Controller

## Quick Start

### 1. Deploy ResourceGraphDefinitions

```bash
# Deploy all component definitions
kubectl apply -f resourcegraphdefinitions/components/

# Deploy the main nested application definition
kubectl apply -f resourcegraphdefinitions/nested-app-rg.yaml
```

### 2. Configure Your Instance

Edit `app-instance.yaml` with your AWS-specific values:

```yaml
apiVersion: kro.run/v1alpha1
kind: DogsvsCatsApp
metadata:
  name: dogsvscats-voting-app
  namespace: default
spec:
  name: dogsvscats-voting-app
  namespace: default
  
  # Your AWS Infrastructure
  vpcID: "vpc-xxxxxxxxx"
  subnetIDs: ["subnet-xxxxxxxx", "subnet-yyyyyyyy", "subnet-zzzzzzzz"]
  
  # Domain Configuration (optional)
  domain:
    enabled: true
    hostedZoneId: "ZXXXXXXXXXXXXX"
    baseDomain: "yourdomain.com"
  
  # Feature Flags
  acm:
    enabled: true    # Set to false for HTTP-only
  ingress:
    enabled: true
  service:
    enabled: true
  
  # Application Images
  voteImage: "dockersamples/examplevotingapp_vote"
  resultImage: "dockersamples/examplevotingapp_result"
  workerImage: "dockersamples/examplevotingapp_worker"
```

### 3. Deploy the Application

```bash
kubectl apply -f app-instance.yaml
```

## Monitoring Deployment

### Check Overall Status

```bash
kubectl get dogsvscatsapp dogsvscats-voting-app -o yaml
```

### Monitor Individual Components

```bash
# Check all related resources
kubectl get votingdatabase,voteapp,resultapp,workerapp,ssldomain

# Check pods
kubectl get pods -l app.kubernetes.io/part-of=dogsvscats-voting-app

# Check services and ingresses
kubectl get svc,ingress -l app.kubernetes.io/part-of=dogsvscats-voting-app
```

### Check ALB Status

```bash
# View ALB ingresses
kubectl get ingress

# Check ALB controller logs
kubectl logs -n kube-system deployment/aws-load-balancer-controller
```

## Troubleshooting

### Common Issues

#### 1. ACM Certificate Stuck in "Pending Validation"

```bash
# Check Route53 validation records
kubectl get recordset

# Verify hosted zone configuration
aws route53 list-hosted-zones
```

#### 2. Database Connection Issues

```bash
# Check RDS instance status
kubectl get dbinstance

# Check security groups allow traffic on port 5432/6379
kubectl describe dbinstance
kubectl describe serverlesscache
```

#### 3. Pods Not Starting

```bash
# Check pod logs
kubectl logs -l app=dogsvscats-voting-app-vote
kubectl logs -l app=dogsvscats-voting-app-result
kubectl logs -l app=dogsvscats-voting-app-worker

# Check resource quotas
kubectl describe resourcequota
```

### Debug Commands

```bash
# Get detailed status of main resource
kubectl describe dogsvscatsapp dogsvscats-voting-app

# Check KRO controller logs
kubectl logs -n kro-system deployment/kro-controller-manager

# Check ACK controller logs
kubectl logs -n ack-system deployment/ack-ec2-controller
kubectl logs -n ack-system deployment/ack-rds-controller
```

## Cleanup

```bash
# Delete the application instance
kubectl delete -f app-instance.yaml

# Wait for resources to be cleaned up, then remove definitions
kubectl delete -f resourcegraphdefinitions/
```

## Advanced Configuration

### Custom Resource Sizing

```yaml
spec:
  voteReplicas: 3
  resultReplicas: 2
  workerReplicas: 2
```

### Custom Images

```yaml
spec:
  voteImage: "your-registry/vote:v1.0"
  resultImage: "your-registry/result:v1.0"
  workerImage: "your-registry/worker:v1.0"
```

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test with a real EKS cluster
5. Submit a pull request

