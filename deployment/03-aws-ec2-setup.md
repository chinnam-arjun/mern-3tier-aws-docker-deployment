# 03 — AWS EC2 Setup

After successfully containerizing and testing the application locally, the next step is preparing an AWS EC2 instance to act as the Docker host.

This EC2 instance will run the frontend and backend containers while communicating with MongoDB Atlas as the external data tier.

---

## Objective

Provision and configure an EC2 instance capable of:

* Hosting Docker containers
* Running the frontend application
* Running the backend API
* Communicating with MongoDB Atlas
* Accepting traffic from the internet
* Providing a secure and manageable deployment environment

---

## Target Architecture

```text
Internet
    │
    ▼
AWS EC2 Instance
    │
Docker Engine
    │
├── Frontend Container
│
└── Backend Container
          │
          ▼
    MongoDB Atlas
```

The EC2 instance acts as the runtime environment for all application containers.

---

## 1. Launch an EC2 Instance

Create a new EC2 instance in your preferred AWS region.

Recommended specifications for a small portfolio project:

| Configuration    | Recommendation            |
| ---------------- | ------------------------- |
| Operating System | Ubuntu Server 24.04 LTS   |
| Instance Type    | t2.micro / t3.micro       |
| Storage          | 20 GB gp3                 |
| Network          | Default VPC or Custom VPC |
| Public IP        | Enabled                   |

The selected instance should provide:

* SSH access
* Internet access
* Sufficient resources for Docker containers
* Outbound connectivity to MongoDB Atlas

---

## 2. Create or Select a Key Pair

A key pair is required to securely connect to the instance.

AWS generates:

* Public key
* Private key (`.pem`)

Store the private key securely.

Example:

```text
my-key.pem
```

Never:

* Upload the key to GitHub
* Share the key publicly
* Store it inside project repositories

---

## 3. Configure Security Groups

The Security Group acts as the instance firewall.

Only required ports should be opened.

### Inbound Rules

| Port | Protocol | Purpose                  |
| ---- | -------- | ------------------------ |
| 22   | TCP      | SSH Access               |
| 80   | TCP      | HTTP Traffic             |
| 443  | TCP      | HTTPS Traffic (Optional) |
| 5000 | TCP      | Backend API (Optional)   |

Example:

```text
Internet
    │
    ▼
Security Group
    │
    ├── Port 22
    ├── Port 80
    ├── Port 443
    └── Port 5000
```

For production environments, backend ports should generally remain private.

---

## 4. Connect to the Instance

After the instance is running, connect using SSH.

Example workflow:

```text
Local Machine
      │
      ▼
SSH Connection
      │
      ▼
AWS EC2 Instance
```

Verify connectivity before proceeding with the deployment configuration.

---

## 5. Update the Operating System

Before installing software, update the package repositories and system packages.

This ensures:

* Latest package metadata
* Security patches
* Updated dependencies

Typical process:

```text
EC2 Instance
      │
      ▼
Update Package Index
      │
      ▼
Upgrade Installed Packages
      │
      ▼
Ready for Software Installation
```

Keeping the system updated reduces compatibility and security issues.

---

## 6. Install Docker Engine

Docker Engine is required to run application containers.

Installation objectives:

* Install Docker Engine
* Enable Docker service
* Verify Docker functionality

Architecture:

```text
EC2 Instance
      │
      ▼
Docker Engine
      │
      ▼
Containers
```

After installation, verify:

```text
Docker Installed
Docker Service Running
Docker Commands Available
```

---

## 7. Configure Docker Service

Docker should automatically start when the EC2 instance boots.

Desired state:

```text
EC2 Reboot
     │
     ▼
Docker Starts Automatically
     │
     ▼
Containers Can Be Started
```

Benefits:

* Improved availability
* Faster recovery
* Simplified administration

---

## 8. Verify Docker Installation

Before deploying containers, verify Docker functionality.

Checks include:

### Docker Version

```text
Docker Engine Available
```

### Docker Service

```text
Docker Daemon Running
```

### Container Runtime

```text
Docker Can Create Containers
```

At this stage the instance should be capable of running application containers.

---

## 9. Configure User Permissions

Docker commands are often executed without requiring elevated privileges.

Recommended setup:

```text
EC2 User
      │
      ▼
Docker Group Membership
      │
      ▼
Execute Docker Commands
```

Benefits:

* Improved usability
* Reduced administrative overhead
* Simplified deployment workflow

---

## 10. Verify Internet Connectivity

The instance requires outbound internet access for:

* Pulling container images
* Installing packages
* Connecting to MongoDB Atlas
* Accessing external services

Connectivity path:

```text
EC2 Instance
      │
      ▼
Internet Gateway
      │
      ▼
Internet
```

Without outbound connectivity:

* Image downloads fail
* Package installation fails
* Atlas connectivity fails

---

## 11. Verify MongoDB Atlas Reachability

The backend container must communicate with MongoDB Atlas.

Connection path:

```text
Backend Container
        │
        ▼
EC2 Network
        │
        ▼
Internet
        │
        ▼
MongoDB Atlas
```

Validation goals:

* DNS resolution works
* Outbound traffic works
* Atlas access rules are configured properly

---

## 12. Prepare Deployment Directory Structure

Create a clean deployment workspace on the EC2 instance.

Example structure:

```text
deployment/
│
├── frontend/
├── backend/
├── docker/
├── logs/
└── configs/
```

Purpose:

* Better organization
* Easier maintenance
* Cleaner deployments

---

## 13. Transfer Deployment Assets

The deployment requires the following resources:

### Frontend Image

Contains:

* React application
* Production build
* Web server configuration

### Backend Image

Contains:

* Node.js runtime
* Express application
* API services

### Configuration Files

Contains:

* Environment configuration
* Runtime settings
* Deployment parameters

Deployment assets may arrive through:

```text
Git Repository
Docker Registry
Manual Transfer
CI/CD Pipeline
```

The transfer mechanism depends on the deployment strategy.

---

## 14. EC2 Validation Checklist

Before deploying containers, verify:

### Infrastructure

* [ ] EC2 instance running
* [ ] Public IP assigned
* [ ] Security Group configured
* [ ] SSH access verified

### Operating System

* [ ] System updated
* [ ] Required packages installed

### Docker

* [ ] Docker installed
* [ ] Docker daemon running
* [ ] Docker accessible to deployment user

### Network

* [ ] Internet access available
* [ ] DNS resolution working
* [ ] MongoDB Atlas reachable

### Deployment Assets

* [ ] Frontend image available
* [ ] Backend image available
* [ ] Environment configuration prepared

---

## Common Issues

### SSH Connection Failure

Possible causes:

* Incorrect key pair
* Wrong public IP
* Security Group blocking port 22
* Incorrect username

---

### Docker Installation Issues

Possible causes:

* Outdated package repositories
* Missing dependencies
* Incomplete installation

---

### Internet Connectivity Problems

Possible causes:

* Missing Internet Gateway
* Route table misconfiguration
* Security Group restrictions
* Network ACL restrictions

---

### MongoDB Atlas Connection Failure

Possible causes:

* Atlas IP restrictions
* Invalid connection string
* DNS issues
* Firewall restrictions

---

## Outcome

At the end of this phase:

* AWS EC2 is provisioned
* Docker Engine is installed
* Network connectivity is verified
* Security Groups are configured
* MongoDB Atlas is reachable
* The server is ready to host containers

The environment is now prepared for application deployment.

---

## Next Step

Proceed to:

**[04 — Networking Configuration](./04-networking.md)**

This section covers:

* Security Group design
* Port mapping
* Container communication
* Frontend ↔ Backend networking
* Backend ↔ MongoDB Atlas connectivity
* Traffic flow analysis
