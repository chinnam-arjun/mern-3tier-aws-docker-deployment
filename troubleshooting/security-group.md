# Security Group Configuration

AWS Security Groups act as a virtual firewall for the EC2 instance. They control which inbound and outbound traffic is allowed.

For this Dockerized 3-tier deployment, only the required ports should be exposed.

---

## 1. Required Inbound Rules

Recommended configuration:

| Port | Protocol | Source      | Purpose               |
| ---: | -------- | ----------- | ---------------------- |
|   22 | TCP      | Your IP     | SSH access              |
|   80 | TCP      | `0.0.0.0/0` | Frontend HTTP           |
|  443 | TCP      | `0.0.0.0/0` | HTTPS, if configured    |

### Backend Port

If the backend is exposed directly through EC2, port `5000` may be required:

| Port | Protocol | Source                | Purpose      |
| ---: | -------- | ---------------------- | ------------ |
| 5000 | TCP      | Required source only   | Backend API  |

Avoid exposing port `5000` publicly unless the architecture requires it.

---

## 2. Recommended Architecture

Prefer exposing only the frontend:

```text
Internet
   │
   ▼
Port 80 / 443
   │
   ▼
Frontend Container
   │
   ▼
Backend Container
   │
   ▼
MongoDB Atlas
```

Instead of:

```text
Internet
   ├── Port 80  → Frontend
   └── Port 5000 → Backend
```

This reduces the public attack surface.

---

## 3. SSH Security

Avoid:

```text
Port 22
Source: 0.0.0.0/0
```

Prefer:

```text
Port 22
Source: <Your-Public-IP>/32
```

This allows SSH access only from your machine.

---

## 4. Outbound Rules

For this deployment, outbound internet access is required for:

* MongoDB Atlas connectivity
* Package/image downloads
* External APIs, if used
* Docker image pulls

The default outbound rule allowing required internet traffic is generally sufficient for this project.

---

## 5. Verification

Check the Security Group associated with the EC2 instance and verify:

```text
✓ SSH restricted
✓ HTTP allowed
✓ HTTPS allowed if configured
✓ Backend port not unnecessarily public
✓ Required outbound traffic allowed
```

---

## Key Principle

**Expose only what users need to access.**

For this deployment:

```text
Public:
  80 / 443
Restricted:
  22
Internal:
  Backend
```

This follows the **principle of least privilege** and keeps the EC2 attack surface smaller.
