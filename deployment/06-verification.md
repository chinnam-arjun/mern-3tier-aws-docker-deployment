# 06 — Deployment Verification

Deployment verification confirms that the containerized application is running correctly on AWS EC2 and that communication between all application layers is functioning as expected.

The verification process is performed from the infrastructure layer toward the application layer.

---

## 1. Verification Architecture

```text
                    AWS EC2
                       │
                Docker Engine
                       │
              ┌────────┴────────┐
              │                 │
              ▼                 ▼
       Frontend Container  Backend Container
              │                 │
              │                 ▼
              │           MongoDB Atlas
              │
              ▼
             User
```

The verification process checks:

```text
EC2
 ↓
Docker
 ↓
Containers
 ↓
Container Network
 ↓
Backend API
 ↓
MongoDB Atlas
 ↓
Frontend
 ↓
End-to-End Application
```

---

# 2. EC2 Instance Verification

First, verify that the EC2 instance is running and reachable.

From the local machine:

```bash
ssh -i <key-file>.pem <user>@<EC2-PUBLIC-IP>
```

Once connected, verify the system:

```bash
hostname
```

Check system resources:

```bash
free -h
df -h
```

These checks help confirm that the instance has sufficient memory and storage for the deployment.

---

# 3. Docker Service Verification

Verify that Docker is installed:

```bash
docker --version
```

Check the Docker service:

```bash
sudo systemctl status docker
```

The Docker service should be in a running state.

Verify the Docker daemon:

```bash
docker info
```

If Docker is functioning correctly, the Docker Engine information should be displayed.

---

# 4. Container Verification

List running containers:

```bash
docker ps
```

Expected architecture:

```text
CONTAINER
│
├── frontend
│
└── backend
```

Verify all containers, including stopped containers:

```bash
docker ps -a
```

This is important because a container may have started and then exited due to an application error.

---

# 5. Container Status

A healthy deployment should show the required containers in the running state.

Conceptually:

```text
Frontend Container
      │
      ▼
    RUNNING

Backend Container
      │
      ▼
    RUNNING
```

If a container repeatedly exits:

```bash
docker ps -a
```

Then inspect its logs:

```bash
docker logs <container-name>
```

---

# 6. Frontend Verification

The frontend is the public-facing application layer.

Verify the frontend container:

```bash
docker ps
```

Check its logs:

```bash
docker logs frontend
```

Verify the port mapping:

```bash
docker port frontend
```

Expected conceptual mapping:

```text
EC2 Port 80
     │
     ▼
Frontend Container Port 80
```

---

# 7. Test Frontend from EC2

Test the frontend directly from the EC2 instance:

```bash
curl http://localhost
```

A successful response indicates that the frontend web server is reachable from the EC2 host.

This helps distinguish between:

```text
Application Problem
```

and:

```text
External Network Problem
```

---

# 8. Test Frontend Externally

From a browser or external machine, access:

```text
http://<EC2-PUBLIC-IP>
```

Expected flow:

```text
Browser
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
   │
   ▼
React Application
```

If the application loads successfully, the public frontend path is functioning.

---

# 9. Backend Verification

Verify the backend container:

```bash
docker ps
```

Check backend logs:

```bash
docker logs backend
```

A successful backend startup should show that the Express server is listening on the configured port.

For example:

```text
Server running on port 5000
```

The exact log message depends on the application.

---

# 10. Backend Port Verification

Check the backend port mapping:

```bash
docker port backend
```

Also check listening ports on EC2:

```bash
ss -tulpn
```

This helps verify that the expected service is listening.

---

# 11. Test Backend API

Test the API locally from EC2:

```bash
curl http://localhost:5000/<api-endpoint>
```

For example:

```bash
curl http://localhost:5000/api/health
```

The endpoint should return the expected HTTP response.

If the API responds correctly:

```text
EC2
 │
 ▼
Backend Container
 │
 ▼
Express API
```

is functioning.

---

# 12. MongoDB Atlas Verification

The backend must successfully connect to MongoDB Atlas.

Check backend logs:

```bash
docker logs backend
```

Look for the application's database connection status.

Expected conceptual flow:

```text
Backend Container
       │
       ▼
MongoDB Connection
       │
       ▼
MongoDB Atlas
       │
       ▼
Successful Authentication
```

If the database connection fails, check:

* MongoDB connection string
* Database credentials
* MongoDB Atlas Network Access
* EC2 outbound connectivity
* DNS resolution
* Environment variables

---

# 13. Docker Network Verification

List Docker networks:

```bash
docker network ls
```

Inspect the application network:

```bash
docker network inspect <network-name>
```

Verify that the required containers are attached:

```text
Application Network
│
├── frontend
└── backend
```

This confirms that the containers are participating in the expected Docker network.

---

# 14. Frontend → Backend Verification

This is one of the most important checks.

The frontend must be able to communicate with the backend API.

Expected flow:

```text
Browser
   │
   ▼
Frontend
   │
   │ API Request
   ▼
Backend
   │
   ▼
API Response
   │
   ▼
Frontend
```

Test an application feature that requires backend communication.

Examples:

* User login
* User registration
* Fetching posts
* Creating a post
* Updating data
* Fetching dashboard information

A successful operation confirms that the application layers are communicating correctly.

---

# 15. Backend → Database Verification

