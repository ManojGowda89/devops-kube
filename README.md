# Kubernetes Next.js Deployment Guide

This repository contains configuration files and instructions for deploying a Next.js application on Kubernetes with auto-scaling capabilities.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Kubernetes Setup](#kubernetes-setup)
- [Application Deployment](#application-deployment)
- [Auto-scaling Configuration](#auto-scaling-configuration)
- [Accessing Your Application](#accessing-your-application)
- [Monitoring and Troubleshooting](#monitoring-and-troubleshooting)
- [CI/CD Pipeline](#cicd-pipeline)

## Prerequisites

- Ubuntu server (20.04 or later)
- Sudo privileges
- At least 2 CPUs and 2GB RAM
- Internet connection

## Kubernetes Setup

### 1. Update System

```bash
sudo apt-get update
sudo apt-get upgrade -y
```

### 2. Install Container Runtime (containerd)

```bash
# Install required packages
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release

# Add Docker's official GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Add Docker repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install containerd
sudo apt-get update
sudo apt-get install -y containerd.io

# Configure containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
sudo systemctl restart containerd
```

### 3. Disable Swap

```bash
# Disable swap immediately
sudo swapoff -a

# Disable swap permanently (comment out swap entries)
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### 4. Configure Kernel Modules and System Configuration

```bash
# Load required modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Configure sysctl parameters
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl parameters
sudo sysctl --system
```

### 5. Install Kubernetes Components

```bash
# Add Kubernetes apt repository
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# Download the GPG key for Kubernetes repositories
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add the Kubernetes apt repository
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Install kubelet, kubeadm, and kubectl
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl

# Pin their version
sudo apt-mark hold kubelet kubeadm kubectl
```

### 6. Initialize Kubernetes Cluster

```bash
# Initialize the cluster with a pod network CIDR
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# Set up kubeconfig for the current user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 7. Install Network Plugin (Calico)

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml
```

### 8. Join Worker Nodes (if applicable)

```bash
# Use the join command that was output from kubeadm init
sudo kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash <hash>
```

### 9. Allow Workloads on Control Plane (for single node clusters)

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

### 10. Install Metrics Server

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### 11. Install Helm (optional)

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

## Application Deployment

### 1. Create Deployment Configuration

Create a file named `nextjs-deployment.yaml`:

```yaml name=nextjs-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nextjs-app
  labels:
    app: nextjs-app
spec:
  replicas: 2  # Starting with minimum 2 pods
  selector:
    matchLabels:
      app: nextjs-app
  template:
    metadata:
      labels:
        app: nextjs-app
    spec:
      containers:
      - name: nextjs
        image: manoj20002/my-nextjs-app
        ports:
        - containerPort: 3000
        resources:
          requests:
            cpu: "100m"  # Request 100m (0.1) CPU cores
            memory: "128Mi"
          limits:
            cpu: "500m"  # Limit to 500m (0.5) CPU cores
            memory: "256Mi"
```

### 2. Create Service Configuration

Create a file named `nextjs-service.yaml`:

```yaml name=nextjs-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nextjs-service
spec:
  selector:
    app: nextjs-app
  ports:
  - port: 80
    targetPort: 3000
  type: NodePort  # Use LoadBalancer if you're on a cloud provider
```

### 3. Configure Horizontal Pod Autoscaler

Create a file named `nextjs-hpa.yaml`:

```yaml name=nextjs-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nextjs-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nextjs-app
  minReplicas: 2
  maxReplicas: 8
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### 4. Apply the Configurations

```bash
# Create the deployment
kubectl apply -f nextjs-deployment.yaml

# Create the service
kubectl apply -f nextjs-service.yaml

# Create the HPA (requires metrics server to be installed)
kubectl apply -f nextjs-hpa.yaml
```

## Accessing Your Application

```bash
# Get the NodePort and Node IP
NODE_PORT=$(kubectl get svc nextjs-service -o jsonpath='{.spec.ports[0].nodePort}')
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}')
echo "Access your application at: http://$NODE_IP:$NODE_PORT"
```

## Monitoring and Troubleshooting

### Verifying Your Deployment

```bash
# Check the deployment
kubectl get deployments

# Check the pods
kubectl get pods

# Check the service
kubectl get svc nextjs-service

# Check the HPA
kubectl get hpa
```

### Testing Auto-scaling

```bash
# Install Apache Benchmark (if not already installed)
sudo apt-get install apache2-utils

# Generate load (replace with your service URL)
ab -n 10000 -c 100 http://$NODE_IP:$NODE_PORT/

# In another terminal, monitor the HPA
kubectl get hpa nextjs-app-hpa --watch

# Monitor pods during scaling
kubectl get pods -l app=nextjs-app --watch
```

### Check Autoscaler Events

```bash
# Check HPA events that show scaling decisions
kubectl describe hpa nextjs-app-hpa
```

## CI/CD Pipeline

This repository includes GitHub Actions workflows to automate the build, test, and deployment process.

### Workflow Files

Here are the GitHub Actions workflow files included:

```yaml name=.github/workflows/nextjs-ci.yml
name: Next.js CI

on:
  push:
    branches: [ main ]
    paths:
      - 'src/**'
      - 'public/**'
      - 'package.json'
      - 'package-lock.json'
      - 'next.config.js'
  pull_request:
    branches: [ main ]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run linter
        run: npm run lint
      
      - name: Build application
        run: npm run build
      
      - name: Run tests
        run: npm test
```

```yaml name=.github/workflows/build-push-deploy.yml
name: Build, Push, and Deploy

on:
  push:
    branches: [ main ]
    paths:
      - 'src/**'
      - 'public/**'
      - 'Dockerfile'
      - 'package.json'
      - 'next.config.js'
      - 'kubernetes/*.yaml'
  
  workflow_dispatch:

env:
  DOCKER_IMAGE: manoj20002/my-nextjs-app
  KUBERNETES_NAMESPACE: default

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Login to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: ${{ env.DOCKER_IMAGE }}:latest,${{ env.DOCKER_IMAGE }}:${{ github.sha }}
          cache-from: type=registry,ref=${{ env.DOCKER_IMAGE }}:buildcache
          cache-to: type=registry,ref=${{ env.DOCKER_IMAGE }}:buildcache,mode=max
      
  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up kubectl
        uses: azure/setup-kubectl@v3
        
      - name: Configure kubectl
        run: |
          mkdir -p $HOME/.kube
          echo "${{ secrets.KUBE_CONFIG }}" > $HOME/.kube/config
          chmod 600 $HOME/.kube/config
      
      - name: Update image tag in manifests
        run: |
          sed -i "s|image: manoj20002/my-nextjs-app|image: manoj20002/my-nextjs-app:${{ github.sha }}|g" nextjs-deployment.yaml
      
      - name: Apply Kubernetes manifests
        run: |
          kubectl apply -f nextjs-deployment.yaml
          kubectl apply -f nextjs-service.yaml
          kubectl apply -f nextjs-hpa.yaml
          
      - name: Verify deployment
        run: |
          kubectl rollout status deployment/nextjs-app
```

### Setup Secrets for GitHub Actions

For the CI/CD pipeline to work, you need to add the following secrets in your GitHub repository settings:

1. `DOCKER_USERNAME`: Your Docker Hub username
2. `DOCKER_PASSWORD`: Your Docker Hub password or access token
3. `KUBE_CONFIG`: Your Kubernetes config file content (get it from `~/.kube/config`)

## Next Steps

1. Consider adding monitoring tools like Prometheus and Grafana
2. Implement proper secrets management
3. Set up environment-specific deployments (dev, staging, prod)
4. Configure ingress for proper domain routing

## Additional Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Next.js Documentation](https://nextjs.org/docs)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)

## License

This project is licensed under the MIT License - see the LICENSE file for details. 




What is KUBE_CONFIG?
KUBE_CONFIG is a GitHub secret that contains your Kubernetes configuration file content. This configuration file has all the details needed to connect to your Kubernetes cluster, including:

Cluster information (API server addresses)
Authentication credentials
Contexts and namespaces
How to Get Your KUBE_CONFIG Content
When you set up Kubernetes on your machine, a configuration file was automatically created at ~/.kube/config. To use this in your GitHub Actions:

On your Kubernetes machine, view the content of your config file:

bash
cat ~/.kube/config
Copy the entire output (it's a YAML formatted file)

In your GitHub repository:

Go to Settings → Secrets and variables → Actions
Click "New repository secret"
Name: KUBE_CONFIG
Value: Paste the entire content of your ~/.kube/config file
Click "Add secret"
Security Note
Your kubeconfig file contains sensitive information that grants access to your Kubernetes cluster. When adding it as a GitHub secret:

Make sure to use repository secrets (not environment variables)
Consider creating a service account with limited permissions specifically for CI/CD
Regularly rotate credentials  


