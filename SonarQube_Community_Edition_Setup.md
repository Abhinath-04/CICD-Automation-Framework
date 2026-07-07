# SonarQube Community Edition

## Part 1: SonarQube Server Installation (Linux)

### 1. System Requirements

- **CPU:** 2+ vCPUs
- **RAM:** 4 GB minimum (8 GB recommended)
- **Java:** OpenJDK 17 (required for SonarQube 10.x)
- **Database:** PostgreSQL 15 (SonarQube no longer supports MySQL/Oracle for Community Edition)
- **OS Settings:** Requires specific kernel limits for its internal Elasticsearch engine

### 2. System Prerequisites

#### 2.1. Kernel Tuning

SonarQube uses an embedded Elasticsearch. You must increase the virtual memory and file descriptor limits.

```bash
sudo nano /etc/sysctl.conf
```

```properties
# Add the following lines:
vm.max_map_count=262144
fs.file-max=65536
```

```bash
# Apply changes
sudo sysctl -p
```

#### 2.2. Install OpenJDK 17

```bash
# For Ubuntu/Debian
sudo apt install openjdk-17-jdk -y

# For RHEL/AlmaLinux
sudo dnf install java-17-openjdk-devel -y
```

### 3. Database Configuration (PostgreSQL)

SonarQube requires a dedicated database.

```bash
# Install PostgreSQL (RHEL example)
sudo dnf install postgresql-server postgresql-contrib -y
sudo postgresql-setup --initdb
sudo systemctl enable --now postgresql

# Switch to postgres user and create sonar user/db
sudo -u postgres psql
```

**Inside the SQL prompt:**

```sql
CREATE USER sonar WITH ENCRYPTED PASSWORD 'YourSecurePassword';
CREATE DATABASE sonarqube OWNER sonar;
GRANT ALL PRIVILEGES ON DATABASE sonarqube TO sonar;
\q
```

### 4. SonarQube Installation

#### 4.1. Download and Extract

```bash
sudo useradd -b /opt/sonarqube -s /bin/bash sonar
cd /opt
sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-10.4.1.88267.zip
sudo unzip sonarqube-*.zip
sudo mv sonarqube-10.4.1.88267 sonarqube
sudo chown -R sonar:sonar /opt/sonarqube
```

#### 4.2. Configure sonar.properties

Edit the configuration to link the database.

```bash
sudo nano /opt/sonarqube/conf/sonar.properties
```

Uncomment and edit the following:

```properties
sonar.jdbc.username=sonar
sonar.jdbc.password=YourSecurePassword
sonar.jdbc.url=jdbc:postgresql://localhost/sonarqube
sonar.web.port=9000
```

### 5. Configure Systemd Service

```bash
sudo nano /etc/systemd/system/sonarqube.service
```

Paste the following:

```ini
[Unit]
Description=SonarQube service
After=syslog.target network.target

[Service]
Type=forking
User=sonar
Group=sonar
ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop
LimitNOFILE=65536
LimitNPROC=4096

[Install]
WantedBy=multi-user.target
```

**Start the service:**

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now sonarqube
```

## Part 2: Integration with Eclipse IDE

To integrate SonarQube with Eclipse, use the **SonarLint** plugin. This allows developers to see issues in the IDE that match the rules configured on the SonarQube server.

### 1. Install SonarLint in Eclipse

1. Open **Eclipse IDE**.
2. Go to **Help > Eclipse Marketplace...**
3. Search for **"SonarLint"**.
4. Click **Install** and follow the prompts to restart Eclipse.

### 2. Connect Eclipse to SonarQube Server

1. In Eclipse, go to **Window > Show View > Other...**
2. Search for **SonarLint** and select **SonarLint Bindings**.
3. In the SonarLint Bindings view, right-click and select **Connect to a SonarQube server...**
4. **Select Server Type:** SonarQube.
5. **Server URL:** `http://your-server-ip:9000`
6. **Authentication:**
   - Login to your SonarQube Web UI.
   - Go to **My Account > Security**.
   - Generate a **User Token**.
   - Paste this token into the Eclipse prompt.
7. Click **Finish**.

### 3. Bind Your Project

1. Right-click on your project in the **Project Explorer**.
2. Select **SonarLint > Configure Binding...**
3. Choose the connection you created and select the corresponding project on the SonarQube server.
4. Click **Finish**.

## Part 3: Post-Installation Checklist

| Category | Item |
|---|---|
| **Security** | Change default admin/admin credentials immediately. |
| **Connectivity** | Ensure firewall port 9000 is open. |
| **Automation** | Integrate SonarQube into your GitLab CI/CD pipeline (using `sonar-scanner`). |
| **Real-time** | SonarLint in Eclipse will highlight "Code Smells," "Bugs," and "Vulnerabilities" directly in the code editor as you type. |
