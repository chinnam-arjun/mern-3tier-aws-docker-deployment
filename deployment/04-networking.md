# 04 — Networking Configuration

Networking is responsible for controlling how traffic moves between the user, AWS EC2, Docker containers, and MongoDB Atlas.

This deployment uses AWS networking at the infrastructure level and Docker networking at the container level.

---

## 1. Network Architecture

The overall traffic flow is:

```text
                         INTERNET
                            │
                            │ HTTP
                            ▼
                    ┌───────────────┐
                    │  AWS EC2      │
                    │ Public IP     │
                    └───────┬───────┘
                            │
                      Security Group
                            │
                            ▼
                     Docker Network
                       │         │
                       ▼         ▼
                ┌──────────┐ ┌──────────┐
                │ Frontend │ │ Backend  │
                │ Container│ │ Container│
                └─────┬────┘ └────┬─────┘
                      │            │
                      │ HTTP/API   │ MongoDB
                      │            │ Connection
                      │            ▼
                      │     ┌──────────────┐
                      │     │ MongoDB Atlas│
                      │     └──────────────┘
                      │
                      ▼
                    User
```

The architecture contains two different networking layers:

```text
AWS Networking
      │
      ▼
EC2 Instance
      │
      ▼
Docker Networking
      │
      ├── Frontend Container
      └── Backend Container
```

---

# 2. AWS Network Layer

The EC2 instance operates inside an AWS VPC.

The basic network path is:

```text
Internet
   │
   ▼
Internet Gateway
   │
   ▼
Route Table
   │
   ▼
Subnet
   │
   ▼
EC2 Instance
```

The exact VPC configuration depends on the AWS environment used for the deployment.

For a public EC2 deployment, the instance needs a network path that allows inbound traffic from the internet and outbound connectivity to external services.

---

# 3. Public IP Address

The EC2 instance uses a public IP address to receive traffic from the internet.

Example:

```text
User
 │
 │ HTTP Request
 ▼
EC2 Public IP
 │
 ▼
EC2 Instance
```

The public IP should not be confused with the private IP assigned inside the VPC.

### Public IP

Used for communication between the internet and the EC2 instance.

### Private IP

Used for communication within the AWS VPC.

---

# 4. Security Group

The EC2 Security Group controls inbound and outbound traffic associated with the instance.

Conceptually:

```text
                  Internet
                     │
                     ▼
             ┌────────────────┐
             │ Security Group │
             └───────┬────────┘
                     │
          ┌──────────┼──────────┐
          │          │          │
          ▼          ▼          ▼
        SSH        HTTP       HTTPS
        :22         :80        :443
                     │
                     ▼
                  EC2
```

Security Groups are stateful, meaning return traffic for an allowed connection is automatically permitted.

---

# 5. Required Ports

The ports exposed by the deployment should be limited to what is actually required.

| Port | Protocol | Purpose            | Public Access    |
| ---: | -------- | ------------------ | ---------------- |
|   22 | TCP      | SSH administration | Restricted       |
|   80 | TCP      | Frontend HTTP      | Yes              |
|  443 | TCP      | HTTPS              | If configured    |
| 5000 | TCP      | Backend API        | Only if required |

The backend port should preferably not be publicly exposed when the frontend can communicate with it through an internal Docker network.

---

# 6. Docker Network Layer

Once traffic reaches the EC2 host, Docker controls communication between containers.

The containers can be connected through a Docker bridge network.

```text
             Docker Host
                  │
            Docker Network
                  │
        ┌─────────┴─────────┐
        │                   │
        ▼                   ▼
  Frontend Container   Backend Container
        │                   │
        │ HTTP/API          │
        └───────────────────┘
```

This provides isolated communication between application components.

---

# 7. Frontend → Backend Flow

When a user performs an action that requires backend data, the frontend sends an API request.

Flow:

```text
User
 │
 ▼
React Application
 │
 │ HTTP Request
 ▼
Backend Container
 │
 ▼
Express API
```

For example:

```text
GET /api/users
POST /api/auth/login
GET /api/posts
```

The frontend should communicate with the backend using the configured API endpoint.

---

# 8. Backend → MongoDB Atlas Flow

The backend communicates with MongoDB Atlas over the internet.

```text
Backend Container
       │
       ▼
EC2 Network
       │
       ▼
Internet Gateway
       │
       ▼
Internet
       │
       ▼
MongoDB Atlas
```

The backend establishes the database connection using the MongoDB connection string supplied through environment configuration.

---

# 9. MongoDB Atlas Network Access

