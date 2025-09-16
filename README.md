# PufferPanel Installation Guide

PufferPanel is a free and open-source game server management panel designed for both individuals and game server providers. It serves as a FOSS alternative to proprietary game server panels like Pterodactyl (with paid features), TCAdmin, GameCP, or AMP (Application Management Panel). PufferPanel provides a web-based interface for managing game servers including Minecraft, CS:GO, Team Fortress 2, and many more, with features for user management, server templates, and Docker integration.

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Supported Operating Systems](#supported-operating-systems)
3. [Installation](#installation)
4. [Configuration](#configuration)
5. [Service Management](#service-management)
6. [Troubleshooting](#troubleshooting)
7. [Security Considerations](#security-considerations)
8. [Performance Tuning](#performance-tuning)
9. [Backup and Restore](#backup-and-restore)
10. [System Requirements](#system-requirements)
11. [Support](#support)
12. [Contributing](#contributing)
13. [License](#license)
14. [Acknowledgments](#acknowledgments)
15. [Version History](#version-history)
16. [Appendices](#appendices)

## 1. Prerequisites

### Hardware Requirements
- **CPU**: 2+ cores (4+ recommended for multiple servers)
- **RAM**: 2GB minimum (4GB+ recommended)
- **Storage**: 20GB+ for panel and game servers
- **Network**: Static IP recommended for game hosting

### Software Requirements
- **Web Server**: Built-in or nginx/Apache for reverse proxy
- **Database**: MariaDB/MySQL 5.7+ or SQLite
- **Docker**: Optional but recommended for isolation
- **systemd**: For service management

### Network Requirements
- **Ports**: 
  - 8080: Web interface (default)
  - 5657: SFTP server (default)
  - Game ports: Varies by game (25565 for Minecraft, 27015 for Source games)

## 2. Supported Operating Systems

PufferPanel officially supports:
- RHEL 8/9 and derivatives (CentOS Stream, Rocky Linux, AlmaLinux)
- Debian 11/12
- Ubuntu 20.04 LTS / 22.04 LTS / 24.04 LTS
- Arch Linux
- Alpine Linux 3.18+
- openSUSE Leap 15.5+ / Tumbleweed
- Fedora 38+
- Windows Server 2019/2022 (limited support)

## 3. Installation

### Method 1: Official Installer (Recommended)

#### RHEL/CentOS/Rocky Linux/AlmaLinux

```bash
# Install dependencies
sudo dnf install -y curl

# Download and run installer
curl -s https://packagecloud.io/install/repositories/pufferpanel/pufferpanel/script.rpm.sh | sudo bash

# Install PufferPanel
sudo dnf install -y pufferpanel

# Install MariaDB (if not using SQLite)
sudo dnf install -y mariadb mariadb-server
sudo systemctl enable --now mariadb
sudo mysql_secure_installation

# Create database
sudo mysql -e "CREATE DATABASE pufferpanel;"
sudo mysql -e "CREATE USER 'pufferpanel'@'localhost' IDENTIFIED BY 'your_secure_password';"
sudo mysql -e "GRANT ALL PRIVILEGES ON pufferpanel.* TO 'pufferpanel'@'localhost';"
sudo mysql -e "FLUSH PRIVILEGES;"

# Enable and start PufferPanel
sudo systemctl enable --now pufferpanel

# Add first user
sudo pufferpanel user add
```

#### Debian/Ubuntu

```bash
# Add PufferPanel repository
curl -s https://packagecloud.io/install/repositories/pufferpanel/pufferpanel/script.deb.sh | sudo bash

# Update and install
sudo apt update
sudo apt install -y pufferpanel

# Install MariaDB
sudo apt install -y mariadb-server
sudo systemctl enable --now mariadb
sudo mysql_secure_installation

# Create database
sudo mysql -e "CREATE DATABASE pufferpanel;"
sudo mysql -e "CREATE USER 'pufferpanel'@'localhost' IDENTIFIED BY 'your_secure_password';"
sudo mysql -e "GRANT ALL PRIVILEGES ON pufferpanel.* TO 'pufferpanel'@'localhost';"
sudo mysql -e "FLUSH PRIVILEGES;"

# Configure PufferPanel
sudo systemctl enable --now pufferpanel

# Create admin user
sudo pufferpanel user add --admin
```

#### Arch Linux

```bash
# Install from AUR
yay -S pufferpanel

# Or manual installation
# Download latest release
PUFFERPANEL_VERSION=$(curl -s https://api.github.com/repos/pufferpanel/pufferpanel/releases/latest | grep tag_name | cut -d '"' -f 4)
wget "https://github.com/pufferpanel/pufferpanel/releases/download/${PUFFERPANEL_VERSION}/pufferpanel_${PUFFERPANEL_VERSION}_linux_amd64.deb"

# Extract binary
ar x pufferpanel_*.deb
tar -xf data.tar.xz
sudo cp -r usr/* /usr/
sudo cp -r etc/* /etc/

# Install dependencies
sudo pacman -S mariadb

# Initialize MariaDB
sudo mysql_install_db --user=mysql --basedir=/usr --datadir=/var/lib/mysql
sudo systemctl enable --now mariadb

# Continue with database setup as above
```

#### Alpine Linux

```bash
# Install dependencies
apk add --no-cache curl mariadb mariadb-client

# Download PufferPanel
PUFFERPANEL_VERSION=$(curl -s https://api.github.com/repos/pufferpanel/pufferpanel/releases/latest | grep tag_name | cut -d '"' -f 4)
wget "https://github.com/pufferpanel/pufferpanel/releases/download/${PUFFERPANEL_VERSION}/pufferpanel_${PUFFERPANEL_VERSION}_linux_amd64.tar.gz"

# Extract and install
tar -xzf pufferpanel_*.tar.gz
mv pufferpanel /usr/local/bin/
chmod +x /usr/local/bin/pufferpanel

# Create directories
mkdir -p /etc/pufferpanel /var/lib/pufferpanel /var/log/pufferpanel
```

### Method 2: Docker Installation

```bash
# Create docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  pufferpanel:
    image: pufferpanel/pufferpanel:latest
    container_name: pufferpanel
    restart: unless-stopped
    ports:
      - "8080:8080"
      - "5657:5657"
    environment:
      PUFFER_PANEL_DATABASE_DIALECT: "sqlite3"
      PUFFER_PANEL_DATABASE_URL: "file:/var/lib/pufferpanel/pufferpanel.db?cache=shared"
      PUFFER_PANEL_TOKEN_PRIVATE: "your_secret_key_here"
      PUFFER_WEB_HOST: "0.0.0.0:8080"
    volumes:
      - pufferpanel_config:/etc/pufferpanel
      - pufferpanel_data:/var/lib/pufferpanel
      - pufferpanel_servers:/var/lib/pufferpanel/servers
      - /var/run/docker.sock:/var/run/docker.sock

volumes:
  pufferpanel_config:
  pufferpanel_data:
  pufferpanel_servers:
EOF

# Start PufferPanel
docker-compose up -d

# Create admin user
docker-compose exec pufferpanel /pufferpanel user add --admin
```

### Method 3: Manual Installation

```bash
# Download binary
PUFFERPANEL_VERSION="2.6.3"  # Check latest version
wget "https://github.com/pufferpanel/pufferpanel/releases/download/v${PUFFERPANEL_VERSION}/pufferpanel_${PUFFERPANEL_VERSION}_linux_amd64.tar.gz"

# Extract files
tar -xzf pufferpanel_*.tar.gz

# Install binary
sudo mv pufferpanel /usr/local/bin/
sudo chmod +x /usr/local/bin/pufferpanel

# Create user and directories
sudo useradd -r -s /bin/false -d /var/lib/pufferpanel pufferpanel
sudo mkdir -p /etc/pufferpanel /var/lib/pufferpanel/servers /var/log/pufferpanel
sudo chown -R pufferpanel:pufferpanel /etc/pufferpanel /var/lib/pufferpanel /var/log/pufferpanel

# Generate config
sudo -u pufferpanel pufferpanel config init
```

## 4. Configuration

### Configuration File

Edit `/etc/pufferpanel/config.json`:
```json
{
  "logs": {
    "level": "info",
    "path": "/var/log/pufferpanel",
    "deleteAfter": 7
  },
  "panel": {
    "database": {
      "dialect": "mysql",
      "host": "localhost",
      "port": 3306,
      "name": "pufferpanel",
      "username": "pufferpanel",
      "password": "your_secure_password"
    },
    "web": {
      "host": "0.0.0.0:8080",
      "ssl": {
        "enabled": false,
        "cert": "/path/to/cert.pem",
        "key": "/path/to/key.pem"
      }
    },
    "sftp": {
      "host": "0.0.0.0:5657",
      "generate": true,
      "key": "/etc/pufferpanel/sftp_host_key"
    },
    "email": {
      "enabled": false,
      "from": "noreply@example.com",
      "host": "smtp.example.com",
      "port": 587,
      "username": "smtp_user",
      "password": "smtp_pass"
    },
    "settings": {
      "defaultTheme": "PufferPanel",
      "registrationEnabled": false,
      "requireEmailVerification": false
    },
    "token": {
      "private": "your_random_secret_key_here"
    }
  },
  "daemon": {
    "data": {
      "root": "/var/lib/pufferpanel/servers"
    },
    "sftp": {
      "enabled": true
    },
    "docker": {
      "enabled": true,
      "socket": "/var/run/docker.sock"
    }
  }
}
```

### Database Configuration

#### MariaDB/MySQL
```sql
-- Create database and user
CREATE DATABASE pufferpanel CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'pufferpanel'@'localhost' IDENTIFIED BY 'secure_password';
GRANT ALL PRIVILEGES ON pufferpanel.* TO 'pufferpanel'@'localhost';
FLUSH PRIVILEGES;
```

#### PostgreSQL
```sql
-- Create database and user
CREATE DATABASE pufferpanel;
CREATE USER pufferpanel WITH ENCRYPTED PASSWORD 'secure_password';
GRANT ALL PRIVILEGES ON DATABASE pufferpanel TO pufferpanel;
```

### Environment Variables

```bash
# Database configuration
export PUFFER_PANEL_DATABASE_DIALECT="mysql"
export PUFFER_PANEL_DATABASE_HOST="localhost"
export PUFFER_PANEL_DATABASE_NAME="pufferpanel"
export PUFFER_PANEL_DATABASE_USERNAME="pufferpanel"
export PUFFER_PANEL_DATABASE_PASSWORD="secure_password"

# Web configuration
export PUFFER_WEB_HOST="0.0.0.0:8080"
export PUFFER_PANEL_REGISTRATION_ENABLED="false"

# Token secret
export PUFFER_PANEL_TOKEN_PRIVATE="your_secret_key_here"
```

## 5. Service Management

### systemd Service

Create `/etc/systemd/system/pufferpanel.service`:
```ini
[Unit]
Description=PufferPanel Game Server Management Panel
After=network.target mariadb.service

[Service]
Type=simple
User=pufferpanel
Group=pufferpanel
WorkingDirectory=/var/lib/pufferpanel
ExecStart=/usr/local/bin/pufferpanel run
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal

# Security hardening
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/lib/pufferpanel /var/log/pufferpanel /etc/pufferpanel

[Install]
WantedBy=multi-user.target
```

```bash
# Enable and start service
sudo systemctl daemon-reload
sudo systemctl enable --now pufferpanel
sudo systemctl status pufferpanel

# View logs
sudo journalctl -u pufferpanel -f
```

### User Management

```bash
# Add admin user
sudo pufferpanel user add --email admin@example.com --name admin --admin

# Add regular user
sudo pufferpanel user add --email user@example.com --name user

# Change user password
sudo pufferpanel user password --email user@example.com

# List users
sudo pufferpanel user list

# Delete user
sudo pufferpanel user delete --email user@example.com
```

### Template Management

```bash
# Import official templates
sudo pufferpanel template import

# List available templates
sudo pufferpanel template list

# Create custom template
cat > minecraft.json << 'EOF'
{
  "type": "minecraft",
  "display": "Minecraft Server",
  "install": {
    "commands": [
      {
        "type": "download",
        "files": "https://launcher.mojang.com/v1/objects/server.jar"
      }
    ]
  },
  "run": {
    "stop": "stop",
    "command": "java -Xmx${memory}M -jar server.jar nogui",
    "workingDirectory": "/server"
  }
}
EOF

sudo pufferpanel template import minecraft.json
```

## 6. Troubleshooting

### Common Issues

1. **Panel not accessible**:
```bash
# Check if service is running
sudo systemctl status pufferpanel

# Check ports
sudo netstat -tlnp | grep -E "8080|5657"

# Check firewall
sudo firewall-cmd --list-ports
sudo ufw status

# Test local connection
curl http://localhost:8080
```

2. **Database connection errors**:
```bash
# Test database connection
mysql -u pufferpanel -p -h localhost pufferpanel

# Check database service
sudo systemctl status mariadb

# Verify credentials in config
sudo cat /etc/pufferpanel/config.json | jq .panel.database
```

3. **Permission issues**:
```bash
# Fix file permissions
sudo chown -R pufferpanel:pufferpanel /var/lib/pufferpanel
sudo chown -R pufferpanel:pufferpanel /etc/pufferpanel
sudo chmod -R 755 /var/lib/pufferpanel/servers

# Check Docker permissions
sudo usermod -aG docker pufferpanel
```

4. **Server creation fails**:
```bash
# Check Docker status
sudo systemctl status docker

# Verify Docker socket permissions
ls -la /var/run/docker.sock

# Check available disk space
df -h /var/lib/pufferpanel/servers

# Review daemon logs
sudo journalctl -u pufferpanel --since "10 minutes ago"
```

### Debug Mode

```bash
# Enable debug logging
sudo pufferpanel config set logs.level debug

# Run in foreground with debug
sudo -u pufferpanel pufferpanel run --debug

# Check detailed logs
tail -f /var/log/pufferpanel/pufferpanel.log
```

## 7. Security Considerations

### Web Interface Security

```nginx
# Nginx reverse proxy with SSL
server {
    listen 443 ssl http2;
    server_name panel.example.com;
    
    ssl_certificate /etc/ssl/certs/panel.crt;
    ssl_certificate_key /etc/ssl/private/panel.key;
    
    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

### Firewall Configuration

```bash
# firewalld (RHEL/CentOS)
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --permanent --add-port=5657/tcp
sudo firewall-cmd --permanent --add-port=25565/tcp  # Minecraft
sudo firewall-cmd --reload

# UFW (Ubuntu/Debian)
sudo ufw allow 8080/tcp
sudo ufw allow 5657/tcp
sudo ufw allow 25565/tcp
sudo ufw enable

# iptables
sudo iptables -A INPUT -p tcp --dport 8080 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 5657 -j ACCEPT
```

### User Permissions

```json
// Role-based permissions in PufferPanel
{
  "roles": {
    "admin": {
      "permissions": ["*"]
    },
    "moderator": {
      "permissions": [
        "server.view",
        "server.console",
        "server.start",
        "server.stop",
        "server.files.view"
      ]
    },
    "user": {
      "permissions": [
        "server.view",
        "server.console",
        "server.files.view"
      ]
    }
  }
}
```

### Docker Security

```bash
# Use Docker rootless mode
dockerd-rootless-setuptool.sh install

# Limit container resources
docker run -d \
  --name minecraft \
  --memory="2g" \
  --cpus="2.0" \
  --security-opt="no-new-privileges:true" \
  --read-only \
  pufferpanel/minecraft
```

## 8. Performance Tuning

### Panel Optimization

```json
// config.json optimizations
{
  "panel": {
    "database": {
      "maxConnections": 100,
      "connectionTimeout": 30,
      "idleTimeout": 600
    },
    "cache": {
      "enabled": true,
      "ttl": 3600
    }
  }
}
```

### Server Resource Limits

```bash
# Set resource limits per server
pufferpanel server config minecraft --memory 2048 --cpu 200

# Configure swap limit
echo "vm.swappiness=10" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### Database Optimization

```sql
-- MariaDB optimizations
SET GLOBAL innodb_buffer_pool_size = 256M;
SET GLOBAL max_connections = 200;
SET GLOBAL query_cache_size = 32M;
SET GLOBAL query_cache_type = 1;

-- Add indexes
ALTER TABLE servers ADD INDEX idx_user_id (user_id);
ALTER TABLE server_permissions ADD INDEX idx_server_user (server_id, user_id);
```

### Storage Performance

```bash
# Use separate disk for game servers
sudo mkdir -p /mnt/games
sudo mount /dev/sdb1 /mnt/games
sudo ln -s /mnt/games /var/lib/pufferpanel/servers

# Enable compression for backups
pufferpanel config set backup.compression gzip
```

## 9. Backup and Restore

### Panel Backup

```bash
#!/bin/bash
# backup-pufferpanel.sh

BACKUP_DIR="/var/backups/pufferpanel"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p $BACKUP_DIR

# Stop PufferPanel
sudo systemctl stop pufferpanel

# Backup database
mysqldump -u pufferpanel -p pufferpanel | gzip > $BACKUP_DIR/database_$DATE.sql.gz

# Backup configuration
tar -czf $BACKUP_DIR/config_$DATE.tar.gz /etc/pufferpanel

# Backup templates
tar -czf $BACKUP_DIR/templates_$DATE.tar.gz /var/lib/pufferpanel/templates

# Backup server data (optional, can be large)
tar -czf $BACKUP_DIR/servers_$DATE.tar.gz /var/lib/pufferpanel/servers

# Start PufferPanel
sudo systemctl start pufferpanel

echo "Backup completed: $BACKUP_DIR"
```

### Server Backup

```bash
# Backup individual server
pufferpanel server backup minecraft --dest /var/backups/servers/

# Automated server backups
cat > /etc/cron.d/pufferpanel-backup << 'EOF'
0 2 * * * pufferpanel /usr/local/bin/pufferpanel server backup --all --dest /var/backups/servers/
EOF
```

### Restore Procedures

```bash
# Restore database
sudo systemctl stop pufferpanel
gunzip -c database_backup.sql.gz | mysql -u pufferpanel -p pufferpanel

# Restore configuration
sudo tar -xzf config_backup.tar.gz -C /

# Restore templates
sudo tar -xzf templates_backup.tar.gz -C /

# Restore servers
sudo tar -xzf servers_backup.tar.gz -C /

# Fix permissions
sudo chown -R pufferpanel:pufferpanel /var/lib/pufferpanel /etc/pufferpanel

# Start PufferPanel
sudo systemctl start pufferpanel
```

## 10. System Requirements

### Minimum Requirements
- **CPU**: 2 cores
- **RAM**: 2GB (plus game server requirements)
- **Storage**: 20GB
- **Network**: 10 Mbps

### Recommended Requirements
- **CPU**: 4+ cores
- **RAM**: 8GB+
- **Storage**: 100GB+ SSD
- **Network**: 100 Mbps+

### Per Game Server Requirements

| Game | RAM | CPU | Storage |
|------|-----|-----|---------|
| Minecraft | 1-4GB | 1-2 cores | 10GB |
| CS:GO | 2GB | 2 cores | 30GB |
| Terraria | 1GB | 1 core | 2GB |
| Rust | 4-8GB | 3-4 cores | 25GB |
| ARK | 8-16GB | 4 cores | 60GB |

## 11. Support

### Official Resources
- **Website**: https://pufferpanel.com
- **GitHub**: https://github.com/pufferpanel/pufferpanel
- **Documentation**: https://docs.pufferpanel.com
- **Discord**: https://discord.gg/pufferpanel

### Community Support
- **Discord Server**: Active community support
- **GitHub Issues**: https://github.com/pufferpanel/pufferpanel/issues
- **Reddit**: r/PufferPanel
- **Forum**: https://community.pufferpanel.com

## 12. Contributing

### How to Contribute
1. Fork the repository on GitHub
2. Create a feature branch
3. Submit pull request
4. Follow Go coding standards
5. Include tests and documentation

### Development Setup
```bash
# Clone repository
git clone https://github.com/pufferpanel/pufferpanel.git
cd pufferpanel

# Install Go
wget https://go.dev/dl/go1.21.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.21.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin

# Install dependencies
go mod download

# Build PufferPanel
go build -o pufferpanel github.com/pufferpanel/pufferpanel/v2/cmd/pufferpanel

# Run tests
go test ./...
```

## 13. License

PufferPanel is licensed under the Apache License 2.0.

Key points:
- Free to use, modify, and distribute
- Commercial use allowed
- Patent grant included
- Must include license notice

## 14. Acknowledgments

### Credits
- **PufferPanel Team**: Core development team
- **Community Contributors**: Template creators and testers
- **Game Developers**: For server software
- **Docker**: Container technology

## 15. Version History

### Recent Releases
- **v2.6.x**: Current stable with improved Docker support
- **v2.5.x**: Enhanced template system
- **v2.4.x**: Improved security features

### Major Features by Version
- **v2.6**: Better Docker integration, improved UI
- **v2.5**: Template marketplace, OAuth support
- **v2.4**: Multi-node support

## 16. Appendices

### A. Game Server Templates

#### Minecraft Template
```json
{
  "name": "Minecraft Java",
  "type": "minecraft-java",
  "install": {
    "commands": [
      {
        "type": "download",
        "files": "https://launcher.mojang.com/v1/objects/server.jar"
      },
      {
        "type": "writefile",
        "target": "eula.txt",
        "text": "eula=true"
      }
    ]
  },
  "run": {
    "stop": "stop",
    "pre": [],
    "post": [],
    "arguments": [
      "-Xmx${memory}M",
      "-jar",
      "server.jar",
      "nogui"
    ],
    "program": "java"
  },
  "environment": {
    "type": "standard"
  },
  "data": {
    "memory": {
      "value": "1024",
      "required": true,
      "desc": "Memory in MB",
      "display": "Memory (MB)",
      "type": "integer"
    }
  }
}
```

### B. API Usage Examples

```bash
# Authenticate
TOKEN=$(curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@example.com","password":"password"}' | jq -r .token)

# List servers
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:8080/api/servers

# Create server
curl -X POST http://localhost:8080/api/servers \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "My Minecraft Server",
    "type": "minecraft-java",
    "users": ["admin@example.com"],
    "node": "local",
    "data": {
      "memory": "2048"
    }
  }'

# Server control
curl -X POST http://localhost:8080/api/servers/{id}/start \
  -H "Authorization: Bearer $TOKEN"
```

### C. Docker Compose Examples

```yaml
# Multi-server setup
version: '3.8'

services:
  panel:
    image: pufferpanel/pufferpanel:latest
    ports:
      - "8080:8080"
    volumes:
      - ./data:/var/lib/pufferpanel
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      PUFFER_PANEL_DATABASE_DIALECT: mysql
      PUFFER_PANEL_DATABASE_HOST: db
      PUFFER_PANEL_DATABASE_NAME: pufferpanel
      PUFFER_PANEL_DATABASE_USERNAME: pufferpanel
      PUFFER_PANEL_DATABASE_PASSWORD: secure_password
    depends_on:
      - db

  db:
    image: mariadb:latest
    environment:
      MYSQL_ROOT_PASSWORD: root_password
      MYSQL_DATABASE: pufferpanel
      MYSQL_USER: pufferpanel
      MYSQL_PASSWORD: secure_password
    volumes:
      - db_data:/var/lib/mysql

volumes:
  db_data:
```

### D. Security Hardening Script

```bash
#!/bin/bash
# harden-pufferpanel.sh

# Disable root login
pufferpanel config set security.disableRoot true

# Enable 2FA requirement
pufferpanel config set security.require2FA true

# Set session timeout
pufferpanel config set security.sessionTimeout 3600

# Enable brute force protection
pufferpanel config set security.maxLoginAttempts 5
pufferpanel config set security.lockoutDuration 900

# Restrict file uploads
pufferpanel config set security.maxUploadSize 104857600
pufferpanel config set security.allowedExtensions ".txt,.properties,.yml,.json"

# Enable audit logging
pufferpanel config set logs.auditEnabled true
pufferpanel config set logs.auditFile "/var/log/pufferpanel/audit.log"

echo "Security hardening complete"
```

---

For more information and updates, visit https://github.com/howtomgr/pufferpanel