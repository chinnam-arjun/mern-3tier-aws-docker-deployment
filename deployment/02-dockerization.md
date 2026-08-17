# 02 — Dockerization

Dockerization is the process of packaging the application components and their runtime dependencies into isolated, portable containers.

For this project, the MERN application is divided into two containerized application components:

* **Frontend container** — React application served through Nginx
* **Backend container** — Node.js + Express API

MongoDB is **not containerized** because the application uses **MongoDB Atlas** as the managed data tier.

---

## 1. Dockerization Architecture

The application follows this container architecture:

```text
                    Docker Host
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
       │ React       │   │ Node.js     │
       │ Nginx       │   │ Express     │
       │ Port 80     │   │ Port 5000   │
       └─────────────┘   └──────┬──────┘
                                │
                                ▼
                         MongoDB Atlas
```

The Docker containers provide isolated runtime environments while allowing the application components to communicate through networking.

---

## 2. Frontend Dockerization

The frontend is a React application.

The Dockerization process consists of:

```text
React Source Code
        │
        ▼
Install Dependencies
        │
        ▼
Create Production Build
        │
        ▼
Build Docker Image
        │
        ▼
Run Frontend Container
        │
        ▼
Serve Application Through Nginx
```

### Production Build

The React application is first converted into a production-ready static build.

The production build contains the optimized assets required by the browser.

These assets are then served by Nginx inside the frontend container.

### Why Nginx?

Nginx is used as the production web server because it is lightweight and well suited for serving static frontend assets.

The resulting container contains:

```text
Frontend Container
├── Nginx
├── React production build
└── Web server configuration
```

---

## 3. Backend Dockerization

The backend consists of Node.js and Express.

The process is:

```text
Node.js / Express Application
             │
             ▼
      Install Dependencies
             │
             ▼
        Build Image
             │
             ▼
      Run Backend Container
             │
             ▼
       Express API Server
             │
             ▼
       MongoDB Atlas
```

The backend container provides the runtime environment for the API.

The backend is responsible for:

* Processing API requests
* Authentication and authorization
* Application business logic
* Database operations
* Communication with MongoDB Atlas

---

## 4. Docker Images

A Docker image is created for each application component.

```text
Application
    │
    ├── Frontend
    │      └── Frontend Docker Image
    │
    └── Backend
           └── Backend Docker Image
```

Images contain the application and the dependencies required to run it.

Once an image has been built, the same image can be used across different environments, such as:

```text
Development
     ↓
Testing
     ↓
AWS EC2
```

This reduces differences between local and deployment environments.

---

## 5. Dockerfile Responsibilities

Each application component has its own Dockerfile.

### Frontend Dockerfile

The frontend Dockerfile is responsible for:

1. Selecting an appropriate build environment
2. Installing frontend dependencies
3. Creating the production build
4. Preparing the Nginx runtime environment
5. Copying the production assets into the web server
6. Starting Nginx

### Backend Dockerfile

The backend Dockerfile is responsible for:

1. Selecting the Node.js runtime
2. Installing backend dependencies
3. Copying the application into the image
4. Configuring the application runtime
5. Exposing the backend application port
6. Starting the Express server

The actual application source code and Dockerfiles are maintained in the separate application repository.

---

## 6. `.dockerignore`

A `.dockerignore` file is used to prevent unnecessary files from being included in the Docker build context.

Typical exclusions include:

```text
node_modules
.git
.env
*.log
```

This helps:

* Reduce build context size
* Reduce image build time
* Prevent unnecessary files from entering the image
* Reduce the risk of accidentally including secrets

---

## 7. Environment Configuration

Application configuration should not be hard-coded into Docker images.

Environment-specific values should be supplied when the container is started.

Examples include:

```text
Database connection string
API configuration
Application port
Authentication secrets
Environment mode
```

Conceptually:

```text
Docker Image
     │
     │
     ├── Application
     └── Dependencies

        +

Environment Configuration
        │
        ▼
Running Container
```

This allows the same image to be used in different environments without rebuilding it for every configuration change.

---

## 8. Building the Images

After preparing the Dockerfiles, build the frontend and backend images.

Example workflow:

```bash
docker build -t <frontend-image> <frontend-context>
docker build -t <backend-image> <backend-context>
```

Verify that the images were created:

```bash
docker images
```

Expected result:

```text
REPOSITORY          TAG       IMAGE ID
frontend            latest    ...
backend             latest    ...
```

The exact image names and tags depend on the deployment configuration.

---

## 9. Running Containers Locally

Before deploying to AWS, the containers should be tested locally.

The basic workflow is:

```text
Docker Images
     │
     ▼
Create Containers
     │
     ▼
Configure Ports
     │
     ▼
Configure Environment Variables
     │
     ▼
Configure Container Network
     │
     ▼
Test Application
```

Verify running containers:

```bash
docker ps
```

Inspect container logs when required:

```bash
docker logs <container-name>
```

---

## 10. Container Networking

The frontend and backend containers need to communicate with each other.

A Docker network can provide an isolated communication layer:

```text
             Docker Network
                   │
          ┌────────┴────────┐
          │                 │
          ▼                 ▼
   Frontend Container  Backend Container
          │                 │
          └─────── HTTP ────┘
```

The backend then establishes an outbound connection to MongoDB Atlas.

```text
Frontend
    │
    │ HTTP API request
    ▼
Backend
    │
    │ MongoDB connection
    ▼
MongoDB Atlas
```

---

## 11. Port Mapping

Container ports and host ports are separate concepts.

For example:

```text
EC2 Host Port 80
       │
       ▼
Frontend Container Port 80
```

For the backend:

```text
EC2 Host Port 5000
       │
       ▼
Backend Container Port 5000
```

Whether the backend port needs to be publicly exposed depends on the architecture.

If the backend is intended to be accessed only internally, it should not unnecessarily be exposed to the public internet.

---

## 12. Local Verification

Before moving to AWS, verify the following:

### Frontend

* Frontend container starts successfully
* Nginx starts successfully
* React production build loads
* Frontend port is accessible

### Backend

* Backend container starts successfully
* Express server starts successfully
* API endpoints respond correctly
* Backend logs do not contain startup errors

### Communication

* Frontend can reach the backend API
* Backend can connect to MongoDB Atlas
* Database operations work correctly

### Container State

```bash
docker ps
```

All required containers should be in the running state.

---

## 13. Dockerization Workflow

The complete Dockerization workflow is:

```text
Application Source
       │
       ▼
Create Dockerfiles
       │
       ▼
Create .dockerignore
       │
       ▼
Build Frontend Image
       │
       ▼
Build Backend Image
       │
       ▼
Run Containers
       │
       ▼
Configure Docker Network
       │
       ▼
Configure Environment Variables
       │
       ▼
Test Application Locally
       │
       ▼
Verify Container Logs
       │
       ▼
Ready for AWS Deployment
```

---

## 14. Key Docker Concepts Used

This deployment demonstrates the following Docker concepts:

* Docker images
* Docker containers
* Dockerfiles
* Docker build context
* `.dockerignore`
* Port mapping
* Environment variables
* Container networking
* Container logs
* Container lifecycle management
* Production builds
* Nginx-based frontend serving

---

## Next Step

After successfully building and testing the application containers locally, the next stage is preparing the **AWS EC2 environment** for deployment.

Continue with:

**[03 — AWS EC2 Setup](./03-aws-ec2-setup.md)**