MongoDB Atlas provides its own network access controls.

The connection must satisfy both:

```text
EC2 Outbound Connectivity
          +
MongoDB Atlas Network Access Rules
          │
          ▼
     Database Access
```

If Atlas does not permit the EC2 source address, the backend will not be able to establish the database connection even if the EC2 instance has internet access.

---

# 10. Port Mapping

Docker port mapping connects a host port to a container port.

Conceptually:

```text
EC2 Host
Port 80
   │
   ▼
Frontend Container
Port 80
```

For the backend:

```text
EC2 Host
Port 5000
   │
   ▼
Backend Container
Port 5000
```

Port mapping is represented as:

```text
HOST_PORT:CONTAINER_PORT
```

Example:

```text
80:80
5000:5000
```

The actual mappings should match the application's deployment configuration.

---

# 11. Binding Address

A common container networking problem occurs when an application listens only on the container's loopback interface.

For a server that needs to accept connections from outside the container, it should generally listen on:

```text
0.0.0.0
```

rather than:

```text
127.0.0.1
```

Conceptually:

```text
Incorrect:

Backend
   │
   └── 127.0.0.1
          │
          └── Accessible only inside container


Correct:

Backend
   │
   └── 0.0.0.0
          │
          └── Accepts connections through container networking
```

This is an important troubleshooting point when an application works inside a container but cannot be reached externally.

---

# 12. Complete Request Flow

The complete application request path is:

```text
                         USER
                          │
                          │ HTTP
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

# 13. Network Boundaries

There are several security boundaries in this architecture.

### Boundary 1 — Internet → AWS

Controlled primarily through:

* Security Groups
* VPC routing
* Network ACLs where applicable

### Boundary 2 — EC2 → Containers

Controlled through:

* Docker networking
* Port mappings
* Container configuration

### Boundary 3 — Backend → MongoDB Atlas

Controlled through:

* EC2 outbound connectivity
* MongoDB Atlas Network Access
* Database authentication

---

# 14. Public vs Private Components

The deployment should expose only the component that needs direct public access.

A simplified model is:

```text
PUBLIC
  │
  ▼
Frontend
  │
  ▼
PRIVATE
Backend
  │
  ▼
EXTERNAL MANAGED SERVICE
MongoDB Atlas
```

If the backend is directly exposed through a public port in the current implementation, that should be documented as a deployment decision and reviewed as a future security improvement.

---

# 15. Network Troubleshooting

When communication fails, troubleshoot from the outside inward.

Recommended sequence:

```text
1. Is the application running?
          ↓
2. Is the container running?
          ↓
3. Is the application listening on the expected port?
          ↓
4. Is the Docker port mapped correctly?
          ↓
5. Is the Docker network configured correctly?
          ↓
6. Is the EC2 Security Group allowing traffic?
          ↓
7. Is the EC2 route/network configuration correct?
          ↓
8. Is the external service reachable?
```

This prevents randomly changing configuration without identifying the actual failure point.

---

## 16. Useful Diagnostic Commands

Check running containers:

```bash
docker ps
```

Check container logs:

```bash
docker logs <container-name>
```

Inspect a container:

```bash
docker inspect <container-name>
```

List Docker networks:

```bash
docker network ls
```

Inspect a Docker network:

```bash
docker network inspect <network-name>
```

Check listening ports on the EC2 host:

```bash
ss -tulpn
```

Test an HTTP endpoint:

```bash
curl http://localhost:<port>
```

These commands help determine whether a problem exists at the application, container, Docker network, or EC2 network layer.

---

# 17. Network Verification Checklist

Before moving to the container deployment stage:

* [ ] EC2 has network connectivity
* [ ] EC2 public IP is reachable
* [ ] Security Group rules are configured
* [ ] Required host ports are mapped
* [ ] Frontend container is connected to the Docker network
* [ ] Backend container is connected to the Docker network
* [ ] Frontend can reach the backend
* [ ] Backend can reach MongoDB Atlas
* [ ] MongoDB Atlas allows the required connection
* [ ] Unnecessary public ports are not exposed

---

# Outcome

At the end of this stage, the network path between the different layers should be understood and verified:

```text
Internet
   ↓
AWS EC2
   ↓
Security Group
   ↓
Docker Network
   ↓
Frontend Container
   ↓
Backend Container
   ↓
MongoDB Atlas
```

The infrastructure is now ready for the actual container deployment.

---

## Next Step

Proceed to:

**[05 — Container Deployment](./05-container-deployment.md)**
