Project Architecture: K8s on AWS (kOps)
Before running commands, it is important to understand how the components interact:

S3 State Store: Acts as the "source of truth," storing your cluster configuration.

Control Plane (Master Node): Uses a t3.small (2GB RAM) to handle the API Server and etcd without crashing.

Worker Node: Uses a t3.micro (1GB RAM) to run your portfolio containers.

Load Balancer (ELB): The public "Front Door" that provides the URL for your browser.

Step-by-Step Implementation
1. Setup SSH Security Keys
We need these to securely log into the nodes if something goes wrong.

Command: ssh-keygen -t rsa -b 2048 -f ~/.ssh/id_rsa -N ""

Explanation: Generates a public/private key pair.

Output: Two files: id_rsa (private) and id_rsa.pub (public) in your .ssh folder.

2. Prepare the State Store (S3)
Command: aws s3 mb s3://kops-laxman-storage-1
export KOPS_STATE_STORE="s3://kops-laxman-storage-1"

Explanation: Creates a bucket in AWS to hold cluster settings.

Output: make_bucket: kops-laxman-storage-1

3. Provision the Cluster
We specify the larger t3.small for the master to ensure stability.

Command:

Bash
kops create cluster \
  --name=myfirstcluster.k8s.local \
  --zones=us-east-1a \
  --node-count=1 \
  --node-size=t3.micro \
  --control-plane-size=t3.small \
  --ssh-public-key=~/.ssh/id_rsa.pub

kops update cluster --name myfirstcluster.k8s.local --yes --admin
Explanation: Defines the infrastructure and then tells AWS to actually build it.

Output: A list of AWS resources being created (VPC, Subnets, EC2 instances).

4. Deploy Your Custom Portfolio
We use a ConfigMap to inject your HTML code into the standard Nginx container.

Command (Step A - Create Content):

Bash
cat <<'EOF' > index.html
EOF

kubectl create configmap portfolio-html --from-file=index.html
Command (Step B - Create Deployment):

Bash
cat <<EOF | kubectl apply --validate=false -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: laxman-portfolio
spec:
  replicas: 1
  selector:
    matchLabels:
      app: laxman-site
  template:
    metadata:
      labels:
        app: laxman-site
    spec:
      containers:
      - name: nginx
        image: nginx
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
  selector:
    app: laxman-site
EOF
Explanation: Mounts your index.html file into the Nginx web directory and creates a Load Balancer.

5. Verify and Access
Command: kubectl get svc laxman-service

Output: An EXTERNAL-IP address (e.g., af10cd5b...us-east-1.elb.amazonaws.com).

Result: Paste this into your browser to see your live portfolio.

6. Safe Deletion
Command: kops delete cluster --name myfirstcluster.k8s.local --yes

Explanation: Completely wipes the AWS resources to prevent billing.

Output: Deleted cluster: "myfirstcluster.k8s.local"