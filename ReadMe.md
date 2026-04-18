🚀 Ultimate Kubernetes Mastery Guide: Local → Cloud Portfolio
Laxman DevOps Engineer - Multi-environment K8s with CIDR Networking, Multi-Domain Ingress, Chaos Engineering.

📁 Project Structure
text
Laxman-k8s-portfolio-project/
├── 📁 config/                 # Production YAMLs
│   ├── portfolio-complete.yaml  # All-in-one deployment
│   ├── ingress-multi-domain.yaml
│   └── poornima-site.yml
├── 📁 docs/
│   └── k8s-mastery-guide.pdf   # This document!
├── 📄 index.html              # Custom HTML
├── 📄 poornima-index.html     # Multi-tenant site
├── 📄 README.md               # Quick start
└── 🐳 minikube-linux-amd64    # Minikube binary
Current Files Organized:

text
✅ configmap.yml     → config/portfolio-complete.yaml
✅ deployment.yml    → config/portfolio-complete.yaml  
✅ service.yml       → config/portfolio-complete.yaml
✅ ingress.yml       → config/ingress-multi-domain.yaml
✅ pod.yml           → (learning only)
✅ poornima-site.yml → config/multi-tenant/
🌐 #0 CIDR Networking Fundamentals
What is CIDR?
text
172.20.0.0/16  → 65,536 IP addresses (kOps VPC)
└── 172.20.1.0/24 → 256 IPs (Public Subnet)
    └── 172.20.1.10 → Control Plane
        └── 172.20.1.11 → Worker Node
kOps Auto-generates:

text
VPC: 172.20.0.0/16
Public Subnet: 172.20.128.0/17 (us-east-1a)
Route: 0.0.0.0/0 → Internet Gateway
Minikube Pod CIDR: 10.244.0.0/16 (Flannel/Kindnet)

🖥️ #1 Local Multi-Node Minikube (KVM2)
bash
minikube start --driver=kvm2 --nodes=2 --pods=100 --service-cluster-ip-range=10.96.0.0/12
#                                    ↑ Custom Service CIDR
☁️ #2 Production AWS kOps (CIDR Config)
bash
kops create cluster myfirstcluster.k8s.local \
  --zones=us-east-1a \
  --vpc=vpc-12345678 \                    # Custom VPC CIDR
  --networking=cilium \                   # CNI with eBPF
  --service-cluster-ip-range=10.100.0.0/16 # Custom Service IPs
Network Flow:

text
Internet (0.0.0.0/0) → NLB (Public IP) → 172.20.1.10:443 (API)
                                                    ↓
                                              Service: 10.100.1.200:80
                                                    ↓
                                              Pod: 10.244.1.5:8080 (nginx)
🌐 #3 Multi-Domain Single IP (Ingress Magic)
One LoadBalancer IP → Multiple Domains!

config/ingress-multi-domain.yaml
text
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-tenant-ingress
  annotations:
    # Minikube
    kubernetes.io/ingress.class: nginx
    # kOps AWS ALB  
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
  # Domain 1: Personal Portfolio
  - host: portfolio.laxman.dev
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: laxman-portfolio
            port: {number: 80}
  
  # Domain 2: Poornima Site (Multi-tenant)
  - host: poornima.laxman.dev
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: poornima-site
            port: {number: 80}
  
  # Domain 3: API
  - host: api.laxman.dev
    http:
      paths:
      - path: /health
        pathType: Prefix
        backend:
          service:
            name: portfolio-service
            port: {number: 80}
Deploy Multi-Tenant
bash
# Poornima site (same structure, different ConfigMap)
kubectl apply -f config/poornima-site.yml
kubectl apply -f config/ingress-multi-domain.yaml

# Test routing
curl -H "Host: portfolio.laxman.dev" $(minikube ip)
curl -H "Host: poornima.laxman.dev" $(minikube ip)
How It Works:

text
Single IP (A Record) → Ingress Controller → Host Header Routing
portfolio.laxman.dev → laxman-portfolio service
poornima.laxman.dev  → poornima-site service
🛠️ #4 Universal Portfolio (Single File)
config/portfolio-complete.yaml - All services in ONE file:

