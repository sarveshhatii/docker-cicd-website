# Automated Docker CI/CD Pipeline and AWS EC2 Deployment using GitHub Actions

##  Overview

This project demonstrates how to automate the build, containerization, and deployment of a static website using **Docker, Docker Hub, GitHub Actions, AWS EC2, and AWS Systems Manager (SSM)**.

Whenever new code is pushed to the `main` branch of the GitHub repository, GitHub Actions automatically builds a Docker image, pushes it to Docker Hub, and deploys the updated application to an AWS EC2 instance using AWS Systems Manager Run Command.

The deployment uses **GitHub OIDC authentication** to access AWS without storing long-lived AWS access keys in GitHub Secrets.

## Objectives

- Containerize a static website using Docker.
- Build and publish Docker images automatically.
- Implement CI/CD using GitHub Actions.
- Deploy the website on an AWS EC2 Ubuntu server.
- Use GitHub OIDC for secure AWS authentication.
- Use AWS Systems Manager instead of SSH for automated deployment.
- Automatically replace the old container with the updated application.
- Verify deployment success through the GitHub Actions workflow.

##  Architecture

```text
Developer
    |
    | Push code to main
    v
GitHub Repository
    |
    v
GitHub Actions Workflow
    |
    | Build Docker Image
    v
Docker Hub
    |
    | Push latest and commit-specific tags
    v
Docker Hub Repository
    |
    ^ Pull latest image
    |
AWS EC2 Ubuntu Server
    ^
    | AWS Systems Manager Run Command
    |
GitHub Actions
    |
    | GitHub OIDC authentication
    v
AWS IAM Role
```

### Application deployment flow

1. A developer updates the website code.
2. The changes are pushed to the GitHub `main` branch.
3. GitHub Actions checks out the repository.
4. Docker Buildx builds the Docker image.
5. The image is pushed to Docker Hub.
6. GitHub Actions authenticates to AWS using GitHub OIDC.
7. The workflow sends deployment commands through AWS Systems Manager.
8. The EC2 instance pulls the latest image from Docker Hub.
9. The existing application container is removed and recreated.
10. The updated website becomes available through the EC2 instance's public IP address.

## 🛠️ Technologies Used

| Technology | Purpose |
|---|---|
| HTML | Website content |
| Docker | Containerization |
| Nginx Alpine | Web server for serving the static website |
| Docker Hub | Container image registry |
| Git and GitHub | Source code management |
| GitHub Actions | CI/CD automation |
| AWS EC2 | Application hosting |
| AWS IAM | Access control and permissions |
| GitHub OIDC | Temporary AWS authentication |
| AWS Systems Manager | Remote server access and deployment |
| Ubuntu Server 24.04 LTS | EC2 operating system |

##  Project Structure

```text
docker-cicd-website/
│
├── .github/
│   └── workflows/
│       └── ci-cd.yml
│
├── Dockerfile
├── index.html
└── README.md
```

### File descriptions

- `index.html`: Contains the static website content.
- `Dockerfile`: Defines how the Docker image is built.
- `.github/workflows/ci-cd.yml`: Contains the automated CI/CD pipeline.
- `README.md`: Documents the project, architecture, setup, and deployment.

## 🐳 Docker Configuration

The project uses the following Dockerfile:

```dockerfile
FROM nginx:alpine

COPY index.html /usr/share/nginx/html/index.html

EXPOSE 80
```

### Explanation

- `FROM nginx:alpine`: Uses a lightweight Nginx image based on Alpine Linux.
- `COPY`: Copies the website's HTML file into Nginx's default web root.
- `EXPOSE 80`: Documents that the containerized web server listens on port 80.

Nginx serves the static HTML page to users who access the application.

##  Run the Application Locally

### Prerequisites

Install the following tools:

- Git
- Docker Desktop
- A GitHub account
- A Docker Hub account
- An AWS account

Verify the local installations:

```powershell
git --version
docker --version
docker info
```

### Build the Docker image

Run these commands from the project directory:

```powershell
docker build -t cicd-website:v1 .
```

### Run the container

```powershell
docker run -d --name cicd-website-local -p 8080:80 cicd-website:v1
```

Open the following address in a browser:

```text
http://localhost:8080
```

### Verify the container

```powershell
docker ps
```

### Stop and remove the local container

```powershell
docker stop cicd-website-local
docker rm cicd-website-local
```

##  Docker Hub Configuration

The Docker image is published to the following repository:

**Docker Hub:** https://hub.docker.com/r/sarveshhatii/cicd-website

