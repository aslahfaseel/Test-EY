# EY-Test
DevOps Assignment 
=======
# EKS Cluster with Terraform and Blue-Green Deployment

This project demonstrates a complete AWS EKS (Elastic Kubernetes Service) infrastructure setup using Terraform, featuring auto-scaling capabilities and Jenkins-based blue-green deployment strategy.

##  Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Setup Instructions](#setup-instructions)
- [Auto-Scaling Demonstration](#auto-scaling-demonstration)
- [Blue-Green Deployment](#blue-green-deployment)
- [Monitoring and Verification](#monitoring-and-verification)
- [Cleanup](#cleanup)
- [Troubleshooting](#troubleshooting)

##  Overview

This project includes:

1. **Infrastructure as Code**: Complete EKS cluster setup using Terraform
2. **Auto-Scaling**: Both cluster-level (Cluster Autoscaler) and pod-level (HPA) auto-scaling
3. **CI/CD**: Jenkins deployment with blue-green deployment pipeline
4. **Sample Application**: Nginx-based demo application for testing
5. **Monitoring**: Metrics server for resource monitoring

##  Prerequisites

Before starting, ensure you have the following installed:

1. **Terraform** (>= 1.0)
   ```bash
   terraform --version
   ```

2. **AWS CLI** (>= 2.0)
   ```bash
   aws --version
   ```

3. **kubectl** (>= 1.28)
   ```bash
   kubectl version --client
   ```

4. **AWS Account**
   - AWS account with appropriate permissions
   - Configured AWS credentials

5. **Git**
   ```bash
   git --version
   ```

### AWS Credentials Setup

Configure your AWS credentials:

```bash
aws configure
```

Or set environment variables:

```bash
export AWS_ACCESS_KEY_ID="your-access-key"
export AWS_SECRET_ACCESS_KEY="your-secret-key"
export AWS_DEFAULT_REGION="us-east-1"
```

Verify credentials:

```bash
aws sts get-caller-identity
```

## Project Structure

```
.
├── terraform/                      # Terraform infrastructure code
│   ├── provider.tf                 # Provider configurations
│   ├── variables.tf                # Variable definitions
│   ├── vpc.tf                      # VPC and networking
│   ├── iam.tf                      # IAM roles and policies
│   ├── eks.tf                      # EKS cluster and node groups
│   ├── outputs.tf                  # Output values
│   └── terraform.tfvars.example    # Example variables file
│
├── k8s-manifests/                  # Kubernetes manifests
│   ├── nginx-test/                 # Test nginx application
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── hpa.yaml                # Horizontal Pod Autoscaler
│   ├── autoscaler/                 # Auto-scaling components
│   │   ├── cluster-autoscaler.yaml # Cluster Autoscaler
│   │   └── metrics-server.yaml     # Metrics Server
│   └── load-generator/             # Load testing
│       └── load-test.yaml
│
├── jenkins/                        # Jenkins deployment
│   ├── namespace.yaml
│   ├── serviceaccount.yaml
│   ├── pvc.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── Jenkinsfile-bluegreen       # Blue-Green pipeline
│
├── blue-green-deployment/          # Blue-Green demo app
│   ├── app-deployment.yaml         # Blue and Green deployments
│   └── service.yaml                # Service configuration
│
└── README.md                       # This file
```

##  Setup Instructions

### Option 1: Automated Setup (Recommended)

Run the automated setup script:

```bash
chmod +x scripts/setup.sh

./scripts/setup.sh
```

The script will:
1. Validate prerequisites
2. Initialize and apply Terraform
3. Configure kubectl
4. Deploy all Kubernetes resources
5. Set up Jenkins and demo applications

### Option 2: Manual Setup

#### Step 1: Configure Terraform Variables

```bash
cd terraform
cp terraform.tfvars.example terraform.tfvars
```

Edit `terraform.tfvars` to customize your deployment:

```hcl
aws_region = "us-east-1"
cluster_name = "my-eks-cluster"
node_desired_size = 2
node_max_size = 5
```

#### Step 2: Deploy Infrastructure

```bash
terraform init

terraform plan

terraform apply
```

#### Step 3: Configure kubectl

```bash
CLUSTER_NAME=$(terraform output -raw cluster_name)
AWS_REGION=$(terraform output -raw region)

aws eks update-kubeconfig --region $AWS_REGION --name $CLUSTER_NAME
```

#### Step 4: Verify Cluster

```bash
kubectl get nodes
kubectl get pods -A
```

#### Step 5: Deploy Metrics Server

```bash
cd ..
kubectl apply -f k8s-manifests/autoscaler/metrics-server.yaml

# Wait for metrics server to be ready
kubectl wait --for=condition=Available deployment/metrics-server -n kube-system --timeout=300s
```

#### Step 6: Deploy Cluster Autoscaler

```bash
cd terraform
CLUSTER_NAME=$(terraform output -raw cluster_name)
CLUSTER_AUTOSCALER_ROLE_ARN=$(terraform output -raw cluster_autoscaler_role_arn)
cd ..

sed "s/\${CLUSTER_NAME}/$CLUSTER_NAME/g; s|\${CLUSTER_AUTOSCALER_ROLE_ARN}|$CLUSTER_AUTOSCALER_ROLE_ARN|g" \
    k8s-manifests/autoscaler/cluster-autoscaler.yaml | kubectl apply -f -
```

#### Step 7: Deploy Test Applications

```bash
kubectl apply -f k8s-manifests/nginx-test/

kubectl apply -f blue-green-deployment/
```

#### Step 8: Deploy Jenkins

```bash
kubectl apply -f jenkins/

kubectl wait --for=condition=Available deployment/jenkins -n jenkins --timeout=600s
```

##  Auto-Scaling Demonstration

### Horizontal Pod Autoscaling (HPA)

The nginx-test application is configured with HPA to scale based on CPU and memory usage.

#### View HPA Status

```bash
kubectl get hpa
kubectl describe hpa nginx-test-hpa
```

#### Generate Load to Trigger Scaling

```bash
kubectl apply -f k8s-manifests/load-generator/load-test.yaml

kubectl get hpa -w

kubectl get pods -w
```

#### Monitor Scaling Activity

```bash
kubectl get pods -l app=nginx-test

kubectl describe hpa nginx-test-hpa

kubectl top pods -l app=nginx-test
```

#### Stop Load Test

```bash
kubectl delete pod load-generator
```

You should see the pods scale down after 5 minutes (stabilization window).

### Cluster Autoscaling

Cluster Autoscaler automatically adjusts the number of nodes in the cluster.

#### View Cluster Autoscaler Logs

```bash
kubectl logs -f deployment/cluster-autoscaler -n kube-system
```

#### Trigger Cluster Scaling

```bash
# Scale nginx deployment beyond node capacity
kubectl scale deployment nginx-test --replicas=20

# Watch for new nodes being added
kubectl get nodes -w
```

#### Monitor Cluster Autoscaler Events

```bash
kubectl get events -n kube-system --sort-by='.lastTimestamp' | grep cluster-autoscaler
```

##  Blue-Green Deployment

### Understanding Blue-Green Deployment

Blue-Green deployment is a strategy that:
- Reduces downtime and risk
- Enables quick rollback
- Allows testing in production before switching traffic

### Access Jenkins

```bash
# Get Jenkins URL
JENKINS_URL=$(kubectl get svc jenkins -n jenkins -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "Jenkins URL: http://$JENKINS_URL:8080"

# Get initial admin password
kubectl exec -n jenkins $(kubectl get pods -n jenkins -l app=jenkins -o jsonpath='{.items[0].metadata.name}') -- cat /var/jenkins_home/secrets/initialAdminPassword
```

### Jenkins Setup

1. Open Jenkins in browser
2. Install suggested plugins
3. Create admin user
4. Install additional plugins:
   - Kubernetes Plugin
   - Pipeline Plugin
   - Git Plugin

### Configure Jenkins Pipeline

1. Create a new Pipeline job
2. Configure pipeline from SCM or paste Jenkinsfile content
3. Use the `jenkins/Jenkinsfile-bluegreen` file

### View Current Deployment

```bash
# Get demo app URL
DEMO_URL=$(kubectl get svc demo-app-service -n default -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "Demo App URL: http://$DEMO_URL"

# Check current version
curl http://$DEMO_URL
# You should see the BLUE version
```

### Manual Blue-Green Switch (Without Jenkins)

```bash
# Check current state
kubectl get deployments -l app=demo-app
kubectl get svc demo-app-service -o yaml | grep -A 3 selector

# Switch from Blue to Green
kubectl scale deployment demo-app-green --replicas=3
kubectl wait --for=condition=Available deployment/demo-app-green --timeout=300s
kubectl patch service demo-app-service -p '{"spec":{"selector":{"version":"green"}}}'

# Verify the switch
curl http://$DEMO_URL
# You should now see the GREEN version

# Scale down blue deployment
kubectl scale deployment demo-app-blue --replicas=0
```

### Execute Blue-Green Deployment via Jenkins

1. Go to Jenkins job
2. Click "Build with Parameters"
3. Select deployment direction (blue-to-green or green-to-blue)
4. Set number of replicas
5. Click "Build"
6. Monitor the pipeline execution

### Pipeline Stages

The Jenkins pipeline performs:
1. **Validate Current State**: Check existing deployments
2. **Scale Up Target**: Scale up the new version
3. **Health Checks**: Verify new deployment health
4. **Switch Traffic**: Update service selector
5. **Verification**: Monitor for 30 seconds
6. **Scale Down Old**: Scale down previous version
7. **Rollback on Failure**: Automatic rollback if issues occur

##  Monitoring and Verification

### Check Cluster Status

```bash
# Cluster info
kubectl cluster-info

# Node status
kubectl get nodes -o wide

# All pods
kubectl get pods -A

# Services
kubectl get svc -A
```

### Resource Usage

```bash
# Node resource usage
kubectl top nodes

# Pod resource usage
kubectl top pods -A
```

### View Logs

```bash
# Nginx test app logs
kubectl logs -l app=nginx-test --tail=50

# Jenkins logs
kubectl logs -l app=jenkins -n jenkins --tail=50

# Cluster autoscaler logs
kubectl logs -l app=cluster-autoscaler -n kube-system --tail=50
```

### Check Auto-Scaling Status

```bash
# HPA status
kubectl get hpa
kubectl describe hpa nginx-test-hpa

# Cluster Autoscaler status
kubectl get configmap cluster-autoscaler-status -n kube-system -o yaml
```


