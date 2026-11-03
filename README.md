# Brain Tasks App — Full AWS DevOps Deployment Guide

## Project Overview
This repository contains the production build of the **Brain Tasks** React application, fully containerized and deployed to **AWS EKS** via **AWS CodePipeline + CodeBuild**.

---

## Repository Structure

```
Brain-Tasks-App/
├── dist/                   # Production React build output
│   ├── index.html
│   └── assets/
├── k8s/
│   ├── deployment.yaml     # Kubernetes Deployment manifest
│   └── service.yaml        # Kubernetes LoadBalancer Service manifest
├── Dockerfile              # Multi-stage Docker build (nginx:alpine)
├── nginx.conf              # Nginx config (serves on port 3000)
├── .dockerignore
├── buildspec.yml           # AWS CodeBuild build specification
└── README.md               # This file
```

---

## Pipeline Architecture

```
GitHub Repo
    │
    ▼
AWS CodePipeline
    ├── Stage 1: Source  ──► GitHub (main branch)
    ├── Stage 2: Build   ──► AWS CodeBuild
    │       ├── Login to ECR
    │       ├── docker build + tag
    │       ├── docker push to ECR
    │       └── kubectl apply to EKS
    └── Stage 3: (Optional) Manual Approval
```

---

## Prerequisites
- AWS CLI v2 installed and configured (`aws configure`)
- Docker Desktop installed and running
- `kubectl` installed
- `eksctl` installed
- An AWS account with permissions for: ECR, EKS, CodeBuild, CodePipeline, IAM, CloudWatch

---

## Step-by-Step Setup

### STEP 1 — Docker: Build & Test Locally

```bash
# Build the Docker image
docker build -t brain-tasks-app:latest .

# Run locally on port 3000
docker run -d -p 3000:3000 --name brain-tasks brain-tasks-app:latest

# Test it
curl http://localhost:3000/health
# Open browser: http://localhost:3000

# Stop container
docker stop brain-tasks && docker rm brain-tasks
```

---

### STEP 2 — AWS ECR: Create Repository & Push Image

```bash
# Set your values
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
AWS_REGION=us-east-1
ECR_REPO=brain-tasks-app
ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

# Create ECR repository
aws ecr create-repository \
  --repository-name $ECR_REPO \
  --region $AWS_REGION \
  --image-scanning-configuration scanOnPush=true

# Login to ECR
aws ecr get-login-password --region $AWS_REGION \
  | docker login --username AWS --password-stdin $ECR_REGISTRY

# Tag and push
docker tag brain-tasks-app:latest $ECR_REGISTRY/$ECR_REPO:latest
docker push $ECR_REGISTRY/$ECR_REPO:latest

echo "Image pushed: $ECR_REGISTRY/$ECR_REPO:latest"
```

---

### STEP 3 — AWS EKS: Create Cluster

```bash
# Create EKS cluster (takes ~15 minutes)
eksctl create cluster \
  --name brain-tasks-cluster \
  --region us-east-1 \
  --nodegroup-name brain-tasks-nodes \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed

# Verify cluster is running
aws eks describe-cluster --name brain-tasks-cluster --region us-east-1 \
  --query 'cluster.status'

# Update kubeconfig
aws eks update-kubeconfig --name brain-tasks-cluster --region us-east-1

# Verify nodes
kubectl get nodes
```

---

### STEP 4 — Update deployment.yaml with Your ECR Image

Edit `k8s/deployment.yaml` and replace the placeholder:
```
<AWS_ACCOUNT_ID>.dkr.ecr.<AWS_REGION>.amazonaws.com/brain-tasks-app:latest
```
with your actual ECR image URI, e.g.:
```
123456789012.dkr.ecr.us-east-1.amazonaws.com/brain-tasks-app:latest
```

---

### STEP 5 — Deploy to EKS Manually (optional test)

```bash
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml

# Watch rollout
kubectl rollout status deployment/brain-tasks-app

# Get LoadBalancer URL (wait ~2-3 min for AWS to provision it)
kubectl get svc brain-tasks-app-service
```

The `EXTERNAL-IP` / hostname in the output is your **LoadBalancer ARN/DNS**.

---

### STEP 6 — IAM Role for CodeBuild

Create an IAM policy and attach to CodeBuild role:

