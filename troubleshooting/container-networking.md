# Container Networking Troubleshooting

This document covers troubleshooting scenarios related to communication between the Docker containers used by the application.

The deployment consists of:

* **Frontend container** — React + Nginx
* **Backend container** — Node.js + Express
* **MongoDB Atlas** — External managed database

The primary objective is to ensure that the frontend and backend containers can communicate correctly while maintaining appropriate network isolation.

---

# 1. Expected Container Network

The expected architecture is:

```text
                    AWS EC2
                       │
                 Docker Engine
                       │
              application-network
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
                                │ Outbound
                                ▼
                         MongoDB Atlas
```

The frontend and backend should be attached to the same Docker network when they need direct container-to-container communication.

---

# 2. How Docker Networking Works

Docker provides networking functionality that allows containers to communicate with each other and with external networks.

For this deployment, a user-defined bridge network can be used:

```text
Docker Host
     │
     ▼
application-network
     │
     ├── frontend
     │
     └── backend
```

Instead of relying on hard-coded container IP addresses, containers can communicate using Docker-provided DNS names.

For example:

```text
frontend → backend:5000
```

The container name or service name can resolve to the backend container's IP address within the Docker network.

---

# 3. Why a User-Defined Network?

A dedicated Docker network provides:

* Container-to-container communication
* Automatic DNS-based service discovery
* Network isolation
* Cleaner deployment configuration
* Easier troubleshooting

Create the network:

```bash
docker network create application-network
```

Verify:

```bash
docker network ls
```

---

# 4. Verify Network Membership

Inspect the network:

```bash
docker network inspect application-network
```

The output should show the containers attached to it.

Conceptually:

```text
application-network
│
├── frontend
│   └── IP: 172.x.x.x
│
└── backend
    └── IP: 172.x.x.x
```

The exact IP addresses are dynamically assigned and should not be treated as permanent identifiers.

---

# 5. Common Problem: Containers on Different Networks

### Symptom

The frontend container cannot communicate with the backend.

### Check

Inspect the networks:

```bash
docker inspect frontend
```

```bash
docker inspect backend
```

Compare their network configuration.

If they are attached to different networks:

```text
frontend
   │
   └── network-A

backend
   │
   └── network-B
```

they cannot directly communicate through those networks.

### Resolution

Connect both containers to the same application network.

```bash
docker network connect application-network frontend
```

```bash
docker network connect application-network backend
```

Or recreate the containers using the correct network configuration.

---

# 6. Common Problem: Using Container IP Addresses

Avoid configuring the frontend with a hard-coded backend container IP.

For example, avoid:

```text
http://172.18.0.3:5000
```

Container IP addresses can change when containers are recreated.

Instead, within a Docker network, use the container/service name:

```text
http://backend:5000
```

Conceptually:

```text
Frontend
   │
   │ backend:5000
   ▼
Docker DNS
   │
   ▼
Backend Container
```

This makes the deployment more resilient to container recreation.

---

# 7. Important Browser Networking Consideration

There is an important distinction between:

```text
Container → Container
```

and:

```text
Browser → Container
```

A browser running on the user's computer **does not participate in the Docker network**.

For example:

```text
Browser
   │
   │ Cannot resolve Docker hostname
   ▼
backend:5000
```

The hostname `backend` is normally meaningful inside the Docker network, not to an external user's browser.

Therefore, if React executes API requests in the browser, the browser must use an endpoint that is reachable from the user's machine.

For example:

```text
Browser
   │
   ▼
Public API Endpoint
   │
   ▼
EC2
   │
   ▼
Backend Container
```

This distinction is critical when designing a Dockerized frontend/backend deployment.

---

# 8. Common Problem: `localhost`

A common mistake is using:

```text
http://localhost:5000
```

for the frontend's production API URL.

There are multiple meanings of `localhost` depending on where the request originates.

### From the EC2 host

```text
localhost
    ↓
EC2 instance
```

### From the backend container

```text
localhost
    ↓
Backend container itself
```

### From the user's browser

```text
localhost
    ↓
User's own computer
```

Therefore, `localhost` should not be assumed to mean the EC2 instance or backend container.

---

# 9. Common Problem: Backend Listening on `127.0.0.1`

A backend application may be configured to listen on:

```text
127.0.0.1
```

Inside a container, this can prevent external interfaces from reaching the application.

