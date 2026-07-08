# Installation Guide: Dependency-Track with Trivy Integration (Self-Hosted, Docker)

## 1. System Requirements

| Component | Minimum | Recommended |
|---|---|---|
| CPU | 2 vCPUs | 4 vCPUs |
| RAM | 4 GB | 8 GB+ (Dependency-Track API server is JVM-based) |
| Storage | 20 GB+ | 50 GB+ (grows with number of projects/BOMs retained) |
| Docker | 24.x+ | Latest stable |
| Docker Compose | v2 (plugin) | Latest stable |
| OS | Ubuntu 22.04 LTS / RHEL 8/9 | Same |

Dependency-Track consists of two containers:
- **`dtrack-apiserver`** — backend API, vulnerability database, analysis engine
- **`dtrack-frontend`** — web UI (talks to the API server)

## 2. Install Docker and Docker Compose

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install -y docker.io docker-compose-plugin
sudo systemctl enable --now docker

# RHEL/AlmaLinux/Rocky
sudo dnf install -y docker docker-compose-plugin
sudo systemctl enable --now docker
```

Add your user to the `docker` group to avoid needing `sudo` for every command:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

## 3. Deploy Dependency-Track

### 3.1. Create Working Directory and Volumes

```bash
mkdir -p /opt/dependency-track
cd /opt/dependency-track
```

### 3.2. Create `docker-compose.yml`

```yaml
version: '3.8'

services:
  dtrack-apiserver:
    image: dependencytrack/apiserver:latest
    container_name: dtrack-apiserver
    restart: unless-stopped
    environment:
      - ALPINE_DATABASE_MODE=external
      - ALPINE_DATABASE_URL=jdbc:postgresql://dtrack-postgres:5432/dtrack
      - ALPINE_DATABASE_DRIVER=org.postgresql.Driver
      - ALPINE_DATABASE_USERNAME=dtrack
      - ALPINE_DATABASE_PASSWORD=YourSecurePassword
    deploy:
      resources:
        limits:
          memory: 4g
    ports:
      - "8081:8080"
    volumes:
      - dtrack-data:/data
    depends_on:
      - dtrack-postgres

  dtrack-frontend:
    image: dependencytrack/frontend:latest
    container_name: dtrack-frontend
    restart: unless-stopped
    environment:
      - API_BASE_URL=http://your-server-ip:8081
    ports:
      - "8082:8080"
    depends_on:
      - dtrack-apiserver

  dtrack-postgres:
    image: postgres:15
    container_name: dtrack-postgres
    restart: unless-stopped
    environment:
      - POSTGRES_DB=dtrack
      - POSTGRES_USER=dtrack
      - POSTGRES_PASSWORD=YourSecurePassword
    volumes:
      - dtrack-postgres-data:/var/lib/postgresql/data

volumes:
  dtrack-data:
  dtrack-postgres-data:
```

> Adjust the `memory` limit and `-Xmx` (via `ALPINE_DATABASE_*` / JVM env vars) based on the number of projects tracked. Large monorepos or many concurrent BOM uploads benefit from 4 GB+.

### 3.3. Start the Stack

```bash
docker compose up -d
docker compose ps
```

Wait 2–3 minutes on first boot — the API server initializes its internal vulnerability database (NVD mirror) on startup.

### 3.4. Open Firewall Ports

```bash
# Ubuntu
sudo ufw allow 8081/tcp   # API server
sudo ufw allow 8082/tcp   # Frontend UI

# RHEL
sudo firewall-cmd --permanent --add-port=8081/tcp
sudo firewall-cmd --permanent --add-port=8082/tcp
sudo firewall-cmd --reload
```

### 3.5. Initial Login

1. Visit `http://your-server-ip:8082`.
2. Default credentials: `admin` / `admin`.
3. You will be forced to change the password on first login.

## 4. Generate an API Key

Dependency-Track uses API keys (teams) for programmatic BOM upload from CI/CD.

1. Log in to the UI.
2. Navigate to **Administration > Access Management > Teams**.
3. Select (or create) a team, e.g. `ci-cd-automation`.
4. Under **Permissions**, grant at minimum:
   - `BOM_UPLOAD`
   - `VIEW_PORTFOLIO`
   - `VULNERABILITY_ANALYSIS`
5. Under **API Keys**, click **+** to generate a key. Copy it — this is your `DTRACK_API_KEY`.

Store this key as a masked GitLab CI/CD variable, not committed to any repository.

## 5. Create a Project in Dependency-Track

