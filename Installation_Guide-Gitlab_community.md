# Installation Guide: GitLab Community Edition (Self-Managed)

## 1. System Requirements

GitLab is a resource-intensive application suite (including PostgreSQL, Redis, and Gitaly). The following specifications are required for a stable environment:

- **CPU:** Minimum 4 vCPUs
- **RAM:** Minimum 4 GB (8 GB highly recommended to avoid performance degradation)
- **Storage:** 50 GB+ (SSD preferred, scaled based on repository volume)
- **OS Support:** RHEL 8/9, AlmaLinux, Rocky Linux, Ubuntu 20.04/22.04 LTS

## 2. Prerequisites

### 2.1. Update System & Install Dependencies

GitLab requires specific helper packages for SSH management and mail notifications.

**For RHEL / AlmaLinux / Rocky Linux:**
```bash
sudo dnf update -y
sudo dnf install -y curl policycoreutils-python-utils perl openssh-server
```

**For Ubuntu / Debian:**
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl openssh-server ca-certificates tzdata perl
```

### 2.2. Configure Postfix (Optional but Recommended)

To allow GitLab to send notification emails, install and enable Postfix.

```bash
# RHEL/AlmaLinux
sudo dnf install postfix -y
sudo systemctl enable --now postfix

# Ubuntu/Debian
sudo apt install postfix -y
```

> **Note:** During Postfix installation, select "Internet Site" and use your server's FQDN as the mail name.

## 3. Installation Steps

### 3.1. Add the GitLab Repository

Download and execute the official GitLab repository script.

```bash
# This script automatically detects your OS version
curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh | sudo bash
```

> For Ubuntu/Debian, use `script.deb.sh` instead of `script.rpm.sh`.

### 3.2. Install GitLab CE

Set your server's external URL during the installation to trigger automatic configuration. Replace `http://gitlab.example.com` with your actual domain or IP address.

```bash
# RHEL/AlmaLinux/Rocky
sudo EXTERNAL_URL="http://your-server-ip" dnf install gitlab-ce -y

# Ubuntu/Debian
sudo EXTERNAL_URL="http://your-server-ip" apt install gitlab-ce -y
```

## 4. Firewall Configuration

GitLab requires ports **80** (HTTP) and **443** (HTTPS) for the web UI, and **22** for SSH (Git operations).

**For RHEL (Firewalld):**
```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-service=ssh
sudo systemctl reload firewalld
```

**For Ubuntu (UFW):**
```bash
sudo ufw allow http
sudo ufw allow https
sudo ufw allow OpenSSH
```

## 5. Initial Access & Security

### 5.1. Retrieve Initial Admin Password

GitLab generates a random temporary password for the `root` user. It is valid for 24 hours.

```bash
sudo cat /etc/gitlab/initial_root_password
```

### 5.2. Login to Web Interface

Navigate to `http://your-server-ip` in your browser.

1. Log in with:
   - **Username:** `root`
   - **Password:** (the password retrieved in the step above)
2. **Critical Security Step:** Change your password immediately via **User Settings > Password**.

## 6. Configuration Management

The primary configuration file for GitLab is located at `/etc/gitlab/gitlab.rb`.

If you need to change the URL, enable SSL, or configure SMTP later, edit this file:

```bash
sudo nano /etc/gitlab/gitlab.rb
```

After making any changes to this file, you **must** apply them by running:

```bash
sudo gitlab-ctl reconfigure
```

## 7. Maintenance & Useful Commands

GitLab provides a CLI tool, `gitlab-ctl`, for all management tasks:

| Command | Description |
|---|---|
| `sudo gitlab-ctl status` | View the status of all GitLab services (Postgres, Nginx, etc.) |
| `sudo gitlab-ctl restart` | Restarts all GitLab components |
| `sudo gitlab-ctl tail` | View real-time logs for troubleshooting |
| `sudo gitlab-backup create` | Manually trigger a full repository and database backup |
