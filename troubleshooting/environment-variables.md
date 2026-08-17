# Environment Variables

Environment variables are used to provide configuration values to the application without hardcoding them into the source code or Docker image.

For this deployment, they are mainly required by the backend container.

---

## 1. Why Use Environment Variables?

Instead of putting sensitive or environment-specific values directly in the code:

```text
MongoDB connection string
JWT secret
API configuration
```

we provide them at runtime.

```text
Environment Variables
        │
        ▼
Backend Container
        │
        ▼
Node.js Application
```

**Benefits**

* Keeps secrets out of source code
* Makes configuration different between local and AWS environments
* Allows the same Docker image to be reused
* Makes deployments easier to manage

---

## 2. Backend Environment Variables

Example:

```env
PORT=5000
MONGODB_URI=<mongodb-connection-string>
JWT_SECRET=<secret>
```

The exact variables depend on the application.

---

## 3. Passing Variables to Docker

One simple approach is using an environment file:

```bash
docker run -d \
  --name backend \
  --env-file .env \
  <backend-image>
```

Docker passes these values into the container.

```text
.env
 │
 ▼
Docker
 │
 ▼
Backend Container
 │
 ▼
Node.js Application
```

---

## 4. Important Security Rules

Never commit production `.env` files to GitHub.

Add:

```text
.env
```

to `.gitignore`.

Do not expose secrets in:

* GitHub repositories
* Dockerfiles
* README files
* Screenshots
* Public logs

---

## 5. Verification

Check that the container received the required variables:

```bash
docker inspect backend
```

Avoid printing secret values unnecessarily.

Also verify the application logs to confirm that the backend successfully starts and connects to MongoDB.

---

## Key Point

Docker images should contain the application, not environment-specific secrets.

```text
Same Docker Image
       │
       ├── Local Environment
       │
       └── AWS Environment
              │
              ▼
       Different Variables
```

This makes the container image reusable across environments.
