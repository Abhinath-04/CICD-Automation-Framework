# Installation Guide: Sonatype Nexus Repository Manager

## 1. System Requirements

For a stable production environment, the following hardware and software specifications are recommended:

- **CPU:** Minimum 4 vCPUs (8 recommended)
- **RAM:** Minimum 8 GB (16 GB+ recommended for large workloads)
- **Storage:** 50 GB+ (SSD preferred, scalable based on artifact volume)
- **Java:** OpenJDK 17 (required for Nexus 3.70+)
- **OS:** Ubuntu 22.04 LTS, Debian 11/12, or RHEL/CentOS 8/9

## 2. Prerequisites & User Setup

Running Nexus as a root user is a security risk. Create a dedicated service user.

### 2.1. Update System & Install Java

```bash
# For Ubuntu/Debian
sudo apt update && sudo apt upgrade -y
sudo apt install openjdk-17-jdk wget -y

# For RHEL/CentOS
sudo yum update -y
sudo yum install java-17-openjdk-devel wget -y
```

### 2.2. Create Nexus User

```bash
sudo useradd -d /opt/nexus -s /bin/bash nexus
```

## 3. Installation Steps

### 3.1. Download and Extract

Navigate to the `/opt` directory and download the latest Unix distribution.

```bash
cd /opt
sudo wget https://download.sonatype.com/nexus/3/latest-unix.tar.gz
sudo tar -xvf latest-unix.tar.gz
```

### 3.2. Organize Directories

Rename the extracted folder to a generic `nexus` name for easier management and future upgrades.

```bash
# Note: The actual folder name will vary by version (e.g., nexus-3.xx.x-xx)
sudo mv nexus-3* nexus
```

### 3.3. Set Permissions

Nexus creates a `sonatype-work` directory for its data. Both this and the application directory must be owned by the `nexus` user.

```bash
sudo chown -R nexus:nexus /opt/nexus
sudo chown -R nexus:nexus /opt/sonatype-work
```

## 4. Configuration

### 4.1. Run as User

Instruct the Nexus script to run as the dedicated user.

```bash
sudo nano /opt/nexus/bin/nexus.rc
```

```properties
# Uncomment and edit the following line:
run_as_user="nexus"
```

### 4.2. JVM Heap Memory (Optional)

If you have 8 GB+ of RAM, you may want to increase the heap size for better performance.

```bash
sudo nano /opt/nexus/bin/nexus.vmoptions
```

```properties
# Adjust -Xms and -Xmx values (e.g., -Xms4G -Xmx4G)
```

## 5. Configure Systemd Service

To ensure Nexus starts automatically on boot and handles crashes gracefully, create a systemd unit file.

```bash
sudo nano /etc/systemd/system/nexus.service
```

Paste the following content:

```ini
[Unit]
Description=nexus service
After=network.target

[Service]
Type=forking
LimitNOFILE=65536
User=nexus
Group=nexus
ExecStart=/opt/nexus/bin/nexus start
ExecStop=/opt/nexus/bin/nexus stop
Restart=on-abort

[Install]
WantedBy=multi-user.target
```

**Enable and start the service:**

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now nexus
```

## 6. Accessing the Web Interface

Nexus runs on port **8081** by default.

1. **Open Firewall:**
   ```bash
   # Ubuntu
   sudo ufw allow 8081/tcp

   # RHEL
   sudo firewall-cmd --permanent --add-port=8081/tcp && sudo firewall-cmd --reload
   ```
2. **Access UI:** Visit `http://your-server-ip:8081` in your browser.
3. **Initial Admin Password:** The system generates a random password during the first boot. Retrieve it with:
   ```bash
   cat /opt/sonatype-work/nexus3/admin.password
   ```

## 7. Securing with Nginx Reverse Proxy (Recommended)

For production, it is highly recommended to use Nginx with SSL (HTTPS).

### 7.1. Basic Nginx Config Snippet

```nginx
server {
    listen 80;
    server_name nexus.example.com;

    location / {
        proxy_pass http://127.0.0.1:8081;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

> **Note:** After setting up Nginx, use Certbot (`sudo certbot --nginx`) to enable HTTPS.

## 8. Post-Installation Checklist

- [ ] Change the default admin password immediately upon first login.
- [ ] Disable "Anonymous Access" if this is a private internal repository.
- [ ] Configure **Blob Stores** on a separate disk partition if managing high-volume artifacts.
- [ ] Set up a backup strategy for the `/opt/sonatype-work` directory.