text
# COMPLETE PRODUCTION STACK - 3 Services, Multi-Domain Ready
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: laxman-html
data:
  index.html: |
    <h1>🚀 Laxman Portfolio - $(kubectl get nodes | wc -l) Nodes</h1>
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: poornima-html
data:
  index.html: |
    <h1>🌸 Poornima Site - Multi-Tenant Demo</h1>
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: laxman-portfolio
spec:
  replicas: 3
  selector: {matchLabels: {app: laxman}}
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        volumeMounts: [{name: html, mountPath: /usr/share/nginx/html}]
        resources: {requests: {cpu: "100m", memory: "128Mi"}}
      volumes: [{name: html, configMap: {name: laxman-html}}]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: poornima-site
spec:
  replicas: 2
  selector: {matchLabels: {app: poornima}}
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        volumeMounts: [{name: html, mountPath: /usr/share/nginx/html}]
        resources: {requests: {cpu: "50m", memory: "64Mi"}}
      volumes: [{name: html, configMap: {name: poornima-html}}]
---
apiVersion: v1
kind: Service
metadata:
  name: laxman-service
spec:
  type: LoadBalancer
  ports: [{port: 80, targetPort: 80}]
  selector: {app: laxman}
---
apiVersion: v1
kind: Service
metadata:
  name: poornima-service
spec:
  type: LoadBalancer
  ports: [{port: 80, targetPort: 80}]
  selector: {app: poornima}
One Command Deploy:

bash
kubectl apply -f config/portfolio-complete.yaml
kubectl get svc  # 2 LoadBalancer IPs!
💥 #5 Chaos Engineering (Enhanced)
bash
# Terminal 1: Network Watcher
watch -n 0.5 "kubectl get pods,svc -l app=laxman -o wide"

# Terminal 2: Chaos Tests
kubectl delete pods -l app=laxman     # Self-heals instantly
kubectl delete pods -l app=poornima   # Multi-tenant test

# Network Chaos
kubectl scale deployment/laxman-portfolio --replicas=0
kubectl scale deployment/laxman-portfolio --replicas=5
📊 #6 Cost & Networking Summary
Platform	VPC CIDR	Service CIDR	Cost/mo
Minikube	192.168.49.0/24	10.96.0.0/12	$0
kOps Dev	172.20.0.0/16	100.64.0.0/13	$25
kOps Prod	Custom /16	Custom /16	$150
🔍 #7 Complete Troubleshooting
Issue	Fix Command
CIDR Conflict	kops edit cluster --subnets
Multi-Domain 404	kubectl describe ingress multi-tenant-ingress
Service Stuck Pending	minikube tunnel (local)
🎯 Recruiter Checklist
text
✅ Project structure (Git-ready)
✅ CIDR mastery (VPC → Pod networking)
✅ Multi-domain Ingress (1 IP → N domains)
✅ Single YAML deployments
✅ Chaos Engineering (self-healing proof)
✅ Zero-to-production pipeline

Deploy Now: kubectl apply -f config/portfolio-complete.yaml


## 🧹 **#8 COMPLETE CLEANUP COMMANDS** (Your Question)

### **Minikube - Free Your Laptop RAM**
```bash
# 1. Quick pause (resume later)
minikube pause

# 2. Graceful stop (keep data)
minikube stop

# 3. Delete cluster only
minikube delete

# 4. NUCLEAR: Delete ALL clusters + config (~30s)
minikube delete --all --purge

# 5. Docker cleanup (frees 5-10GB)
docker system prune -a --volumes

# 6. Verify clean slate
minikube status  # "No minikube clusters found"
```

### **kOps AWS - Stop Billing**
```bash
# 1. Delete cluster (keeps S3 state)
kops delete cluster myfirstcluster.k8s.local --yes

# 2. Delete S3 state (permanent)
aws s3 rb s3://$KOPS_STATE_STORE --force

# 3. Verify zero cost
aws ec2 describe-instances --filters "Name=tag:KubernetesCluster,Values=myfirstcluster.k8s.local"
```

### **kubectl Cleanup**
```bash
# Delete all your apps
kubectl delete all --all -l app=portfolio
kubectl delete all --all -l app=poornima
kubectl delete ingress --all
```
