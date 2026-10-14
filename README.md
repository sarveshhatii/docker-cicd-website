Automated Docker CI/CD Pipeline with AWS EC2 and HTTPS
A static HTML website deployment project that automates Docker image
building and deployment using GitHub Actions, Docker Hub, AWS
EC2, and AWS Systems Manager (SSM). The public website is served
through Nginx Proxy Manager, with a Let's Encrypt certificate
for HTTPS.
Project status note: The CI/CD build and deployment workflow has
been working in the shared project. DNS, Nginx Proxy Manager,
certificate issuance, and HTTPS redirection should be verified in the
live environment before describing them as fully operational.

Table of contents
- Project overview
- Architecture
- Technology stack
- Repository structure
- How the CI/CD pipeline works
- Dockerfile
- Docker Compose
- GitHub Actions workflow
- AWS and GitHub configuration
- DNS and HTTPS setup
- Deploying a website change
- Verification commands
- Troubleshooting
- Security notes
- Future improvements
Project overview
The goal is to deploy a website automatically when code is pushed to the
main branch. Rather than manually copying files to a server, the
pipeline builds a Docker image, publishes it to Docker Hub, and updates
the website service on an AWS EC2 instance.
Nginx Proxy Manager (NPM) is used as the public reverse proxy. It routes
requests for the custom domain to the website container and manages the
Let's Encrypt TLS certificate.
Project links
- GitHub repository:
  https://github.com/sarveshhatii/docker-cicd-website
- Docker Hub image:
  https://hub.docker.com/r/sarveshhatii/cicd-website
- Website domain: https://sarvesh.pranat.online/
Main outcomes
- Package a static HTML website with NGINX in a Docker image.
- Build and publish the image automatically with GitHub Actions.
- Store the image in Docker Hub using both latest and commit-SHA
  tags.
- Authenticate GitHub Actions to AWS using OpenID Connect (OIDC).
- Deploy to EC2 by invoking AWS Systems Manager (SSM), without
  requiring an SSH private key in the workflow.
- Use Docker Compose to define the website and Nginx Proxy Manager
  services.
- Configure DNS, a Let's Encrypt certificate, and HTTP-to-HTTPS
  redirection.
Architecture
Developer edits index.html
        |
        v
Git commit and push to GitHub main
        |
        v
GitHub Actions runner
  1. Checkout source
  2. Set up Docker Buildx
  3. Log in to Docker Hub
  4. Build and push image tags
  5. Assume AWS IAM role using OIDC
  6. Send deployment command using SSM
        |
        v
AWS EC2 (Ubuntu + Docker Engine)
  +-- website container: cicd-website
  |      NGINX serves HTML on container port 80
  |
  +-- nginx-proxy-manager
         Host port 80  -> HTTP
         Host port 81  -> admin dashboard
         Host port 443 -> HTTPS
        |
        v
DNS: sarvesh.pranat.online -> EC2 public IPv4
        |
        v
Browser -> Nginx Proxy Manager -> http://cicd-website:80
        |
        v
Website over HTTPS
Technology stack
  Technology                          Purpose
  HTML                                Static website content
  NGINX Alpine                        Serves the HTML file inside the
                                      container
  Docker                              Packages the website and NGINX
  Docker Compose                      Defines and manages the website and
                                      NPM services
  Git and GitHub                      Source control and repository
                                      hosting
  GitHub Actions                      Automates build, image publishing,
                                      and deployment
  Docker Hub                          Container image registry
  AWS EC2                             Hosts the Docker containers
  AWS IAM and GitHub OIDC             Grants the workflow temporary AWS
                                      credentials
  AWS Systems Manager                 Executes deployment commands on EC2
  Nginx Proxy Manager                 Reverse proxy and TLS certificate
                                      management
  Let's Encrypt                       Free TLS certificate authority
Repository structure
The core project files are organized like this:
docker-cicd-website/
├── index.html
├── Dockerfile
├── docker-compose.yml
└── .github/
    └── workflows/
        └── ci-cd.yml