The server should generally listen on:

```text
0.0.0.0
```

Conceptually:

```text
Backend Container
       │
       ├── 127.0.0.1
       │       └── Container-local
       │
       └── 0.0.0.0
               └── Container network accessible
```

This is one of the first things to check when:

> The application is running inside the container, but the mapped port cannot reach it.

---

# 10. Common Problem: Incorrect Port

Suppose the backend listens internally on port `5000`.

The Docker configuration must expose the appropriate port.

Conceptually:

```text
EC2 Host
Port 5000
    │
    ▼
Docker
    │
    ▼
Backend Container
Port 5000
```

Verify:

```bash
docker port backend
```

Also check:

```bash
docker ps
```

Look for a mapping similar to:

```text
0.0.0.0:5000->5000/tcp
```

---

# 11. Container Port vs Host Port

These are different concepts.

Example:

```text
5000:5000
```

means:

```text
HOST PORT : CONTAINER PORT
```

Therefore:

```text
EC2 :5000
     │
     ▼
Container :5000
```

Another possible configuration could be:

```text
8080:5000
```

which means:

```text
EC2 :8080
     │
     ▼
Container :5000
```

The application itself continues listening on `5000` inside the container.

---

# 12. Testing Container Connectivity

If the frontend container has a shell and networking utilities available, enter the container:

```bash
docker exec -it frontend sh
```

Then test the backend:

```bash
curl http://backend:5000/<api-endpoint>
```

If this succeeds:

```text
Frontend Container
       │
       ▼
Docker DNS
       │
       ▼
Backend Container
```

is functioning.

If it fails, investigate the Docker network and backend service.

> Minimal production images may not contain tools such as `curl`, `ping`, or `nslookup`. In that case, use a temporary diagnostic container attached to the same Docker network.

---

# 13. Verify Docker DNS

Docker's user-defined networks provide internal DNS resolution.

From a container attached to the application network:

```text
backend
```

should resolve to the backend container.

The conceptual flow is:

```text
backend:5000
     │
     ▼
Docker DNS
     │
     ▼
Backend Container IP
     │
     ▼
Port 5000
```

This is preferable to hard-coding container IP addresses.

---

# 14. Common Problem: Backend Container Not Running

If the frontend cannot connect to the backend, verify:

```bash
docker ps
```

Then:

```bash
docker ps -a
```

If the backend has stopped:

```bash
docker logs backend
```

Typical causes include:

* Application startup error
* Missing environment variable
* Database connection failure
* Port conflict
* Dependency problem
* Runtime exception

The network may be functioning correctly while the backend application itself is unhealthy.

---

# 15. Common Problem: Network Exists but Container Is Not Attached

A Docker network can exist while the required container is not connected to it.

Check:

```bash
docker network inspect application-network
```

If the backend is missing, connect it:

```bash
docker network connect application-network backend
```

Then verify again:

```bash
docker network inspect application-network
```

---

# 16. Common Problem: Network Recreation

Container IP addresses can change when containers are recreated.

For example:

```text
Old Backend Container
172.18.0.3
```

After recreation:

```text
New Backend Container
172.18.0.5
```

This is why applications should use stable service/container names rather than container IP addresses.

Preferred:

```text
backend:5000
```

Avoid:

```text
172.18.0.3:5000
```

---

# 17. Frontend → Backend Network Models

There are two different scenarios to understand.

### Model A — Browser Directly Calls Backend

```text
Browser
   │
   │ HTTP
   ▼
EC2 Public IP
   │
   ▼
Backend Container
```

In this model, the backend needs a reachable host endpoint and appropriate network/security configuration.

### Model B — Reverse Proxy

A more production-oriented pattern is:

```text
Browser
   │
   ▼
Nginx / Reverse Proxy
   │
   ├── Static Frontend
   │
   └── /api → Backend Container
```

Then:

```text
Browser
   │
   ▼
Nginx
   │
   └── /api
        │
        ▼
     Backend
```

This can avoid exposing the backend port directly to the internet.

---

# 18. Backend → MongoDB Atlas

MongoDB Atlas is outside the Docker network.

The backend container communicates externally:

```text
Backend Container
       │
       ▼
Docker Host
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

Therefore, MongoDB Atlas does not need to be added to the Docker network.

The backend needs:

* Valid MongoDB connection string
* Valid database credentials
* EC2 outbound connectivity
* Correct Atlas Network Access configuration

---

# 19. Troubleshooting Sequence

When container communication fails, use this sequence:

```text
1. Is the backend container running?
             ↓
