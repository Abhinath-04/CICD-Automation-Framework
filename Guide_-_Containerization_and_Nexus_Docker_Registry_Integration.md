# Guide: Containerization and Nexus Docker Registry Integration

## 1. Configuring Nexus as a Docker Registry

By default, Nexus does not listen for Docker traffic. You must create a specific repository and assign it a dedicated port.

### 1.1. Enable Docker Bearer Token Realm

Before Docker can authenticate, you must enable the security realm in the Nexus UI:

1. Navigate to **Administration (cog icon) > Security > Realms**.
2. Move **Docker Bearer Token Realm** to the **Active** column.
3. Click **Save**.

### 1.2. Create a Docker (Hosted) Repository

1. Navigate to **Repository > Repositories > Create repository**.
2. Select **docker (hosted)**.
3. **Name:** `docker-private`
4. **HTTP Connector:** Check the box and enter a port (e.g., 8082).
5. **Enable Docker V1 API:** Leave unchecked (use V2).
6. **Storage:** Select your default Blob Store.
7. Click **Create repository**.

### 1.3. Open Firewall for Docker Port

Ensure the port defined in the connector (8082) is open on your Linux server:

```bash
sudo ufw allow 8082/tcp  # Ubuntu

# OR
sudo firewall-cmd --permanent --add-port=8082/tcp && sudo firewall-cmd --reload  # RHEL
```

## 2. Docker Creation Methods

To ensure professional, production-ready images, follow these two standard patterns.

### 2.1. Method A: Standard Dockerfile (Simple)

Best for simple scripts or basic services.

```dockerfile
FROM alpine:latest
RUN apk add --no-cache python3 py3-pip
WORKDIR /app
COPY . /app
CMD ["python3", "app.py"]
```

### 2.2. Method B: Multi-Stage Build (Recommended)

Best for compiled languages (Java, Go, Node). It minimizes the final image size by excluding build tools.

```dockerfile
# Stage 1: Build
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /build
COPY . .
RUN mvn clean package -DskipTests

# Stage 2: Runtime
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=build /build/target/app-1.0.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

## 3. Authenticating and Pushing to Nexus

### 3.1. Docker Daemon Configuration (Insecure Registry)

If you are not using HTTPS (SSL), you must tell Docker to trust your Nexus server.

1. Edit `/etc/docker/daemon.json`:
   ```json
   {
     "insecure-registries" : ["your-server-ip:8082"]
   }
   ```
2. Restart Docker:
   ```bash
   sudo systemctl restart docker
   ```

### 3.2. Login to Nexus Registry

```bash
docker login your-server-ip:8082
# Enter Nexus admin credentials when prompted
```

### 3.3. Tag and Push Workflow

To push an image, it must be tagged with the registry's address.

```bash
# 1. Build the local image
docker build -t my-application:v1 .

# 2. Tag for Nexus (the tag must match the registry URL and port)
docker tag my-application:v1 your-server-ip:8082/my-application:v1

# 3. Push to Nexus
docker push your-server-ip:8082/my-application:v1
```

## 4. Automating with GitLab CI/CD

To automate this within your `.gitlab-ci.yml` (using a GitLab Runner), use the following job template:

```yaml
variables:
  NEXUS_REGISTRY: "your-server-ip:8082"
  IMAGE_NAME: "my-application"

stages:
  - build-and-push

publish_image:
  stage: build-and-push
  script:
    - echo "$NEXUS_PASSWORD" | docker login $NEXUS_REGISTRY -u "$NEXUS_USER" --password-stdin
    - docker build -t $NEXUS_REGISTRY/$IMAGE_NAME:$CI_COMMIT_SHORT_SHA .
    - docker push $NEXUS_REGISTRY/$IMAGE_NAME:$CI_COMMIT_SHORT_SHA
```

> **Note:** Store `NEXUS_USER` and `NEXUS_PASSWORD` in GitLab **Settings > CI/CD > Variables**.