The workflow publishes two image tags:

```text
sarveshhatii/cicd-website:latest
sarveshhatii/cicd-website:<GitHub-commit-SHA>
```

The `latest` tag identifies the image used by the deployment commands. The commit-specific tag helps identify the image associated with a particular source-code revision.

### GitHub repository secrets

Configure the following secrets in:

**GitHub Repository → Settings → Secrets and variables → Actions → New repository secret**

| Secret | Value |
|---|---|
| `DOCKERHUB_USERNAME` | Your Docker Hub username |
| `DOCKERHUB_TOKEN` | A Docker Hub access token |

Never put Docker Hub access tokens or other credentials directly into source code or this README.

## ☁️ AWS EC2 Configuration

The website is hosted on an AWS EC2 Ubuntu instance.

### Instance configuration

| Setting | Configuration |
|---|---|
| Operating system | Ubuntu Server 24.04 LTS |
| Instance type | `t3.micro` |
| AWS Region | `ap-south-1` (Mumbai) |
| Web server | Nginx running inside Docker |
| Application port | TCP 80 |
| Deployment method | AWS Systems Manager Run Command |

### Security group

Configure the EC2 security group according to the intended access requirements.

| Protocol | Port | Purpose |
|---|---:|---|
| HTTP | 80 | Allows users to access the website |
| SSH | 22 | Optional; only required if direct SSH access is needed |

For a public website, HTTP port 80 may be allowed from `0.0.0.0/0`. Restrict SSH to trusted source IP addresses if SSH is required. If SSH is no longer needed and SSM access is verified, remove the inbound SSH rule.

### Install Docker on EC2

Connect through AWS Systems Manager Session Manager and run:

```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable --now docker
sudo docker --version
```

Pull and run the application manually for an initial deployment:

```bash
sudo docker pull sarveshhatii/cicd-website:v1

sudo docker run -d \
  --name cicd-website \
  --restart unless-stopped \
  -p 80:80 \
  sarveshhatii/cicd-website:v1
```

Verify the container:

```bash
sudo docker ps
```

Verify the web server locally on EC2:

```bash
curl -I http://localhost
```

Then open `http://<EC2-PUBLIC-IP>` in your browser.

##  GitHub OIDC and AWS IAM Configuration

GitHub Actions requires AWS permissions to send deployment commands. This project uses OpenID Connect (OIDC) to obtain temporary AWS credentials instead of storing long-lived AWS access keys in GitHub Secrets.

### Required configuration

1. Register GitHub's OIDC provider in AWS IAM.
2. Create the `GitHubActions-EC2-Deploy` IAM role.
3. Configure its trust policy to allow the intended GitHub repository and branch.
4. Attach a least-privilege permissions policy that allows the workflow to send the required SSM command and check its execution status.
5. Attach an EC2 instance role named `EC2-SSM-Role` with the AWS-managed `AmazonSSMManagedInstanceCore` policy.
6. Confirm that the EC2 instance is online in Systems Manager.

The GitHub Actions role and the EC2 instance role serve different purposes:

- **GitHubActions-EC2-Deploy:** Allows the workflow to request deployment commands.
- **EC2-SSM-Role:** Allows the EC2 instance's SSM Agent to communicate with Systems Manager.

The workflow's OIDC trust policy must match the subject claim GitHub actually issues for the repository and branch. Do not assume that every repository uses the same subject format.

The EC2 instance must have suitable outbound connectivity to Systems Manager, usually over HTTPS port 443, or access through appropriately configured VPC endpoints.

## ⚙️ GitHub Actions CI/CD Workflow

Workflow location:

```text
.github/workflows/ci-cd.yml
```

The workflow is triggered whenever code is pushed to the `main` branch.
 
            on:
            push:
                branches:
                - main

 

**Note:** This workflow assumes the EC2 instance, IAM role, SSM connectivity, Docker installation, and GitHub secrets are already configured. The AWS account ID, region, instance ID, role ARN, and Docker Hub image name must match the actual environment before reuse.

##  How Automatic Deployment Works

When a change is pushed to GitHub:

1. GitHub Actions starts the workflow.
2. The source code is checked out.
3. Docker Buildx builds a new image.
4. The image is published to Docker Hub using the `latest` and commit-SHA tags.
5. GitHub Actions obtains temporary AWS credentials using OIDC.
6. AWS Systems Manager receives the deployment command.
7. EC2 pulls the new `latest` image.
8. The old application container is removed.
9. A new container starts with port 80 mapped to port 80.
10. The workflow polls Systems Manager for the command result and fails if deployment does not complete successfully within the configured time.

