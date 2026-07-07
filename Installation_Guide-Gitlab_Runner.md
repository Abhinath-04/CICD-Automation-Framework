# Installation Guide: GitLab Runner (Self-Managed)

## 1. System Overview

A GitLab Runner is the lightweight agent that runs your CI/CD jobs. For security and performance, it is highly recommended to host the Runner on a **separate machine** from your GitLab instance.

### 1.1. Recommended Specifications

- **CPU:** 2 vCPUs (scales with job concurrency)
- **RAM:** 2 GB – 4 GB (more if using Docker-in-Docker or heavy builds)
- **Storage:** 20 GB+ (SSD preferred for faster build caching)
- **OS:** RHEL 8/9, AlmaLinux, Rocky Linux, or Ubuntu 20.04/22.04 LTS

## 2. Installation Steps

### 2.1. Add the Official GitLab Runner Repository

**For RHEL / AlmaLinux / Rocky Linux:**
```bash
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.rpm.sh" | sudo bash
```

**For Ubuntu / Debian:**
```bash
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
```

### 2.2. Install the Runner Package

**For RHEL / AlmaLinux / Rocky Linux:**
```bash
sudo dnf install gitlab-runner -y
```

**For Ubuntu / Debian:**
```bash
sudo apt-get install gitlab-runner -y
```

## 3. Registration (New Workflow — GitLab 16.0+)

As of GitLab 16.0, the "Registration Token" method is deprecated. You must now create a Runner in the GitLab UI first to obtain an **Authentication Token** (prefixed with `glrt-`).

### 3.1. Create Runner in GitLab UI

1. In GitLab, go to **Settings > CI/CD > Runners**.
2. Click **New project runner** (or "New instance runner" if you are an admin).
3. Specify the **Platform** (Linux), **Tags**, and **Description**.
4. Click **Create runner**.
5. **Copy the "Authentication Token"** provided on the next screen.

### 3.2. Register on the Linux Server

Run the following command on your server, replacing the placeholders:

```bash
sudo gitlab-runner register \
  --url https://your-gitlab-domain.com \
  --token YOUR_AUTHENTICATION_TOKEN
```

- **Enter the GitLab instance URL:** (e.g., `http://192.168.1.10` or `https://gitlab.example.com`)
- **Enter the executor:** Most common choices are `shell` (runs directly on the host) or `docker` (runs in isolated containers).

## 4. Choosing an Executor

| Executor | Use Case | Requirements |
|---|---|---|
| **Shell** | Simplest setup; runs jobs as the `gitlab-runner` user | Build tools (Node, Java, etc.) must be installed on the host |
| **Docker** | Clean, isolated environments for every job | Docker must be installed on the host server |

### Setup for Docker Executor (Optional)

If you chose `docker` during registration:

1. **Install Docker:** `sudo dnf install docker` (RHEL) or `sudo apt install docker.io` (Ubuntu).
2. **Edit Config:** Open `/etc/gitlab-runner/config.toml` to set the default image (e.g., `image = "alpine:latest"`).

## 5. Managing the Runner Service

The Runner operates as a system service. Use these commands for maintenance:

| Action | Command |
|---|---|
| Check Status | `sudo gitlab-runner status` |
| Start Service | `sudo gitlab-runner start` |
| Restart Service | `sudo gitlab-runner restart` |
| Verify Connectivity | `sudo gitlab-runner verify` |
| View Config | `sudo cat /etc/gitlab-runner/config.toml` |

## 6. Advanced Configuration (config.toml)

You can adjust concurrency and environment variables by editing the configuration file:

```bash
sudo nano /etc/gitlab-runner/config.toml
```

**Example for high-concurrency builds:**

```toml
concurrent = 4  # Allow up to 4 jobs to run simultaneously
check_interval = 0

[[runners]]
  name = "Production-Runner-01"
  url = "https://gitlab.example.com"
  token = "glrt-xxxxxx"
  executor = "docker"
  [runners.docker]
    tls_verify = false
    image = "ubuntu:22.04"
    privileged = false
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
```

## 7. Post-Installation Checklist

- [ ] Verify the Runner appears with a **green circle (Online)** status in the GitLab UI.
- [ ] If using the **Shell** executor, ensure the `gitlab-runner` user has permissions to the directories it needs.
- [ ] Test a sample `.gitlab-ci.yml` file in a repository to ensure the job picks up correctly.
- [ ] Configure log rotation for `/var/log/gitlab-runner` to prevent disk saturation.