2. Is the backend application healthy?
             ↓
3. Is the backend listening on the expected port?
             ↓
4. Are both containers on the same network?
             ↓
5. Can Docker DNS resolve the backend?
             ↓
6. Can the frontend container reach backend:port?
             ↓
7. Is the frontend using the correct API endpoint?
             ↓
8. Is EC2/security configuration blocking external traffic?
```

This approach isolates the failure instead of changing several configurations simultaneously.

---

# 20. Useful Diagnostic Commands

### List containers

```bash
docker ps
```

### List all containers

```bash
docker ps -a
```

### List networks

```bash
docker network ls
```

### Inspect network

```bash
docker network inspect application-network
```

### Inspect container networking

```bash
docker inspect frontend
```

```bash
docker inspect backend
```

### Check port mapping

```bash
docker port backend
```

### Check logs

```bash
docker logs backend
```

### Execute a shell inside a container

```bash
docker exec -it frontend sh
```

### Check container resource usage

```bash
docker stats
```

---

# 21. Troubleshooting Decision Tree

```text
Frontend cannot reach Backend
             │
             ▼
      Backend running?
        /          \
      NO            YES
      │              │
      ▼              ▼
 Check logs      Same Docker network?
                     /        \
                   NO          YES
                   │             │
                   ▼             ▼
             Attach network   Backend port?
                                /      \
                              NO        YES
                              │          │
                              ▼          ▼
                         Fix port    Test DNS
                                       │
                                       ▼
                                Test backend:5000
                                       │
                                  ┌────┴────┐
                                FAIL      SUCCESS
                                  │          │
                                  ▼          ▼
                           Check backend   Check frontend
                           configuration   API endpoint
```

---

# 22. Security Considerations

Container networking should follow the principle of least exposure.

Avoid unnecessarily exposing backend ports:

```text
Internet
   │
   ├── :80 → Frontend
   │
   └── :5000 → Backend  ← Avoid when unnecessary
```

Prefer:

```text
Internet
   │
   ▼
Frontend / Reverse Proxy
   │
   ▼
Backend
   │
   ▼
MongoDB Atlas
```

The backend should only be publicly accessible when there is a specific architectural requirement.

---

# 23. Key Lessons

### 1. Container networking is separate from AWS networking

AWS controls traffic around the EC2 instance.

Docker controls communication between containers.

```text
AWS Networking
      ↓
EC2
      ↓
Docker Networking
      ↓
Containers
```

### 2. `localhost` is context-dependent

`localhost` inside a container refers to that container, not another container and not the EC2 host.

### 3. Use service/container names instead of IP addresses

Container IP addresses are dynamic.

### 4. A running container does not guarantee connectivity

The application must also:

* Be healthy
* Listen on the correct interface
* Listen on the correct port
* Be attached to the expected network

### 5. Browser traffic is different from container traffic

A user's browser does not automatically have access to Docker's internal network or DNS.

---

# 24. Final Verification

A correctly configured container network should resemble:

```text
                    AWS EC2
                       │
                 Docker Engine
                       │
             application-network
                       │
              ┌────────┴────────┐
              │                 │
              ▼                 ▼
         Frontend            Backend
         Container           Container
              │                 │
              │    HTTP/API     │
              └─────────────────┘
                                │
                                │ MongoDB Connection
                                ▼
                         MongoDB Atlas
```

The key communication paths are:

```text
Frontend ↔ Backend
Backend  → MongoDB Atlas
```

while the public internet should have access only to the components that are intentionally exposed.

---

## Troubleshooting Summary

When container networking fails:

1. Verify that the containers are running.
2. Verify that both containers are attached to the expected Docker network.
3. Verify the backend listening address.
4. Verify container and host ports.
5. Test backend connectivity independently.
6. Test Docker DNS/service-name resolution.
7. Test frontend-to-backend communication from the appropriate network context.
8. Check the frontend API endpoint.
9. Check EC2 Security Group rules for externally exposed services.
10. Check MongoDB Atlas connectivity separately from Docker networking.

This layered approach makes it possible to identify whether the failure originates from **Docker networking, application configuration, AWS networking, or the external database connection**.
