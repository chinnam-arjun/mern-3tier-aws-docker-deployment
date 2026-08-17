# Backend Connectivity Troubleshooting

This document covers troubleshooting scenarios where the React frontend cannot communicate with the Node.js/Express backend after the application is deployed on AWS EC2 using Docker.

The goal is to identify whether the problem exists at the **frontend, Docker, EC2, security group, or backend application layer**.

---

## 1. Expected Request Flow

The expected communication path is:

```text
User Browser
     │
     ▼
Frontend Container
     │
     │ HTTP API Request
     ▼
Backend Container
     │
     ▼
Express API
     │
     ▼
MongoDB Atlas
```

If the frontend cannot retrieve data from the backend, each layer should be checked independently.

---

# 2. Problem Scenario

### Symptom

The frontend application loads successfully, but operations that require the backend API fail.

Examples:

* Login does not work
* Signup does not work
* Posts do not load
* API requests remain pending
* Browser displays network errors
* Frontend reports `Failed to fetch`
* API returns `502`, `404`, or connection errors

This indicates that the frontend itself may be working while the communication path to the backend is failing.

---

# 3. Troubleshooting Strategy

Instead of changing multiple configurations at once, isolate the problem layer by layer.

```text
Frontend
   │
   ▼
API URL
   │
   ▼
EC2 Network
   │
   ▼
Docker Port Mapping
   │
   ▼
Backend Container
   │
   ▼
Express Server
   │
   ▼
MongoDB Atlas
```

The investigation should follow this order.

---

# 4. Step 1 — Check Backend Container

First verify whether the backend container is actually running.

```bash
docker ps
```

Expected:

```text
CONTAINER
│
└── backend
       STATUS: Up
```

If the backend is not listed, check all containers:

```bash
docker ps -a
```

If the container exited, inspect the logs:

```bash
docker logs backend
```

---

# 5. Step 2 — Check Backend Logs

Backend logs are usually the fastest way to identify application startup problems.

```bash
docker logs backend
```

Look for errors related to:

* Missing environment variables
* MongoDB connection failure
* Port conflicts
* Node.js runtime errors
* Missing dependencies
* Application exceptions

For continuous monitoring:

```bash
docker logs -f backend
```

If the application starts successfully, the logs should indicate that the Express server is listening on its configured port.

---

# 6. Step 3 — Verify Backend Port

Check the Docker port mapping:

```bash
docker port backend
```

For example:

```text
5000/tcp -> 0.0.0.0:5000
```

This represents:

```text
EC2 Host
Port 5000
    │
    ▼
Backend Container
Port 5000
```

If the expected port mapping is missing or incorrect, requests will not reach the backend.

---

# 7. Step 4 — Test Backend Directly

Before testing the frontend, test the backend independently from the EC2 host.

```bash
curl http://localhost:5000
```

Or test an appropriate API endpoint:

```bash
curl http://localhost:5000/<api-endpoint>
```

### If this succeeds

The backend application and Docker port mapping are likely functioning.

Continue investigating:

```text
Frontend
   ↓
API URL
   ↓
EC2 Security Group
```

### If this fails

The problem is likely inside:

* Backend container
* Express application
* Port configuration
* Environment variables
* Docker networking

---

# 8. Step 5 — Verify the Backend Listening Address

A frequent container networking issue occurs when the backend server listens only on the loopback interface.

For example:

```text
127.0.0.1
```

Inside a container, this can prevent connections arriving through the container's network interface.

The backend should generally listen on:

```text
0.0.0.0
```

Conceptually:

```text
Incorrect
Backend
   │
   └── 127.0.0.1
          │
          └── Container-local access


Correct
Backend
   │
   └── 0.0.0.0
          │
          └── Container network access
```

This is an important check when:

> "The backend works inside the container but cannot be reached through the mapped port."

---

# 9. Step 6 — Check Docker Network

List Docker networks:

```bash
docker network ls
```

Inspect the application network:

```bash
docker network inspect <network-name>
```

Verify that both containers are attached:

```text
Application Network
│
├── frontend
└── backend
```

If the frontend and backend are intended to communicate through a Docker network, both containers must participate in the same network.

---

# 10. Step 7 — Verify Container-to-Container Connectivity

If the backend is reachable locally but the frontend still cannot communicate with it, test communication from the frontend container.

First access the frontend container:

```bash
docker exec -it frontend sh
```

Then test the backend endpoint from inside the container.

For example:

```bash
curl http://backend:5000/<api-endpoint>
```

The exact hostname depends on the Docker network configuration.

Docker's internal DNS can resolve a container by its container/service name when both containers are attached to the same user-defined network.

Expected flow:

```text
Frontend Container
       │
       │ HTTP
       ▼
backend:5000
       │
       ▼
Backend Container
```

---

# 11. Step 8 — Check the Frontend API URL

Another common cause is an incorrect API URL configured in the frontend.

For example, the frontend may still be configured with:

```text
http://localhost:5000
```

This is problematic when the request originates from the user's browser.

### Why?

`localhost` refers to the **machine running the browser**, not the EC2 server.

Conceptually:

```text
Browser
   │
   └── localhost:5000
           │
           └── User's own machine
```

It does not automatically refer to:

```text
EC2:5000
```

The frontend must use an API endpoint that is reachable from the browser and matches the deployment architecture.

---

# 12. Step 9 — Check Browser Developer Tools

Open the browser's Developer Tools and inspect:

```text
Network → API Request
```

Check:

* Request URL
* HTTP method
* Status code
* Response
* Request timing
* CORS errors
* Connection errors

Example:

