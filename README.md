# Setting up a CI/CD pipeline using Jenkins and Argo CD, focusing on deploying a Python-based application on Kubernetes.

## What is CI/CD?**

**Continuous Integration (CI):**
- Developers frequently merge code changes into a shared repository.
- The system automatically builds, tests, and validates code changes.
- Early detection of integration bugs.

**Continuous Delivery (CD):**
- Automates the deployment process to staging or production environments.
- Ensures new changes are always ready for deployment.

**Goal:**
- Automate the build, test, and deployment process, reducing manual effort and increasing reliability.

## Tools Used:

**Jenkins - Handles Continuous Integration (CI):**
- Builds the application.
- Creates a Docker image.
- Pushes the image to a container registry (e.g., Docker Hub).
- Updates Kubernetes manifests with the new image tag.

**Argo CD - Handles Continuous Delivery (CD):**
- Monitors Kubernetes manifest files in a Git repository.
- Automatically deploys new changes to a Kubernetes cluster.
- Auto-healing capabilities: Ensures deployments match the desired state in Git.
- Provides a user-friendly dashboard.
- Declarative configuration with GitOps.

## Workflow Overview

**Step 1: Application Change**
- A developer makes a change in the application code (e.g., modifies an HTML header).
- The code is pushed to GitHub.

**Step 2: Jenkins Pipeline**
- The Jenkins pipeline is triggered (either automatically or manually).
- Stages:
    - Checkout: Pulls the latest code from the repository.
    - Build Docker Image: Builds a new Docker image with a unique tag.
    - Push Image: Uploads the Docker image to Docker Hub.
    - Update Kubernetes Manifests: Updates the image tag in Kubernetes YAML files.

**Step 3: Argo CD Deployment**
- Argo CD continuously watches the Kubernetes manifest repository for changes.
- Argo CD detects changes in the Kubernetes manifest repository.
When it detects an updated image tag in the Kubernetes manifest repository, it automatically applies the changes to the Kubernetes cluster.
- If someone manually alters configurations in the Kubernetes cluster, Argo CD detects the drift and resets it based on the GitHub configuration.

**Step 4: Verification**
- The application is deployed with the new changes.
- Changes can be verified via the service URL.

## Implementation Steps

![todo App](https://raw.githubusercontent.com/shreys7/django-todo/develop/staticfiles/todoApp.png)

![Screenshot 2023-02-01 at 2 48 06 PM](https://user-images.githubusercontent.com/43399466/216001659-74024e94-2c3c-4f1a-8e2e-3ef69b3a88ad.png)

### Step 1: Launch an Ec2 Instance for CICD setup
- Choose Ubuntu machine
- Open inbound traffic rules for necessary ports (e.g., `22`, `80`, `443`, `8000`, `8080`).
- SSH into the EC2 instance: 
    `ssh -i <your-key.pem> ubuntu@<ec2-ip>`
```bash
# Update system packages:
sudo apt update && sudo apt upgrade -y

# Install Required Dependencies:
sudo apt install apt-transport-https ca-certificates git curl wget unzip   software-properties-common -y
```

### Step 2: Jenkins Setup (Continuous Integration Tool)
```bash
# Install Java
sudo apt install openjdk-17-jre -y

# Add Jenkins Repository and Install Jenkins
wget -q -O - https://pkg.jenkins.io/debian/jenkins.io.key | sudo apt-key add -
sudo sh -c 'echo deb http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
sudo apt update -y
sudo apt install jenkins -y
ps -ef | grep jenkins

# Start Jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins

# By default, Jenkins will not be accessible to the external world due to the inbound traffic restriction by AWS. Open port 8080 in the inbound traffic rules as show below.
# Access Jenkins via http://<EC2_Public_IP>:8080.

# Retrieve the initial admin password:
sudo cat /var/lib/jenkins/secrets/initialAdminPassword

# Create first Admin User or Skip the step [If you want to use this Jenkins instance for future use-cases as well, better to create admin user]

# Install required plugins.
# Git Plugin, Docker Pipeline Plugin, Kubernetes Continuous Deploy plugin.
# Go to Manage Jenkins → Manage Plugins → Available Plugins and install them.
# Restart Jenkins after the plugins are installed.
# http://<ec2-instance-public-ip>:8080/restart

# Create a new pipeline job (New Item -> Pipeline) and configure it with the Git repository URL for the Python application.

# Add Github credentials (Settings -> Developer Settings -> Personal Access Token -> Generate Token)
# Go to Manage Jenkins → Manage Credentials → Add Credentials
```

### Step 3: Install Docker
```bash
sudo apt install docker.io
sudo usermod -aG docker jenkins
sudo usermod -aG docker ubuntu
sudo systemctl restart docker
sudo systemctl restart jenkins
# Another way to restart jenkins: `http://<ec2-instance-public-ip>:8080/restart`

# Configure DockerHub Credentials in Jenkins
# Go to Manage Jenkins → Manage Credentials → Add Credentials
# Add DockerHub Token under Global Credentials.
```

### Step 4: Minikube and Argo CD Setup (Continuous Delivery)
```bash
# Install Minikube
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube /usr/local/bin/
rm minikube

# Install Kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install kubectl /usr/local/bin/
rm kubectl
kubectl get nodes

# Start Minikube Cluster
minikube start --driver=docker
minikube status

# Create namespace
kubectl create namespace argocd

# Install Argo CD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods

# Update the Argo CD server service to from type: "ClusterIP" to "NodePort".
kubectl get svc
kubectl edit svc argocd-server
kubectl get svc

# Access ArgoCD UI
kubectl get pods
minikube service argocd-server
minikube service list

# Open the provided URL in your browser

# Username: admin
# Retrieve the default password
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d

# Login to the ArgoCD UI and craete a new application providing the github repo (url + path) of manifest files.

# Sync the application and verify
kubectl get deploy
kubectl get pods
```

### Step 5: Validate the Workflow
- Commit changes to the Git repository.
- Jenkins pipeline triggers automatically.
- Docker builds and pushes the image.
- Argo CD updates Kubernetes manifests and deploys the app.
- Validate the deployment in minikube cluster.
- Monitor deployment logs.
- Verify the running application.
- Fix any issues that arise.





