K8s on AWS (kOps) - Complete Project Architecture & Lifecycle
🏗️ Project Architecture Overview
text
┌─────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│   Your Browser  │───▶│ AWS NLB (API)    │───▶│ Control Plane     │
│                 │    │ api.myfirst...   │    │ t3.small          │
└─────────────────┘    │ Port 443/3988    │    │ • kube-apiserver  │
                        └──────────────────┘    │ • etcd (main)     │
                        ┌──────────────────┘    │ • etcd (events)   │
                        │ AWS Classic LB    │    └──────────────────┘
                        │ laxman-service    │           │
                        │ EXTERNAL-IP       │           │ Internal Traffic
                        └──────────────────┘           ▼
                                                      ┌──────────────────┐
                                                      │ Worker Node       │
                                                      │ t3.micro          │
                                                      │ • Nginx + HTML    │
                                                      └──────────────────┘
                           State Management
┌──────────────────┐
│ S3 Bucket        │
│ kops-laxman...   │  ← Cluster spec, SSH keys, backups
└──────────────────┘
🔄 Kubernetes Lifecycle with kOps
text
1. kops create cluster     → Generate cluster spec (S3 state)
2. kops update cluster     → PROVISION AWS resources
3. kubectl apply           → Deploy workloads
4. kubectl get/scale/edit  → Day 2 operations
5. kops delete cluster     → TEARDOWN (zero cost)
📋 Prerequisites Checklist
bash
# Verify these are ready
aws configure list          # AWS credentials
kubectl version --client    # kubectl installed
kops version               # kops 1.24+ installed
ssh-keygen -l -f ~/.ssh/id_rsa.pub  # SSH key exists
🚀 Step-by-Step Implementation
Step 1: SSH Key Setup (Node Access)
bash
ssh-keygen -t rsa -b 2048 -f ~/.ssh/id_rsa -N "" -C "k8s-cluster"
cat ~/.ssh/id_rsa.pub      # Copy this output
Purpose: Secure SSH access to EC2 instances for debugging.

Step 2: S3 State Store (Cluster Brain)
bash
aws s3 mb s3://kops-laxman-storage-1 --region us-east-1
export KOPS_STATE_STORE=s3://kops-laxman-storage-1
Purpose: GitOps-style cluster configuration storage.

Step 3: Cluster Provisioning (Infrastructure)
bash
kops create cluster \
  --name=myfirstcluster.k8s.local \
  --zones=us-east-1a \
  --node-count=1 \
  --node-size=t3.micro \
  --control-plane-size=t3.small \
  --ssh-public-key=~/.ssh/id_rsa.pub \
  --control-plane-volume-size=10 \
  --node-volume-size=10

# CRITICAL: This actually builds AWS resources
kops update cluster myfirstcluster.k8s.local --yes --admin
Wait 8-12 minutes. Check: kubectl get nodes

Generated Resources:

text
✅ VPC: 172.20.0.0/16 (public subnet)
✅ NLB: api.myfirstcluster.k8s.local (control plane)
✅ ASG: 1x t3.small master + 1x t3.micro worker
✅ IAM: Roles/profiles for nodes
✅ etcd: 2x 20GB gp3 volumes
Step 4: Deploy Portfolio (Your Application)
bash
# A. HTML Content as ConfigMap
cat <<'EOF' > portfolio.html
<!DOCTYPE html>
<html>
<head><title>Laxman Portfolio</title></head>
<body>
  <h1>🚀 Live on Kubernetes!</h1>
  <p>Deployed via kOps + AWS in us-east-1a</p>
</body>
</html>
EOF

kubectl create configmap portfolio-html --from-file=portfolio.html

# B. Deployment + LoadBalancer Service
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: laxman-portfolio
spec:
  replicas: 2
  selector:
    matchLabels:
      app: laxman-portfolio
  template:
    metadata:
      labels:
        app: laxman-portfolio
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html-volume
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html-volume
        configMap:
          name: portfolio-html
---
apiVersion: v1
kind: Service
metadata:
  name: laxman-service
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: laxman-portfolio
EOF
Step 5: Verify & Access
bash
kubectl get svc laxman-service -w
kubectl get nodes -o wide
kubectl logs deployment/laxman-portfolio
Success Output:

text
NAME             TYPE           EXTERNAL-IP     PORT(S)        AGE
laxman-service   LoadBalancer   af10cd5b...elb  80:xxxxx/TCP   2m
🌐 Access: http://af10cd5b...us-east-1.elb.amazonaws.com

Step 6: Lifecycle Management Commands
bash
# Day 2 Operations
kops get cluster                    # List clusters
kops edit ig nodes-us-east-1a       # Scale nodes: --asg-max-size=3
kubectl scale deployment/laxman-portfolio --replicas=3

# Monitoring
kubectl top nodes
kubectl get hpa,pv,pvc,ingress

# CLEANUP (IMPORTANT - stops AWS billing)
kops delete cluster --name=myfirstcluster.k8s.local --yes
aws s3 rb s3://kops-laxman-storage-1 --force
🎯 Cost Optimization
Resource	Dev Cost/mo	Production
t3.small master	~$15	t3.medium + Multi-AZ
t3.micro worker	~$8	t3.large + ASG
Total	~$23/mo	Use Spot + Savings Plans
🔍 Troubleshooting
text
Cluster not ready? → kops validate cluster
Nodes not joining? → kubectl get nodes -o wide
Service not external? → kubectl describe svc laxman-service
SSH failing? → aws ec2 describe-instances --filters "Name=tag:Name,Values=*myfirstcluster*"
📈 Scaling to Production
bash
# Multi-AZ HA cluster
kops create cluster \
  --name=prod.k8s.local \
  --zones=us-east-1a,us-east-1b,us-east-1c \
  --node-count=3 \
  --node-size=t3.large \
  --control-plane-size=t3.medium \
  --topology private \
  --networking cilium
This README now provides: Complete architecture visualization, full Kubernetes lifecycle, production scaling path, cost awareness, and troubleshooting - perfect for your DevOps portfolio! 🎉
