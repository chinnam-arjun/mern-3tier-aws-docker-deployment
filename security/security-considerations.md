# Security Considerations

This document summarizes the main security considerations for the containerized 3-tier application deployed on AWS EC2.

Network exposure and SSH rules are covered in detail in `security-group-configuration.md`, and secret handling is covered in detail in `environment-variables.md`. This document focuses on the remaining considerations: database access, transport security, image hygiene, container privileges, patching, and information exposure.

---

## 1. Protect MongoDB Atlas

MongoDB Atlas should only allow connections from trusted sources.

Configure Network Access appropriately and use strong database credentials.

Avoid unnecessarily allowing:

```text
0.0.0.0/0
```

for database access.

---

## 2. Use HTTPS

For a production deployment, use HTTPS instead of sending application traffic over plain HTTP.

```text
Browser
   │
   │ HTTPS
   ▼
EC2 / Reverse Proxy
   │
   ▼
Application
```

TLS protects data while it is travelling between the client and server.

---

## 3. Keep Docker Images Minimal

Use production images containing only the dependencies required to run the application.

**Benefits**

* Smaller attack surface
* Smaller image size
* Faster deployment
* Fewer unnecessary packages

Avoid running unnecessary services inside containers.

---

## 4. Avoid Running Containers as Root

Where possible, configure application containers to run as a non-root user.

```text
Root Container
     ↓
Higher Privileges

Non-root Container
     ↓
Reduced Privileges
```

This limits the impact of a potential application compromise.

---

## 5. Keep Software Updated

Regularly update:

* EC2 operating system
* Docker Engine
* Base Docker images
* Application dependencies
* Node.js runtime

Outdated components may contain known vulnerabilities.

---

## 6. Do Not Expose Internal Details

Avoid exposing:

* Database credentials
* JWT secrets
* Internal IP addresses
* Stack traces
* Environment variables
* Docker configuration details

especially in API responses and public logs.

---

## Security Checklist

```text
[ ] SSH restricted to trusted IP
[ ] Only required ports exposed
[ ] Backend not unnecessarily public
[ ] Secrets stored outside source code
[ ] .env excluded from Git
[ ] MongoDB Atlas access restricted
[ ] HTTPS configured for production
[ ] Docker images kept minimal
[ ] Containers run with least privilege
[ ] OS and dependencies regularly updated
```

---

## Key Principle

The main security approach for this deployment is:

```text
Least Privilege
      +
Minimal Exposure
      +
Protected Secrets
      +
Network Isolation
      +
Regular Updates
```

Security should be considered at every layer rather than treated as a separate step after deployment.