```text
Request URL:
http://<EC2-PUBLIC-IP>:5000/api/...
```

If the request is pointing to the wrong host or port, the frontend configuration needs to be corrected.

---

# 13. Step 10 — Check EC2 Security Group

If the backend is intentionally exposed through a public port, verify that the EC2 Security Group allows the required inbound traffic.

Example:

```text
Type: Custom TCP
Port: 5000
Source: <required source>
```

However, avoid exposing the backend publicly unless the architecture requires it.

A better production architecture is often:

```text
Internet
   │
   ▼
Frontend
   │
   ▼
Private Backend
```

rather than:

```text
Internet
   │
   ├── Frontend :80
   │
   └── Backend :5000
```

---

# 14. Step 11 — Check EC2 Listening Ports

Check which ports are listening on the EC2 instance:

```bash
ss -tulpn
```

You can also inspect Docker's port mappings:

```bash
docker ps
```

For example:

```text
0.0.0.0:5000->5000/tcp
```

This confirms that Docker has published the backend container port to the host.

---

# 15. Step 12 — Check MongoDB Separately

If the backend receives the request but returns an error, the problem may not be network connectivity between frontend and backend.

The backend may be failing while processing the request because MongoDB Atlas is unreachable.

Check:

```bash
docker logs backend
```

Look for:

```text
MongoServerSelectionError
Connection timeout
Authentication failed
DNS resolution failure
```

Then verify:

```text
Backend Container
       │
       ▼
Internet Connectivity
       │
       ▼
MongoDB Atlas Network Access
       │
       ▼
Database Authentication
```

---

# 16. Common Failure Patterns

## Scenario A — Frontend Loads, API Completely Fails

Possible causes:

```text
Incorrect API URL
Security Group
Backend not running
Wrong port mapping
```

Start with:

```bash
docker ps
docker logs backend
docker port backend
```

Then inspect the browser's Network tab.

---

## Scenario B — Backend Works With `curl`, Frontend Cannot Reach It

If:

```bash
curl http://localhost:5000/<endpoint>
```

works on EC2 but the browser cannot reach the API, investigate:

```text
Frontend API URL
       ↓
Public/Private accessibility
       ↓
Security Group
       ↓
CORS
```

The backend itself is likely functioning.

---

## Scenario C — Backend Container Is Running but API Fails

Check:

```bash
docker logs backend
```

Then verify:

```text
Application Port
      ↓
Listening Address
      ↓
Docker Port Mapping
```

A running container does not necessarily mean the application inside it is healthy.

---

## Scenario D — Frontend → Backend Works, Backend → Database Fails

The application reaches the backend successfully, but database operations fail.

Investigate:

```text
MongoDB URI
MongoDB Credentials
Atlas Network Access
EC2 Outbound Connectivity
DNS
```

---

# 17. Diagnostic Command Sequence

A useful troubleshooting sequence is:

```bash
docker ps
```

```bash
docker ps -a
```

```bash
docker logs backend
```

```bash
docker port backend
```

```bash
docker network ls
```

```bash
docker network inspect <network-name>
```

```bash
curl http://localhost:<backend-port>
```

```bash
ss -tulpn
```

Then inspect the browser:

```text
Developer Tools
      ↓
Network
      ↓
Failed API Request
      ↓
Request URL
      ↓
Status / Error
```

This provides a systematic debugging path instead of guessing.

---

# 18. Troubleshooting Decision Tree

```text
Frontend cannot reach Backend
             │
             ▼
     Is backend running?
        /          \
      NO            YES
      │              │
      ▼              ▼
docker logs      Test localhost
backend          API endpoint
                     │
                ┌────┴────┐
              FAIL       SUCCESS
                │           │
                ▼           ▼
          Check backend   Check frontend
          configuration   API URL
                              │
                              ▼
                       Check Security Group
                              │
                              ▼
                        Check CORS
                              │
                              ▼
                    Check container network
```

---

# 19. Root Cause Classification

When troubleshooting, classify the failure before applying a fix.

| Layer          | Example Failure              |
| -------------- | ---------------------------- |
| Frontend       | Incorrect API URL            |
| Browser        | CORS / network error         |
| Docker         | Wrong port mapping           |
| Docker Network | Containers not connected     |
| Backend        | Express server crashed       |
| EC2            | Port inaccessible            |
| Security Group | Inbound traffic blocked      |
| MongoDB Atlas  | Network access denied        |
| Configuration  | Missing environment variable |

This approach makes troubleshooting faster and easier to explain during technical interviews.

---

# 20. Lessons From the Scenario

The main lesson is that **"container is running" does not mean "application is reachable."**

A successful deployment requires multiple layers to work together:

```text
Application
     ↓
Container
     ↓
Docker Network
     ↓
EC2 Port
     ↓
Security Group
     ↓
Internet
```

A failure at any layer can prevent the frontend from reaching the backend.

The most effective approach is therefore to test each layer independently and move outward only after the previous layer has been verified.

---

## Troubleshooting Summary

When the frontend cannot communicate with the backend:

1. Check whether the backend container is running.
2. Inspect backend logs.
3. Verify the backend listening port.
4. Verify Docker port mapping.
5. Test the backend directly from EC2.
6. Verify the application is listening on the correct interface.
7. Inspect Docker network membership.
8. Test container-to-container connectivity.
9. Verify the frontend API URL.
10. Check browser Network and Console errors.
11. Check EC2 Security Group rules.
12. Check MongoDB Atlas connectivity if the backend receives requests but database operations fail.

This systematic process isolates the failing layer and prevents unnecessary configuration changes.