```bash
# Create policy file
cat > codebuild-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:PutImage"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "eks:DescribeCluster",
        "eks:ListClusters"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    }
  ]
}
EOF

# Create and attach policy to CodeBuild role
aws iam create-policy \
  --policy-name CodeBuildECR-EKSPolicy \
  --policy-document file://codebuild-policy.json
```

Also add CodeBuild's role to EKS `aws-auth` ConfigMap:
```bash
# Get CodeBuild role ARN
CODEBUILD_ROLE_ARN=$(aws iam get-role --role-name codebuild-brain-tasks-service-role --query Role.Arn --output text)

# Edit aws-auth
kubectl edit configmap aws-auth -n kube-system
# Add under mapRoles:
# - rolearn: <CODEBUILD_ROLE_ARN>
#   username: codebuild
#   groups:
#     - system:masters
```

---

### STEP 7 — AWS CodeBuild Project

Create via AWS Console or CLI:

```bash
aws codebuild create-project \
  --name brain-tasks-build \
  --source '{"type":"GITHUB","location":"https://github.com/karthikeyan141120/Brain-task-app.git","buildspec":"buildspec.yml"}' \
  --artifacts '{"type":"NO_ARTIFACTS"}' \
  --environment '{
    "type":"LINUX_CONTAINER",
    "image":"aws/codebuild/standard:7.0",
    "computeType":"BUILD_GENERAL1_SMALL",
    "privilegedMode":true,
    "environmentVariables":[
      {"name":"AWS_REGION","value":"us-east-1"},
      {"name":"EKS_CLUSTER_NAME","value":"brain-tasks-cluster"},
      {"name":"ECR_REPO_NAME","value":"brain-tasks-app"}
    ]
  }' \
  --service-role "arn:aws:iam::<ACCOUNT_ID>:role/codebuild-brain-tasks-service-role" \
  --region us-east-1
```

---

### STEP 8 — AWS CodePipeline

Create via AWS Console:
1. **Pipeline name**: `brain-tasks-pipeline`
2. **Source Stage**:
   - Provider: GitHub (Version 2)
   - Repository: `karthikeyan141120/Brain-task-app`
   - Branch: `main`
3. **Build Stage**:
   - Provider: AWS CodeBuild
   - Project name: `brain-tasks-build`
4. **Deploy Stage**: Skip (CodeBuild handles kubectl deploy)

---

### STEP 9 — CloudWatch Monitoring

```bash
# View CodeBuild logs
aws logs describe-log-groups --log-group-name-prefix "/aws/codebuild/brain-tasks"

# Stream live logs
aws logs tail /aws/codebuild/brain-tasks-build --follow

# View application logs from EKS pod
kubectl logs -l app=brain-tasks-app --tail=50 -f
```

Set up a CloudWatch Alarm for build failures:
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "CodeBuild-brain-tasks-failure" \
  --metric-name FailedBuilds \
  --namespace AWS/CodeBuild \
  --statistic Sum \
  --period 300 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 1 \
  --dimensions Name=ProjectName,Value=brain-tasks-build \
  --alarm-actions <YOUR_SNS_TOPIC_ARN>
```

---

## Verifying the Deployment

```bash
# Check pods are running
kubectl get pods -l app=brain-tasks-app

# Get LoadBalancer DNS (this is your submission ARN/URL)
kubectl get svc brain-tasks-app-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'

# Test the app
curl http://<LOADBALANCER_DNS>/health
```

---

## Submission Checklist

- [ ] GitHub repository link: `https://github.com/karthikeyan141120/Brain-task-app`
- [ ] Docker image in ECR: `<account>.dkr.ecr.us-east-1.amazonaws.com/brain-tasks-app:latest`
- [ ] EKS cluster running: `brain-tasks-cluster`
- [ ] Application accessible via LoadBalancer DNS on port 80
- [ ] CodePipeline configured with GitHub → CodeBuild → EKS
- [ ] CloudWatch logs enabled
- [ ] LoadBalancer DNS/ARN captured and submitted

---

## Screenshots Required for Submission
1. ECR repository showing pushed image
2. EKS cluster nodes (`kubectl get nodes`)
3. Running pods (`kubectl get pods`)
4. LoadBalancer service (`kubectl get svc`)
5. CodeBuild build success log
6. CodePipeline successful run
7. Application running in browser via LoadBalancer URL
8. CloudWatch log group for CodeBuild
