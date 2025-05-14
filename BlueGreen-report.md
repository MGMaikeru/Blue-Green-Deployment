# Blue-Green Deployment Implementation Report

_Date: May 14, 2025_

## Executive Summary

This document details the implementation of a Blue-Green deployment strategy for a banking application using Kubernetes (EKS), Jenkins, SonarQube, and Nexus. Blue-Green deployment is a technique that reduces downtime and risk by running two identical production environments called Blue and Green. At any time, only one of the environments is live, serving all production traffic. This approach allows for seamless updates and quick rollbacks if needed.

## Project Infrastructure

The project infrastructure consists of four virtual machines:

| Service     | IP Address     | Purpose                                   |
| ----------- | -------------- | ----------------------------------------- |
| Source Code | 18.191.233.153 | Main server for AWS CLI, Terraform & code |
| Jenkins     | 3.136.20.30    | CI/CD pipeline orchestration              |
| SonarQube   | 3.129.22.222   | Code quality analysis                     |
| Nexus       | 3.145.112.50   | Artifact repository management            |

## Implementation Process

### Before starting

It is necessary to configure the security group so that the vms belong to it. For this, a terraform code was used to initialize an initial VM and the security group (https://github.com/MGMaikeru/aws-blue-green).

![alt text](images/image.png)

In addition, it is necessary to create an access key (to configure aws cli) and security key type .pem to make the connection with the vms. You can connect to a vm usign:

```bash
ssh -i {.pem file name} ubuntu@{vm api adress}
```

### 1. Setting Up the Main Server

The first step was to configure the main server with necessary tools for AWS infrastructure deployment:

```bash
# Update system packages
sudo apt update
sudo apt upgrade -y

# Install AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install

# Configure AWS credentials
aws configure

# Install Terraform
sudo snap install terraform --classic

# Clone project repository
git clone https://github.com/MGMaikeru/Blue-Green-Deployment.git

# Initialize and apply Terraform configuration
terraform init
terraform apply
```

SSH access to all virtual machines was established using:

```bash
ssh -i Blue-Green.pem ubuntu@<server-ip>
```

### 2. Jenkins Server Setup

Jenkins was installed and configured on the dedicated server:

```bash
sudo apt update
sudo apt install openjdk-17-jre-headless -y
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key

echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt-get update
sudo apt-get install jenkins
sudo systemctl enable jenkins
sudo systemctl start jenkins
```

Later, Docker and additional tools were installed on the Jenkins server:

```bash
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker jenkins

# Install Trivy for security scanning
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy

# Install kubectl
sudo snap install kubectl --classic
```

### 3. Nexus Repository Setup

Nexus was set up as a Docker container:

```bash
sudo apt update
sudo apt install docker.io -y
sudo usermod -aG docker $USER
newgrp docker
docker run -d --name nexus3 -p 8081:8081 sonatype/nexus3

# Retrieve initial admin password
docker exec -it nexus3 cat /nexus-data/admin.password
```

### 4. SonarQube Setup

SonarQube was also deployed as a Docker container:

```bash
sudo apt update
sudo apt install docker.io -y
sudo usermod -aG docker ubuntu
newgrp docker
docker run -d -p 9000:9000 sonarqube:lts-community
```

### 5. Kubernetes Configuration

After EKS cluster creation through Terraform, the required Kubernetes resources were configured:

```bash
# Install kubectl on the main server
sudo snap install kubectl --classic

# Configure kubectl to use the EKS cluster
aws eks --region us-east-2 update-kubeconfig --name devopsshack-cluster

# Create namespace for web applications
kubectl create ns webapps

# Apply Kubernetes RBAC configurations
kubectl apply -f sa.yml
kubectl apply -f role.yml
kubectl apply -f rolebind.yml
kubectl apply -f sec.yaml -n webapps

# Retrieve service account token for Jenkins
kubectl describe secret mysecretname -n webapps
```

The retrieved token was:

```
eyJhbGciOiJSUzI1NiIsImtpZCI6IlpqbWFOSkVFVlpWaTFvR01LYXo0NTFFLXhCUE41cVZsMHQwLXdqX0d6S28ifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJ3ZWJhcHBzIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6Im15c2VjcmV0bmFtZSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJqZW5raW5zIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiMDBlNmYwNTYtZjAxZC00ODQxLTg5NjQtMmEwYWJmNGUyNGIzIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OndlYmFwcHM6amVua2lucyJ9.WRPmj4dozeebfH-htys2oT2lWzdqIMtOsTP2BbqxlWBHeei6ZcVMKYC2liqxueLB_zbxqTZZyXQfR3OwuCv2LRp323s9eyMfp9aC84gbN-yQ-vY5tcCrV_g-UBWcyl95GrVGHCYM_CZyESKjuS8_nfSJQJJLaorN2i5v__IpV9fxt4yBOo-ZBQxKDsPCh0srqaL2Os40w8PnMsxH8NRHgZlV1fwWrBJsaQlCR9C3fLh6-c035jKL2Uk0I2AepUUpjNfh5MCiqQu-c-8o1tBUnLT9OS9-8hANIqnIaUhhUJZYc08A3Ff-2wdhgKfpym2CyvmcW5FloaJnz2V42Nko6A
```

### 6. Jenkins Pipeline Configuration

In Jenkins, the following steps were taken:

1. Created credentials for:

   - Docker Hub
   - Git repository access
   - Kubernetes cluster access
   - SonarQube access

2. Created and configured the initial pipeline
3. Set up Maven configuration XML
4. Configured Maven and SonarQube scanner with correct names
5. Added necessary libraries for the pipeline
6. Added webhook integration with SonarQube for quality gate feedback

## Blue-Green Deployment Architecture

The implemented Blue-Green deployment architecture consists of:

1. **Two identical environments**:

   - Blue environment (`app-deployment-blue.yml`)
   - Green environment (`app-deployment-green.yml`)

2. **Single service** (`bankapp-service.yml`) that routes traffic to either Blue or Green environment based on the selector label `version: blue/green`

3. **Jenkins pipeline** with parameters:
   - `DEPLOY_ENV`: Choose which environment to deploy (blue/green)
   - `DOCKER_TAG`: Select Docker image tag for deployment (blue/green)
   - `SWITCH_TRAFFIC`: Boolean to control traffic switching between environments

## Deployment Process

The Blue-Green deployment process follows these steps:

1. **Initial Deployment**: The application is deployed to the Blue environment, and the service points to Blue.

2. **New Version Deployment**:

   - A new version is deployed to the inactive environment (Green)
   - The CI/CD pipeline builds, tests, and deploys the application
   - Security scanning is performed using Trivy

3. **Testing**: The new version in the Green environment is tested independently without affecting production traffic.

4. **Traffic Switch**:

   - Once validated, traffic is switched from Blue to Green by updating the service selector

   ```bash
   kubectl patch service bankapp-service -p '{"spec": {"selector": {"app": "bankapp", "version": "green"}}}' -n webapps
   ```

   - The switch is immediate and transparent to users

5. **Rollback**: If issues are detected, traffic can be quickly switched back to the previous environment.

## Benefits Realized

1. **Zero Downtime Deployments**: Users experience no service interruption during updates
2. **Reduced Risk**: New versions are fully validated before receiving production traffic
3. **Immediate Rollback**: Simple rollback mechanism by switching traffic back to the previous environment
4. **Independent Testing**: New versions can be tested in production-like conditions without affecting users
5. **Enhanced Quality Control**: Integration with SonarQube ensures code quality standards are maintained

## Conclusion

The Blue-Green deployment strategy implemented for this banking application provides a robust mechanism for releasing new versions with minimal risk. By leveraging Kubernetes, Jenkins, SonarQube, and Nexus, the entire CI/CD pipeline ensures high-quality, secure, and reliable software delivery.

The architecture allows for future enhancements such as automated canary releases or A/B testing by extending the current deployment framework. The implemented solution meets the requirements for modern, cloud-native application deployment with a focus on reliability and user experience.