Test an operation that requires database access.

For example:

```text
Frontend
   │
   ▼
API Request
   │
   ▼
Backend
   │
   ▼
MongoDB Query
   │
   ▼
MongoDB Atlas
   │
   ▼
Database Response
   │
   ▼
Backend
   │
   ▼
Frontend
```

Successful database operations confirm that the complete application data path is working.

---

# 16. End-to-End Verification

The final verification should test the application as an actual user would.

Example:

```text
User opens application
        ↓
Frontend loads
        ↓
User performs login
        ↓
Frontend sends API request
        ↓
Backend processes request
        ↓
Backend queries MongoDB Atlas
        ↓
MongoDB returns data
        ↓
Backend returns API response
        ↓
Frontend updates UI
```

This verifies all three application tiers together.

---

# 17. HTTP Verification

Use HTTP status codes to identify application behavior.

Typical responses:

| Status | Meaning                 |
| -----: | ----------------------- |
|    200 | Request successful      |
|    201 | Resource created        |
|    400 | Invalid request         |
|    401 | Authentication required |
|    403 | Access denied           |
|    404 | Resource not found      |
|    500 | Server-side error       |

When troubleshooting an API, the HTTP status code provides an initial indication of where the problem may exist.

---

# 18. Container Logs

Logs are one of the primary troubleshooting tools.

Frontend:

```bash
docker logs frontend
```

Backend:

```bash
docker logs backend
```

Follow logs continuously:

```bash
docker logs -f backend
```

Display recent logs:

```bash
docker logs --tail 100 backend
```

Logs can reveal:

* Application startup failures
* Missing environment variables
* Database connection errors
* Port conflicts
* Runtime exceptions
* Dependency problems

---

# 19. Resource Verification

Docker containers share the resources of the EC2 instance.

Check container resource usage:

```bash
docker stats
```

This displays information such as:

* CPU usage
* Memory usage
* Network usage
* Block I/O

This is particularly important on small EC2 instance types where memory or CPU constraints can cause containers to become unstable.

---

# 20. Deployment Health Checklist

### Infrastructure

* [ ] EC2 instance is running
* [ ] SSH connection works
* [ ] Internet connectivity works
* [ ] Security Group is configured

### Docker

* [ ] Docker Engine is running
* [ ] Frontend container is running
* [ ] Backend container is running
* [ ] No unexpected stopped containers

### Frontend

* [ ] Frontend container responds
* [ ] Nginx/web server is running
* [ ] Port mapping is correct
* [ ] Application loads externally

### Backend

* [ ] Backend container is running
* [ ] Express server starts successfully
* [ ] API endpoint responds
* [ ] No application errors in logs

### Database

* [ ] MongoDB Atlas is reachable
* [ ] Credentials are valid
* [ ] Network access is configured
* [ ] Database operations succeed

### Networking

* [ ] Docker network exists
* [ ] Containers are attached
* [ ] Frontend can reach backend
* [ ] Backend can reach MongoDB Atlas

### End-to-End

* [ ] Login works
* [ ] API requests work
* [ ] Database operations work
* [ ] Application behaves correctly from the browser

---

# 21. Verification Workflow

The complete verification process can be represented as:

```text
              EC2 Verification
                     │
                     ▼
              Docker Verification
                     │
                     ▼
             Container Verification
                     │
                     ▼
              Network Verification
                     │
             ┌───────┴────────┐
             ▼                ▼
        Frontend           Backend
             │                │
             │                ▼
             │          MongoDB Atlas
             │                │
             └───────┬────────┘
                     ▼
              End-to-End Test
                     │
                     ▼
             Deployment Verified
```

---

# 22. Successful Deployment Criteria

The deployment can be considered successful when:

```text
EC2
 │
 ├── Docker Running
 │
 ├── Frontend Container ── RUNNING
 │
 └── Backend Container ─── RUNNING
              │
              ▼
       MongoDB Atlas ─────── CONNECTED
```

and:

```text
User
 │
 ▼
Frontend
 │
 ▼
Backend API
 │
 ▼
MongoDB Atlas
 │
 ▼
Response
 │
 ▼
Frontend
 │
 ▼
User
```

works successfully without unexpected errors.

---

# Outcome

At the end of this verification stage, the complete deployment has been validated across the infrastructure, container, networking, API, and database layers.

The final deployment state is:

```text
                         INTERNET
                            │
                            ▼
                     AWS EC2 INSTANCE
                            │
                     Docker Engine
                            │
                  ┌─────────┴─────────┐
                  │                   │
                  ▼                   ▼
             FRONTEND              BACKEND
             CONTAINER             CONTAINER
                  │                   │
                  │       API         │
                  └───────────────────┘
                                      │
                                      │ Database
                                      ▼
                              MONGODB ATLAS
```

The application is now considered successfully deployed and verified.

---

## Next Documentation

After deployment verification, the next important area is documenting **real troubleshooting scenarios encountered during deployment**.

Recommended troubleshooting topics:

* Backend API not reachable
* Container running but application inaccessible
* Incorrect Docker port mapping
* EC2 Security Group blocking traffic
* Frontend using an incorrect backend API URL
* Backend unable to connect to MongoDB Atlas
* Environment variable configuration problems
* Docker container startup failures
* Application listening on the wrong interface
* Container-to-container communication issues
