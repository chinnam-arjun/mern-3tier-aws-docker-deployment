# Prerequisites

Before starting the deployment, ensure that the required application, containerization, AWS, and database resources are available.

## 1. Application Requirements

The application should already be developed and working locally.

This deployment uses a MERN-based 3-tier architecture:

* **Presentation Tier:** React
* **Application Tier:** Node.js + Express
* **Data Tier:** MongoDB Atlas

> The application source code is maintained in a separate repository. This repository documents only the containerization and AWS deployment process.

## 2. Local Development Environment

The following tools are required on the development machine:

* Git
* Docker
* Docker Compose
* Node.js and npm
* AWS CLI
* SSH client

Verify the installations:

```bash
git --version
docker --version
docker compose version
node --version
npm --version
aws --version
ssh -V
```

## 3. Docker Requirements

Docker is used to package the application components into portable containers.

The deployment requires:

* A frontend Dockerfile
* A backend Dockerfile
* `.dockerignore` files
* Proper application port configuration
* Environment-based configuration for sensitive values

The application should first be tested locally using Docker before deploying it to AWS.

Example:

```bash
docker build -t frontend .
docker build -t backend .
```

Verify the generated images:

```bash
docker images
```

## 4. AWS Requirements

An AWS account with permission to create and manage EC2 resources is required.

The deployment requires:

* AWS account
* AWS CLI configured with appropriate credentials
* EC2 instance
* Key pair for SSH access
* Security Group
* Public or reachable network connectivity
* Sufficient EC2 compute and storage resources

Verify AWS CLI authentication:

```bash
aws sts get-caller-identity
```

## 5. EC2 Requirements

The EC2 instance acts as the Docker host.

The instance should have:

* A Linux-based operating system
* Internet connectivity
* SSH access
* Docker Engine installed
* Required application ports configured
* Sufficient CPU, memory, and storage for the containers

After connecting to the instance, verify Docker:

```bash
docker --version
```

and:

```bash
docker ps
```

## 6. Network Requirements

The EC2 Security Group must allow only the ports required by the deployment.

Typical requirements include:

| Port | Protocol | Purpose                            |
| ---: | -------- | ---------------------------------- |
|   22 | TCP      | SSH administration                 |
|   80 | TCP      | Frontend HTTP traffic              |
|  443 | TCP      | HTTPS traffic, if configured       |
| 5000 | TCP      | Backend API, if externally exposed |

The backend port should **not be publicly exposed unless required**. If the frontend and backend communicate internally through Docker networking, the backend does not necessarily need to be accessible directly from the internet.

> The exact ports used in the deployment should match the actual application and Docker configuration.

## 7. MongoDB Atlas Requirements

MongoDB Atlas is used as the managed data tier.

Before deployment:

* Create or have access to a MongoDB Atlas cluster
* Create a database user
* Obtain the MongoDB connection string
* Configure Atlas Network Access appropriately
* Ensure the EC2 instance can establish an outbound connection to MongoDB Atlas

The MongoDB connection string must be provided through an environment variable.

Example:

```env
MONGODB_URI=<your-mongodb-connection-string>
```

**Never commit the actual connection string or database credentials to GitHub.**

## 8. Environment Variables

Prepare all environment variables required by the backend and frontend.

Example:

```env
PORT=5000
MONGODB_URI=<your-mongodb-connection-string>
JWT_SECRET=<your-secret>
```

Use an `.env` file locally or another secure configuration mechanism during deployment.

The actual `.env` file should be excluded from version control:

```gitignore
.env
.env.*
!.env.example
```

## 9. SSH Access

An EC2 key pair is required to connect to the instance.

Example:

```bash
ssh -i <key-file>.pem ubuntu@<EC2-PUBLIC-IP>
```

The username depends on the AMI used by the EC2 instance.

For example:

* Ubuntu → `ubuntu`
* Amazon Linux → `ec2-user`

The private key must be stored securely and should never be committed to the repository.

## 10. Pre-Deployment Checklist

Before proceeding with the deployment, verify:

* [ ] Application works locally
* [ ] Frontend Docker image builds successfully
* [ ] Backend Docker image builds successfully
* [ ] Containers run successfully in the local environment
* [ ] Docker networking has been tested
* [ ] AWS CLI is configured
* [ ] EC2 instance is available
* [ ] EC2 Security Group is configured
* [ ] SSH access to EC2 is working
* [ ] Docker is installed on EC2
* [ ] MongoDB Atlas cluster is available
* [ ] MongoDB connection string is available
* [ ] Required environment variables are prepared
* [ ] No credentials or secrets are present in the Git repository

## Next Step

After completing these prerequisites, the application can be prepared for containerized deployment.

Continue with:

**[02 — Dockerization](./02-dockerization.md)**