Your repository may include additional files. Keep the workflow under
.github/workflows/ so GitHub Actions can discover it.
How the CI/CD pipeline works
1. A developer changes index.html.
2. The change is committed and pushed to main.
3. GitHub Actions starts a job on a GitHub-hosted Ubuntu runner.
4. The workflow checks out the repository and prepares Docker Buildx.
5. It logs in to Docker Hub using GitHub repository secrets.
6. It builds the Docker image and pushes:
   - sarveshhatii/cicd-website:latest
   - sarveshhatii/cicd-website:<commit-sha>
7. GitHub Actions assumes the GitHubActions-EC2-Deploy AWS role
   through OIDC.
8. The workflow calls SSM to run deployment commands on EC2.
9. EC2 pulls the latest website image and updates only the website
   Compose service.
10. The browser reaches the domain through Nginx Proxy Manager, which
    forwards requests to the website container.
Dockerfile
FROM nginx:alpine

COPY index.html /usr/share/nginx/html/index.html

EXPOSE 80
Explanation
- FROM nginx:alpine uses NGINX on the lightweight Alpine Linux base
  image.
- COPY places the website file in NGINX's default document root.
- EXPOSE 80 documents the container port used by NGINX. It does not
  publish the port to the EC2 host by itself.
Docker Compose
The HTTPS-oriented Compose configuration shared for this project is:
version: '3.8'

services:
  website:
    image: sarveshhatii/cicd-website:latest
    container_name: cicd-website
    restart: unless-stopped

  nginx-proxy-manager:
    image: 'jc21/nginx-proxy-manager:latest'
    container_name: nginx-proxy-manager
    restart: unless-stopped
    ports:
      - '80:80'
      - '81:81'
      - '443:443'
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
Important configuration details
- The website service does not publish host port 80. Nginx Proxy
  Manager owns the public ports 80 and 443.
- NPM forwards requests to cicd-website on container port 80 over
  the Compose network.
- Port 81 is the NPM administration dashboard. Restrict it to a
  trusted IP in the AWS security group where possible.
- The ./data and ./letsencrypt mounts preserve NPM configuration
  and certificate data when its container is recreated.
- The workflow's SSM command expects the Compose file at
  /home/ubuntu/docker-cicd-website/docker-compose.yml and a service
  named website.
Port conflict warning: If an older standalone cicd-website
container still publishes host port 80, NPM cannot bind to that same
host port. Inspect the live containers and plan the changeover before
stopping or removing anything.

GitHub Actions workflow
The current workflow builds and pushes the image, authenticates to AWS
through OIDC, and uses SSM to update the website service. The file path
is .github/workflows/ci-cd.yml.
name: Docker CI/CD to AWS EC2

on:
  push:
    branches:
      - main

permissions:
  contents: read
  id-token: write

env:
  DOCKERHUB_IMAGE: sarveshhatii/cicd-website
  AWS_REGION: ap-south-1
  EC2_INSTANCE_ID: i-050f5b054472305cf

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: |
            ${{ env.DOCKERHUB_IMAGE }}:latest
            ${{ env.DOCKERHUB_IMAGE }}:${{ github.sha }}

      - name: Configure AWS credentials using OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::020671572525:role/GitHubActions-EC2-Deploy
          aws-region: ${{ env.AWS_REGION }}

      - name: Deploy image to EC2 using SSM
        shell: bash
        run: |
          set -euo pipefail

          COMMAND_ID=$(aws ssm send-command \
            --instance-ids "$EC2_INSTANCE_ID" \
            --document-name "AWS-RunShellScript" \
            --comment "Deploy website updates safely" \
            --parameters '{"commands":[
              "set -e",
              "cd /home/ubuntu/docker-cicd-website",
              "sudo docker compose pull website",
              "sudo docker compose up -d --no-deps website",
              "sudo docker compose ps"
            ]}' \
            --query 'Command.CommandId' \
            --output text)

          echo "SSM Command ID: $COMMAND_ID"

          for i in {1..30}; do
            STATUS=$(aws ssm get-command-invocation \
              --command-id "$COMMAND_ID" \
              --instance-id "$EC2_INSTANCE_ID" \
              --query 'Status' \
              --output text 2>/dev/null || true)

            echo "Deployment status: $STATUS"

            if [ "$STATUS" = "Success" ]; then
              exit 0
            fi

            if [ "$STATUS" = "Failed" ] || \
               [ "$STATUS" = "Cancelled" ] || \
               [ "$STATUS" = "TimedOut" ]; then
              aws ssm get-command-invocation \
                --command-id "$COMMAND_ID" \
                --instance-id "$EC2_INSTANCE_ID" \
                --output json
              exit 1
            fi

            sleep 10
          done

          echo "Deployment did not finish within the expected time."
          exit 1
