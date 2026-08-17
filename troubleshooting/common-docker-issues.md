# Common Docker Issues

This document covers common Docker problems encountered while deploying the application on AWS EC2.

---

## 1. Container Exits Immediately

**Check**

```bash
docker ps -a
```

Then check the logs:

```bash
docker logs <container-name>
```

**Common Causes**

* Application startup error
* Missing environment variables
* Incorrect configuration
* Application crash

**Solution:** Check the logs first instead of restarting the container repeatedly.

---

## 2. Container Is Running but Application Is Not Accessible

Check the port mapping:

```bash
docker ps
```

Example:

```text
0.0.0.0:5000->5000/tcp
```

Also verify that the application listens on:

```text
0.0.0.0
```

rather than only:

```text
127.0.0.1
```

---

## 3. Port Already in Use

**Error**

```text
Bind for 0.0.0.0:5000 failed: port is already allocated
```

**Check**

```bash
sudo ss -tulpn | grep 5000
```

Also check existing containers:

```bash
docker ps
```

**Solution:** Stop the process/container using the port or use another host port.

---

## 4. Image Not Found

**Check**

```bash
docker images
```

If using a registry, make sure the image was pulled successfully:

```bash
docker pull <image>
```

Verify again:

```bash
docker images
```

---

## 5. Environment Variables Missing

If the application fails because configuration values are missing, verify the environment file:

```bash
cat .env
```

Then recreate the container with:

```bash
docker run --env-file .env <image>
```

Never commit production secrets to GitHub.

---

## 6. Frontend Cannot Reach Backend

**Check:**

```bash
docker ps
docker network ls
docker network inspect <network-name>
```

**Verify:**

* Backend container is running
* Correct backend port is used
* Containers are on the expected network
* Frontend API URL is correct

Remember that `localhost` in the browser refers to the user's machine, not the EC2 instance.

---

## 7. Container Cannot Connect to MongoDB Atlas

Check backend logs:

```bash
docker logs backend
```

**Verify:**

* `MONGODB_URI` is correct
* MongoDB credentials are valid
* MongoDB Atlas Network Access allows the connection
* EC2 has outbound internet connectivity

---

## 8. Changes Are Not Appearing

Docker may still be using an older image.

Check images:

```bash
docker images
```

Rebuild when necessary:

```bash
docker build -t <image-name> .
```

Then recreate the container using the new image.

---

## 9. Container Has No Internet Access

Test from the container if diagnostic tools are available:

```bash
docker exec -it <container-name> sh
```

Then test connectivity:

```bash
curl https://example.com
```

If this fails, investigate:

* EC2 outbound connectivity
* Docker networking
* DNS configuration
* AWS Security Group rules

---

## 10. Useful First Commands

When something goes wrong, start with these:

```bash
docker ps
```

```bash
docker ps -a
```

```bash
docker logs <container-name>
```

```bash
docker images
```

```bash
docker network ls
```

```bash
docker stats
```

These commands quickly show whether the problem is related to the container, image, application, network, or resources.

---

## Troubleshooting Principle

Do not randomly change multiple Docker settings.

Follow the layers:

```text
Image
  ↓
Container
  ↓
Application
  ↓
Port
  ↓
Docker Network
  ↓
EC2 Network
  ↓
External Service
```

Identify the failing layer first, then fix that specific problem.
