# Containerized MERN 3-Tier Application Deployment on AWS EC2

## Overview

This repository documents the process of containerizing and deploying a MERN (MongoDB, Express, React, Node.js) 3-tier application on AWS EC2 using Docker.

It covers the architecture, containerization process, AWS infrastructure setup, networking configuration, deployment workflow, real-world troubleshooting scenarios, and security considerations involved in taking a MERN application from source code to a running, publicly accessible deployment on AWS.

This is a **deployment and DevOps documentation repository** — it does not contain the application's source code. The focus here is on _how the application was containerized, deployed, configured, tested, and troubleshot_, not on the application logic itself.

---

## Original Application

The application source code is maintained in a separate repository.

- **Application:** [APPLICATION NAME]
- **Repository:** [LINK TO YOUR MERN APPLICATION REPOSITORY]
- **Stack:** React (frontend), Node.js/Express (backend), MongoDB Atlas (database)

This repository focuses exclusively on the containerization and AWS deployment process for that application.

---

## Architecture

```
                         INTERNET
                             │
                             ▼
                    ┌─────────────────┐
                    │    AWS EC2      │
                    │  Docker Host    │
                    │                 │
                    │ ┌─────────────┐ │
                    │ │   React     │ │
                    │ │  Container  │ │
                    │ └──────┬──────┘ │
                    │        │        │
                    │        ▼        │
                    │ ┌─────────────┐ │
                    │ │ Node/Express│ │
                    │ │  Container  │ │
                    │ └──────┬──────┘ │
                    └────────┼────────┘
                             │
                             │ HTTPS
                             ▼
                    ┌─────────────────┐
                    │ MongoDB Atlas   │
                    │  Data Tier      │
                    └─────────────────┘
```

### Presentation Tier — React

- Serves the user interface
- Sends API requests to the backend
- Handles client-side routing

### Application Tier — Node.js + Express

- Exposes REST APIs
- Handles authentication and business logic
- Communicates with the database

### Data Tier — MongoDB Atlas

- Stores persistent application data
- Managed, externally hosted database service

---

## Technology Stack

| Layer                 | Technology          |
| --------------------- | ------------------- |
| Frontend              | React               |
| Backend               | Node.js, Express    |
| Database              | MongoDB Atlas       |
| Containerization      | Docker              |
| Compute               | AWS EC2             |
| Networking / Security | AWS Security Groups |

---

## Container Architecture

```
                 EC2 INSTANCE
                      │
                 Docker Engine
                      │
             ┌────────┴────────┐
             │                 │
             ▼                 ▼
      ┌─────────────┐   ┌─────────────┐
      │   Frontend  │   │   Backend   │
      │  Container  │   │  Container  │
      │             │   │             │
      │ React       │   │ Node        │
      │ Web Server  │   │ Express     │
      └─────────────┘   └──────┬──────┘
                               │
                               ▼
                        MongoDB Atlas
```

The frontend and backend are isolated into separate containers. Docker provides the runtime environment and internal networking required for the containers to communicate with each other and with the outside world.

---

## Deployment Workflow

```
Application Repository
          │
          ▼
     Dockerization
          │
          ▼
    Build Images
          │
          ▼
 Local Container Testing
          │
          ▼
      AWS EC2 Setup
          │
          ▼
    Install Docker
          │
          ▼
 Transfer / Pull Images
          │
          ▼
   Start Containers
          │
          ▼
 Configure Networking
          │
          ▼
 Connect MongoDB Atlas
          │
          ▼
 Deployment Verification
```

---

## Step 1 — Application Preparation

- [ ] Reviewed application structure (frontend/backend separation)
- [ ] Identified environment variables required by each service
- [ ] Confirmed MongoDB Atlas connection string and network access rules

---

## Step 2 — Dockerization

The application was split into independently containerized components.

**Frontend:**

```
React application → Production build → Web server → Docker image → Frontend container
```

**Backend:**

```
Node.js application → Dependency installation → Express server → Docker image → Backend container
```

Topics covered:

- Base images used for frontend and backend
- Multi-stage Dockerfile builds (if used)
- Dependency installation strategy
- Port exposure (`EXPOSE`)
- `.dockerignore` configuration
- Image build commands
- Container run commands

> See `deployment/02-dockerization.md` for the full write-up and commands used.

---

## Step 3 — Local Container Testing

- Built images locally with `docker build`
- Ran containers locally with `docker run` / `docker compose`
- Verified frontend–backend communication before deploying to AWS

---

## Step 4 — AWS EC2 Setup

```
EC2 Instance
     │
     ├── Operating System
     ├── Instance type
     ├── Security Group
     ├── Public IP
     └── Docker Engine
```

Steps followed:

1. Launched EC2 instance (free-tier eligible)
2. Configured Security Group (inbound/outbound rules)
3. Connected via SSH
4. Updated system packages
5. Installed Docker Engine
6. Verified Docker installation (`docker --version`, `docker run hello-world`)
7. Prepared the deployment environment (env variables, directories)

---

## Step 5 — Networking & Security Groups