To deploy a website update, edit `index.html`, commit the changes, and push to `main`.

```powershell
git add .
git commit -m "Update website content"
git push origin main
```

Monitor the run under:

**GitHub Repository → Actions → Docker CI/CD to AWS EC2**

##  Verification

### Check GitHub Actions

Confirm that all workflow steps complete successfully:

- Checkout repository
- Set up Docker Buildx
- Log in to Docker Hub
- Build and push Docker image
- Configure AWS credentials using OIDC
- Deploy image to EC2 using SSM

### Check Docker Hub

Open:

https://hub.docker.com/r/sarveshhatii/cicd-website

Verify that the `latest` and commit-specific tags are present.

### Check EC2

Using Session Manager, run:

```bash
sudo docker ps
sudo docker images
curl -I http://localhost
```

### Check the public website

Open:

```text
http://<EC2-PUBLIC-IP>
```

Use HTTP unless HTTPS has been separately configured. If the instance is stopped and started, its public IPv4 address may change unless an Elastic IP is assigned.

##  Security Best Practices

- Use GitHub OIDC instead of long-lived AWS access keys.
- Give the GitHub Actions IAM role only the permissions it needs.
- Restrict the OIDC trust policy to the intended repository and branch.
- Keep Docker Hub tokens in GitHub Secrets.
- Never commit `.pem` private keys, passwords, or tokens.
- Rotate or revoke any credential that has been exposed.
- Restrict SSH port 22 to trusted IP addresses when SSH is needed.
- Remove the SSH inbound rule when direct SSH is no longer required and an alternative recovery method is confirmed.
- Keep the EC2 instance role separate from the GitHub Actions role.
- Monitor AWS usage and costs.

##  Troubleshooting

| Problem | Possible cause | What to check |
|---|---|---|
| Docker build fails | Invalid Dockerfile or missing files | Check the build logs and project directory |
| Docker Hub login fails | Incorrect username or token | Check the GitHub Secrets configuration |
| OIDC authentication fails | Incorrect trust policy or subject claim | Verify the GitHub OIDC `sub`, audience, and role ARN |
| SSM command cannot be sent | Insufficient IAM permissions | Check the GitHub Actions role policy |
| EC2 is unavailable in SSM | Instance role, agent, or network problem | Verify `AmazonSSMManagedInstanceCore`, SSM Agent, and outbound connectivity |
| Docker pull fails on EC2 | Registry access or image problem | Check Docker Hub visibility, image name, and tag |
| Container fails to start | Port conflict or Docker error | Run `sudo docker ps -a` and inspect container logs |
| Website is unreachable | Incorrect public IP or inbound HTTP rule | Check EC2 state, public IP, security group, and port 80 |
| SSH connection fails | Source IP, key, username, or network issue | Verify SSH configuration or use Session Manager |

To inspect application logs:

```bash
sudo docker logs cicd-website
```

To inspect container status:

```bash
sudo docker ps -a
```

##  Cost Considerations

AWS resources may incur charges depending on usage, account eligibility, and current pricing. Check the AWS Billing Console and AWS Pricing Calculator before running the project for extended periods.

Consider EC2 compute, storage, data transfer, and related AWS services. Stop or delete resources that are no longer needed, while remembering that some resources can continue to incur charges even when an instance is stopped.

##  Future Improvements

- Configure HTTPS with a domain name and TLS certificate.
- Add automated tests and validation before deployment.
- Use Docker image digests for stronger deployment reproducibility.
- Add deployment health checks and rollback support.
- Add monitoring and alerting.
- Use separate development and production environments.
- Add an approval step for production deployments.

##  Learning Outcomes

Through this project, I gained practical experience with:

- Docker image creation and container management.
- Publishing images to Docker Hub.
- GitHub Actions workflows and automated CI/CD.
- AWS EC2 provisioning and security groups.
- AWS IAM roles and least-privilege access.
- GitHub OIDC authentication.
- AWS Systems Manager Session Manager and Run Command.
- Automated remote deployment without requiring inbound SSH access.
- Troubleshooting cloud deployment and networking issues.

 

GitHub: https://github.com/sarveshhatii

Project Repository: https://github.com/sarveshhatii/docker-cicd-website

Docker Hub: https://hub.docker.com/r/sarveshhatii/cicd-website

---

⭐ If you find this project useful, feel free to explore the repository and learn how Docker and GitHub Actions can automate application deployment to AWS EC2.