Workflow concepts
  Item                                       Explanation
  on.push.branches: main                   Triggers the workflow on pushes to
                                             main.
  runs-on: ubuntu-latest                   Runs the job on a GitHub-hosted
                                             runner.
  actions/checkout                         Checks out the repository source.
  Docker Buildx                              Prepares Docker build tooling.
  Docker login action                        Authenticates with Docker Hub using
                                             secrets.
  Docker build/push action                   Builds and publishes the image
                                             tags.
  id-token: write                          Allows the workflow to request a
                                             GitHub OIDC token.
  AWS credentials action                     Assumes the AWS IAM role using
                                             OIDC.
  aws ssm send-command                     Requests remote deployment commands
                                             on EC2.
  docker compose pull website              Pulls the latest image for the
                                             website service.
  docker compose up -d --no-deps website   Updates the website service without
                                             deliberately recreating its
                                             dependencies.
  Polling loop                               Waits for SSM to finish and reports
                                         failures/timeouts.
AWS and GitHub configuration
AWS resources used
The project configuration records:
- Region: ap-south-1 (Mumbai)
- EC2 instance ID: i-050f5b054472305cf
- GitHub Actions IAM role: GitHubActions-EC2-Deploy
- EC2 Systems Manager role: EC2-SSM-Role
These values are specific to the project account. Verify the instance,
role ARNs, trust policies, and permissions before reusing the workflow
in another account.
GitHub repository secrets
In GitHub → repository Settings → Secrets and variables → Actions,
configure:
  Secret                 Value
  DOCKERHUB_USERNAME   Docker Hub username
  DOCKERHUB_TOKEN      Docker Hub personal access token
Never commit credentials into the repository. If a token was exposed,
revoke it and update the GitHub secret.
OIDC setup summary
OIDC allows GitHub Actions to obtain short-lived AWS credentials instead
of storing permanent AWS access keys in GitHub. The provider URL is
https://token.actions.githubusercontent.com, and the audience is
sts.amazonaws.com.
The role trust policy must match the actual repository identity and OIDC
claims. Use least-privilege permissions for SSM deployment rather than
broad administrator access.
EC2 security group
  Port                    Purpose                 Suggested access
  TCP 80                  HTTP and Let's Encrypt  Public
                          HTTP-01 validation      
  TCP 443                 HTTPS                   Public
  TCP 81                  NPM administration      Trusted IP only
  TCP 22                  Optional SSH            Trusted IP only; the
                      administration          workflow uses SSM
The instance also needs a reachable public endpoint, a suitable subnet
route, and outbound connectivity for Docker image pulls and Systems
Manager.
DNS and HTTPS setup
1. Configure DNS
Create or verify an A record for sarvesh.pranat.online pointing to
the EC2 instance's current public IPv4 address. DNS providers may expect
the host/name as sarvesh for this subdomain.
Check resolution in Windows PowerShell:
nslookup sarvesh.pranat.online
The returned IP should match the EC2 public IPv4 address. The address
may change after stopping and starting EC2 unless an Elastic IP is
attached.
2. Access Nginx Proxy Manager
Open:
http://<EC2-PUBLIC-IP>:81
Replace the placeholder with the current EC2 public IP. Ensure NPM is
running and port 81 is permitted from your trusted IP. Change
initial/default administrator credentials immediately.
3. Add the proxy host
In NPM, navigate to Hosts → Proxy Hosts → Add Proxy Host.
  Setting                 Value
  Domain Names            sarvesh.pranat.online
  Scheme                  http
  Forward Hostname / IP   cicd-website
  Forward Port            80
  Block Common Exploits   Enabled