```
Internet
   │
   ▼
EC2 Public IP
   │
   ▼
Security Group
   │
   ▼
Docker Host
   │
   ├── Frontend Container
   │
   └── Backend Container
```

**Security Group rules:** which ports were opened and why (e.g. 22 for SSH, 80/3000 for frontend, 5000 for backend API).

**Docker port mapping:**

```
EC2 Port → Docker Host Port → Container Port
```

**Internal communication:** how the frontend container reaches the backend container (Docker network / container name resolution / internal IP).

**External database communication:** how the backend container reaches MongoDB Atlas over the internet (IP allowlisting, connection string, TLS).

> See `deployment/04-networking.md` for full details.

---

## Step 6 — Docker Deployment on EC2

- Transferred / pulled Docker images onto the EC2 instance
- Started containers with the correct port mappings and environment variables
- Verified containers were running (`docker ps`)

---

## Step 7 — MongoDB Atlas Connectivity

- Configured Atlas network access to allow connections from the EC2 instance
- Passed the MongoDB URI to the backend container via environment variables (never hardcoded)

---

## Step 8 — Deployment Verification

- Confirmed frontend was reachable via the EC2 public IP
- Confirmed frontend successfully called backend APIs
- Confirmed backend successfully read/wrote to MongoDB Atlas
- Checked container logs for errors (`docker logs <container>`)

---

## Troubleshooting

This section documents real issues encountered during deployment, not just the final working state.

### 1. Backend API Not Reachable from Frontend

**Symptom:** Frontend requests were not reaching the backend.

**Investigation:**

- Checked running containers (`docker ps`)
- Checked container logs (`docker logs <container>`)
- Checked exposed/mapped ports
- Checked EC2 Security Group inbound rules
- Checked the backend's listening address (`0.0.0.0` vs `localhost`)
- Checked the frontend's configured API base URL

**Resolution:** [Describe the actual fix — e.g. backend was bound to `localhost` instead of `0.0.0.0`, or the API base URL pointed to `localhost` instead of the EC2 public IP.]

---

### 2. EC2 Port Inaccessible from Outside

**Symptom:** Application worked when accessed from inside the EC2 instance but was unreachable externally.

**Investigation:**

- Verified Docker port mapping (`-p host:container`)
- Verified Security Group inbound rules for the relevant port
- Verified the application was listening on the correct port inside the container
- Confirmed the EC2 public IP being used

**Resolution:** [Describe the actual fix.]

---

### 3. Container Running but API Failing

**Investigation commands used:**

```bash
docker ps
docker logs <container>
docker exec -it <container> sh
```

**Root cause:** [Describe what was actually found — e.g. missing environment variable, wrong MongoDB URI, dependency not installed in image.]

**Resolution:** [Describe the actual fix.]

---

---

## Security Considerations

- No secrets or credentials committed to GitHub
- Environment variables used for all sensitive configuration
- MongoDB Atlas credentials excluded from the repository
- Security Group inbound rules restricted to only the required ports
- MongoDB Atlas network access restricted appropriately
- SSH access restricted where possible

Example of how credentials are referenced (never the real values):

```
MONGODB_URI=<your-mongodb-connection-string>
```

---

## Screenshots

| Screenshot                                 | Description                        |
| ------------------------------------------ | ---------------------------------- |
| `screenshots/architecture.png`             | High-level architecture            |
| `screenshots/ec2.png`                      | Running EC2 instance               |
| `screenshots/publishing-images-to-ecr.png` | pushing local docker images to ECR |
| `screenshots/docker-containers.png`        | Running containers on EC2          |
| `screenshots/docker-images.png`            | Built Docker images                |
| `screenshots/application.png`              | Deployed application in browser    |

---

## Key Challenges

- [What was genuinely difficult — e.g. container networking, environment variable management, Security Group misconfiguration]

## Lessons Learned

- [What you'd do differently or now understand better as a result of this deployment]

---

## Future Improvements

- Add an Application Load Balancer
- Add HTTPS using AWS Certificate Manager (ACM)
- Move container deployment to Amazon ECS
- Introduce CI/CD using GitHub Actions
- Push images to Amazon ECR instead of manual transfer
- Add Auto Scaling
- Add CloudWatch monitoring and alarms
- Use AWS Secrets Manager for sensitive configuration
- Introduce Terraform for infrastructure as code
- Separate frontend and backend into distinct networks/subnets

---

## Repository Structure

```
mern-3tier-aws-docker-deployment/
│
├── README.md
│
├── architecture/
│   ├── architecture.png
│   ├── network-flow.png
│   └── container-flow.png
│
├── deployment/
│   ├── 01-prerequisites.md
│   ├── 02-dockerization.md
│   ├── 03-aws-ec2-setup.md
│   ├── 04-networking.md
│   ├── 05-container-deployment.md
│   └── 06-verification.md
│
├── troubleshooting/
│   ├── backend-connectivity.md
│   ├── container-networking.md
│   ├── security-group.md
│   ├── environment-variables.md
│   └── common-docker-issues.md
│
├── security/
│   └── security-considerations.md
│
├── screenshots/
│   ├── architecture.png
│   ├── ec2.png
│   ├── docker-containers.png
│   └── application.png
│

```

---
