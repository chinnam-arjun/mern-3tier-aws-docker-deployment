# 05 — Container Deployment

This stage covers the process of deploying the containerized application to the AWS EC2 instance.

The application images have already been built and tested. The EC2 instance has been prepared with Docker, networking has been configured, and MongoDB Atlas is available as the external data tier.

---

## 1. Deployment Architecture

The deployment target is:

```text
                    AWS EC2
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
       │ React/Nginx │   │ Node/Express│
       └─────────────┘   └──────┬──────┘
                                │
                                ▼
                         MongoDB Atlas
```

The deployment process is:

```text
Docker Images
     │
     ▼
Transfer / Pull Images
     │
     ▼
Configure Environment
     │
     ▼
Create Docker Network
     │
     ▼
Start Backend Container
     │
     ▼
Start Frontend Container
     │
     ▼
Verify Connectivity
     │
     ▼
Verify Application
```

---

# 2. Deployment Methods

Container images can be made available on EC2 using different approaches.

### Method 1 — Docker Registry

```text
Local Machine
     │
     ▼
Docker Image
     │
     ▼
Container Registry
     │
     ▼
AWS EC2
     │
     ▼
Docker Container
```

For AWS deployments, **Amazon ECR** can be used as the container registry.

### Method 2 — Direct Image Transfer

Images can also be transferred directly to the EC2 instance.

```text
Local Machine
     │
     ▼
Docker Image
     │
     ▼
EC2 Instance
     │
     ▼
Docker Container
```

The deployment method should be selected based on the project requirements.

---

# 3. Verify Docker Environment

Before deploying the images, verify that Docker is running on EC2.

```bash
docker --version
```

Check the Docker service:

```bash
sudo systemctl status docker
```

Check existing containers:

```bash
docker ps
```

Check available images:

```bash
docker images
```

The server should be ready before proceeding.

---

# 4. Make Images Available on EC2

The frontend and backend images must be available to the Docker Engine running on EC2.

Conceptually:

```text
Frontend Image
      │
      ▼
EC2 Docker Image Store

Backend Image
      │
      ▼
EC2 Docker Image Store
```

If using Amazon ECR:

```text
Local Docker Image
       │
       ▼
Amazon ECR Repository
       │
       ▼
EC2 pulls image
       │
       ▼
Docker Image
```

Verify the images:

```bash
docker images
```

Expected:

```text
REPOSITORY          TAG       IMAGE ID
frontend            latest    ...
backend             latest    ...
```

---

# 5. Configure Environment Variables

The backend requires environment-specific configuration.

Typical variables include:

```env
PORT=5000
MONGODB_URI=<mongodb-connection-string>
JWT_SECRET=<secret>
```

The actual values should be supplied securely during deployment.

Do not place production secrets directly inside:

* Dockerfiles
* Git repositories
* README files
* Public screenshots
* Docker image layers

---

# 6. Create Docker Network

Create a dedicated Docker network for application containers.

Conceptually:

```text
                  application-network
                         │
                ┌────────┴────────┐
                │                 │
                ▼                 ▼
         Frontend Container  Backend Container
```

Example:

```bash
docker network create application-network
```

Verify:

```bash
docker network ls
```

Inspect the network:

```bash
docker network inspect application-network
```

---

# 7. Deploy Backend Container

The backend should be started before the frontend because it provides the API consumed by the frontend.

Conceptually:

```text
Backend Docker Image
        │
        ▼
Backend Container
        │
        ├── Environment Variables
        ├── Port Configuration
        └── Docker Network
```

Example deployment pattern:

```bash
docker run -d \
  --name backend \
  --network application-network \
  --env-file .env \
  -p 5000:5000 \
  <backend-image>
```

The exact command should match the application's configuration.

Verify:

```bash
docker ps
```

---

# 8. Verify Backend Container

Check the container:

```bash
docker ps
```

Check logs:

```bash
docker logs backend
```

Follow logs in real time:

```bash
docker logs -f backend
```

The backend should successfully:

* Start the Node.js process
* Start the Express server
* Listen on the configured port
* Establish a connection to MongoDB Atlas

If any of these fail, investigate the logs before continuing.

---

# 9. Test Backend API

Before starting the frontend, verify the backend independently.

From the EC2 host:

```bash
curl http://localhost:5000
```

Or test an appropriate API endpoint:

```bash
curl http://localhost:5000/<api-endpoint>
```

The exact endpoint depends on the application.

This isolates backend problems from frontend problems.

---

# 10. Deploy Frontend Container

Once the backend is verified, start the frontend container.

Conceptually:

```text
Frontend Docker Image
        │
        ▼
Frontend Container
        │
        ├── Nginx
        ├── React Production Build
        └── Docker Network
```

Example deployment pattern:

```bash
docker run -d \
  --name frontend \
  --network application-network \
  -p 80:80 \
  <frontend-image>
```

Verify:

```bash
docker ps
```

---

# 11. Verify Frontend Container

Check frontend logs:

```bash
docker logs frontend
```

Inspect the container:

```bash
docker inspect frontend
```

Verify the exposed port:

```bash
docker port frontend
```

Expected mapping:

```text
80/tcp → 0.0.0.0:80
```

The exact mapping depends on the Docker configuration.

---

# 12. Container Communication

The frontend and backend should now be connected through the Docker network.

```text
              application-network
                      │
             ┌────────┴────────┐
             │                 │
             ▼                 ▼
      Frontend Container  Backend Container
             │                 │
             │ HTTP API        │
             └─────────────────┘
```

The frontend sends API requests to the backend.

The backend communicates with MongoDB Atlas.

```text
Frontend
    │
    │ API Request
    ▼
Backend
    │
    │ Database Request
    ▼
MongoDB Atlas
```

---

# 13. Verify Container Network

Check the network:

```bash
docker network inspect application-network
```

Confirm that both containers appear as members of the network.

Expected conceptual result:

```text
application-network
│
├── frontend
└── backend
```

This confirms that Docker networking has been established.

---

# 14. Verify External Access

After both containers are running, test the application from an external machine.

Flow:

```text
User Browser
     │
     ▼
EC2 Public IP
     │
     ▼
Security Group
     │
     ▼
Port 80
     │
     ▼
Frontend Container
```

Open:

```text
http://<EC2-PUBLIC-IP>
```

The frontend should load.

---

# 15. Complete Application Flow

A successful deployment should produce the following request path:

```text
                         USER
                          │
                          ▼
                    EC2 Public IP
                          │
                          ▼
                   Security Group
                          │
                          ▼
                  Frontend Container
                          │
                          │ API Request
                          ▼
                   Backend Container
                          │
                          │ Database Query
                          ▼
                   MongoDB Atlas
                          │
                          │ Response
                          ▼
                   Backend Container
                          │
                          │ API Response
                          ▼
                  Frontend Container
                          │
                          ▼
                         USER
```

---

# 16. Container Health Checks

The deployment should be validated at multiple levels.

### Docker Level

```bash
docker ps
```

Confirm that both containers are running.

### Application Level

```bash
docker logs frontend
docker logs backend
```

Look for startup errors or repeated crashes.

### Network Level

```bash
docker network inspect application-network
```

Confirm container connectivity.

### HTTP Level

```bash
curl http://localhost:<port>
```

Verify that the expected service responds.

### Browser Level

Access the application using:

```text
http://<EC2-PUBLIC-IP>
```

---

# 17. Container Lifecycle Management

Useful commands during deployment:

### Stop a container

```bash
docker stop <container-name>
```

### Start a stopped container

```bash
docker start <container-name>
```

### Restart a container

```bash
docker restart <container-name>
```

### Remove a container

```bash
docker rm <container-name>
```

### View all containers

```bash
docker ps -a
```

These commands are useful when troubleshooting or updating the deployment.

---

# 18. Updating the Deployment

When a new application version is available, the typical container deployment workflow is:

```text
New Application Version
          │
          ▼
Build New Docker Image
          │
          ▼
Tag Image
          │
          ▼
Push / Transfer Image
          │
          ▼
Stop Existing Container
          │
          ▼
Remove Old Container
          │
          ▼
Start New Container
          │
          ▼
Verify Application
```

For a production environment, this process can later be automated using CI/CD.

---

# 19. Deployment Verification Checklist

* [ ] Frontend image available on EC2
* [ ] Backend image available on EC2
* [ ] Docker network created
* [ ] Backend container running
* [ ] Backend logs verified
* [ ] Backend API tested
* [ ] Frontend container running
* [ ] Frontend logs verified
* [ ] Containers connected to the expected Docker network
* [ ] MongoDB Atlas connection verified
* [ ] EC2 Security Group configured
* [ ] Frontend accessible from the internet
* [ ] Frontend can communicate with the backend
* [ ] Backend can communicate with MongoDB Atlas

---

# 20. Deployment Outcome

At the end of this stage, the application is running as Docker containers on AWS EC2:

```text
AWS EC2
│
└── Docker Engine
    │
    ├── Frontend Container
    │
    └── Backend Container
            │
            ▼
       MongoDB Atlas
```

The application is now deployed and accessible through the EC2 instance.

The next stage is to systematically verify the deployment and confirm that every application layer is functioning correctly.

---

## Next Step

Proceed to:

**[06 — Deployment Verification](./06-verification.md)**

This section covers:

* Container verification
* API verification
* Frontend verification
* Database connectivity
* End-to-end request testing
* Deployment health checks