Projects can be created manually via the UI (**Projects > Create Project**) or auto-created on first BOM upload (recommended for CI/CD — Dependency-Track creates the project by name/version if it doesn't exist, given the API key has `PROJECT_CREATION_UPLOAD` permission).

## 6. Install Trivy

```bash
# Ubuntu/Debian
sudo apt install -y wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo gpg --dearmor -o /usr/share/keyrings/trivy.gpg
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt update
sudo apt install -y trivy

# RHEL/AlmaLinux/Rocky
cat <<EOF | sudo tee /etc/yum.repos.d/trivy.repo
[trivy]
name=Trivy repository
baseurl=https://aquasecurity.github.io/trivy-repo/rpm/releases/\$releasever/\$basearch/
gpgcheck=1
enabled=1
gpgkey=https://aquasecurity.github.io/trivy-repo/rpm/public.key
EOF
sudo dnf install -y trivy
```

Verify:
```bash
trivy --version
```

## 7. Integration Pattern: Trivy → Dependency-Track

There are two complementary ways to feed Trivy data into Dependency-Track:

**Pattern A — Trivy generates a CycloneDX SBOM with embedded vulnerability data, uploaded directly to Dependency-Track.**
This is the simplest path and is what the `tsp-web` / `policy-web-ui-Container` projects use.

**Pattern B — Trivy scans separately (SARIF/table output) for pipeline gating, while a distinct CycloneDX SBOM (from `cdxgen`/Maven plugin) is uploaded to Dependency-Track for continuous SCA.**
Use this if you want Trivy's container-layer findings to fail the pipeline independently of Dependency-Track's portfolio-level tracking.

### 7.1. Local Test (Pattern A)

```bash
# Scan a built image and output CycloneDX SBOM with vulnerabilities
trivy image --format cyclonedx --output result.cdx.json your-server-ip:8082/my-application:v1

# Upload to Dependency-Track
BOM_BASE64=$(base64 -w0 result.cdx.json)
curl -X PUT "http://your-server-ip:8081/api/v1/bom" \
  -H "X-Api-Key: $DTRACK_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"projectName\": \"my-application\",
    \"projectVersion\": \"v1\",
    \"autoCreate\": true,
    \"bom\": \"$BOM_BASE64\"
  }"
```

## 8. GitLab CI/CD Integration

Add a scan stage that runs Trivy against the built container image and uploads the resulting SBOM to Dependency-Track. This slots into the existing pipeline between `docker-publish` and `deploy`, consistent with the `scan` stage already used for `trivy-container-scan`.

```yaml
variables:
  DTRACK_URL: "http://your-server-ip:8081"
  IMAGE_REF: "$NEXUS_REGISTRY/$IMAGE_NAME:$CI_COMMIT_SHORT_SHA"

trivy-container-scan:
  stage: scan
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  script:
    - trivy image --format cyclonedx --output trivy-report.cdx.json "$IMAGE_REF"
    # Optional: fail the pipeline on High/Critical findings
    - trivy image --exit-code 1 --severity CRITICAL,HIGH "$IMAGE_REF"
  artifacts:
    paths:
      - trivy-report.cdx.json
    expire_in: 30 days

dependency-track-upload-image-sbom:
  stage: sbom-upload
  image: curlimages/curl:latest
  needs:
    - trivy-container-scan
  script:
    - |
      BOM_BASE64=$(base64 -w0 trivy-report.cdx.json)
      curl -X PUT "$DTRACK_URL/api/v1/bom" \
        -H "X-Api-Key: $DTRACK_API_KEY" \
        -H "Content-Type: application/json" \
        -d "{
          \"projectName\": \"$CI_PROJECT_NAME\",
          \"projectVersion\": \"$CI_COMMIT_REF_NAME\",
          \"autoCreate\": true,
          \"bom\": \"$BOM_BASE64\"
        }"
```

> Store `DTRACK_API_KEY` under **Settings > CI/CD > Variables**, masked and protected.
>
> The `--exit-code 1 --severity CRITICAL,HIGH` line in `trivy-container-scan` is what enforces fail-closed behavior at the pipeline gate (see Table VIII, Test Case 3 in `pipeline-gate-test-cases.md`). Omit it if you want detection-only behavior with triage happening in the Dependency-Track UI instead.

## 9. Verifying Integration

1. Trigger a pipeline run.
2. Confirm the `trivy-container-scan` and `dependency-track-upload-image-sbom` jobs pass.
3. In the Dependency-Track UI, navigate to **Projects > [your project] > [branch]**.
4. Check the **Audit Vulnerabilities** tab — findings from the Trivy-generated SBOM should appear with `Trivy` listed as the **Analyzer**, as seen in the `tsp-web` and `policy-web-ui-Container` project dashboards.

## 10. Post-Installation Checklist

- [ ] Change the default `admin` password immediately.
- [ ] Restrict the API key's team permissions to only what CI/CD requires (`BOM_UPLOAD`, `PROJECT_CREATION_UPLOAD`, `VIEW_PORTFOLIO`).
- [ ] Store `DTRACK_API_KEY` as a masked, protected GitLab CI/CD variable — never commit it to a repository.
- [ ] Configure a backup strategy for the `dtrack-postgres-data` volume.
- [ ] Set up notification rules (**Administration > Notifications**) to alert on new Critical/High findings.
- [ ] Consider putting Dependency-Track behind an Nginx reverse proxy with HTTPS for production use.