Save the host and verify that it appears as Online. Both containers
should be connected to the same Compose network.
4. Request a Let's Encrypt certificate
1. Edit the proxy host and open the SSL tab.
2. Select Request a new SSL Certificate with Let's Encrypt.
3. Enter an email address and agree to the terms.
4. Save and wait for issuance.
5. Verify that the certificate is selected and active.
Let's Encrypt HTTP-01 validation requires the domain to resolve to the
server and inbound TCP port 80 to reach NPM. HTTPS requires inbound TCP
port 443.
5. Enforce HTTPS
Edit the proxy host, open the SSL tab, enable Force SSL, and save.
Test the website:
curl -I http://sarvesh.pranat.online
curl -I https://sarvesh.pranat.online
Expected result: HTTP redirects to HTTPS, and the HTTPS website loads
without a certificate warning. Verify this against the live server; a
successful build alone does not prove that HTTPS is configured
correctly.
Deploying a website change
After the pipeline is configured, normal content updates should follow
this process:
cd C:\Users\patil\Desktop\docker-cicd-website
git status
git add index.html
git commit -m "Update website content"
git push origin main
Then:
1. Open the repository's Actions tab.
2. Confirm the workflow run succeeds.
3. Check that the image was pushed to Docker Hub.
4. Confirm the SSM deployment step succeeded.
5. Refresh https://sarvesh.pranat.online/. Use Ctrl+F5 if cached
   content appears.
Verification commands
On the local computer
git status
git log -1 --oneline
docker images
docker ps
nslookup sarvesh.pranat.online
curl.exe -I https://sarvesh.pranat.online
On EC2 through AWS Systems Manager
The SSM shell may start in an agent directory rather than the deployment
directory. Use the full Compose path to avoid a
no configuration file provided error.
pwd
sudo systemctl status docker
sudo docker ps -a
sudo cat /home/ubuntu/docker-cicd-website/docker-compose.yml
sudo docker compose -f /home/ubuntu/docker-cicd-website/docker-compose.yml config
sudo docker compose -f /home/ubuntu/docker-cicd-website/docker-compose.yml ps
sudo ss -tulpn | grep -E ':(80|81|443)\b'
sudo docker logs --tail 100 cicd-website
sudo docker logs --tail 100 nginx-proxy-manager
Troubleshooting
  Problem                             What to check
  docker compose ps says no         Use
  configuration file                  -f /home/ubuntu/docker-cicd-website/docker-compose.yml.
  Deployment directory cannot be      Check pwd and locate the actual Compose file before
  found                               changing anything.
  Port 80 is already in use           Inspect sudo docker ps -a and port listeners before
                                      stopping/removing the old container.
  NPM dashboard is unavailable        Check NPM container state, port 81 mapping, and
                                      security-group access.
  Proxy host returns 502 or Offline   Confirm the upstream is cicd-website:80, both services
                                      are running, and they share a Compose network.
  Let's Encrypt certificate request   Check DNS, inbound port 80, port ownership, and NPM logs.
  fails                               
  HTTPS times out                     Check inbound TCP 443 and NPM's port mapping.
  GitHub OIDC role assumption fails   Check the IAM trust policy's repository subject and
                                      audience.
  SSM deployment fails                Inspect SSM command output, IAM permissions, instance ID,
                                      region, Compose path, and service name.
  Old page appears                    Review the Actions run and Docker Hub tags, then
                                  hard-refresh the browser.
Safety: Do not remove or restart production containers blindly.
First inspect the current Compose file, running containers, port
listeners, and logs.
Security notes
- Revoke and replace any exposed Docker Hub personal access token.
- Never commit credentials, passwords, private keys, AWS access keys,
  or full OIDC tokens.
- Change NPM's default admin credentials and restrict port 81.
- Use least-privilege IAM permissions.
- Keep ports 80 and 443 available for public web traffic and
  certificate validation.
- Keep NPM configuration and certificate data in persistent volumes.
- Review AWS charges for EC2, storage, and public IPv4 resources.