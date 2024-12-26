# SSH Connection and Security Guide

[← Back to IP Assignment](ip-assignment.md) | [Next: SSL Setup →](ssl-certificate.md)

---

## Table of Contents
1. [SSH Key Generation](#ssh-key-generation)
2. [SSH Connection Methods](#ssh-connection-methods)
3. [SSH Configuration](#ssh-configuration)
4. [Security Best Practices](#security-best-practices)
5. [Troubleshooting](#troubleshooting)

## SSH Key Generation

### Creating SSH Keys (Optional/Advanced)
```bash
# Generate RSA key (4096 bits recommended)
ssh-keygen -t rsa -b 4096 -C "your_email@domain.com"

# Generate Ed25519 key (more secure, recommended)
ssh-keygen -t ed25519 -C "your_email@domain.com"

# Generate key with custom filename
ssh-keygen -t ed25519 -f ~/.ssh/vps_key -C "vps_access"
```

### Key Management (Optional/Advanced)
```bash
# Set correct permissions for SSH directory and keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub

# Copy public key to server
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server_ip
# OR manually
cat ~/.ssh/id_ed25519.pub | ssh user@server_ip "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

## SSH Connection Methods

### Basic Connection
```bash
# Connect using password (not recommended for production)
ssh username@server_ip

# Connect using SSH key (recommended)
ssh -i ~/.ssh/id_ed25519 username@server_ip

# Connect using custom port (optional)
ssh -p 2222 username@server_ip
```

### Advanced Connection Options
```bash
# Connect with compression (useful for slow connections)
ssh -C username@server_ip

# Connect with X11 forwarding
ssh -X username@server_ip

# Connect with verbose output (debugging)
ssh -v username@server_ip
```

## SSH Configuration

### Client Configuration
```bash
# ~/.ssh/config
# Default settings for all hosts
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
    HashKnownHosts yes
    AddKeysToAgent yes

# Specific host configuration
Host vps-prod
    HostName 203.0.113.10
    User admin
    Port 22
    IdentityFile ~/.ssh/vps_key
    ForwardAgent no

# Development server with tunnel
Host vps-dev
    HostName 203.0.113.11
    User developer
    Port 2222
    LocalForward 8080 localhost:80
    IdentityFile ~/.ssh/dev_key
```

### Server Configuration
```bash
# /etc/ssh/sshd_config
# Basic security settings
Port 22
Protocol 2
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
PermitEmptyPasswords no
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2

# Restrict access
AllowUsers admin developer
AllowGroups ssh-users

# Logging
SyslogFacility AUTH
LogLevel VERBOSE
```

## Security Best Practices

### Access Control
1. **Key-based Authentication Only**
   ```bash
   # Disable password authentication
   sudo sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
   sudo systemctl restart sshd
   ```

2. **Restrict Root Login**
   ```bash
   # Disable root login
   sudo sed -i 's/#PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
   sudo systemctl restart sshd
   ```

### Port and Access Management
```bash
# Change default SSH port
sudo sed -i 's/#Port 22/Port 2222/' /etc/ssh/sshd_config

# Configure firewall for new SSH port
# For UFW
sudo ufw allow 2222/tcp

# For firewalld
sudo firewall-cmd --permanent --add-port=2222/tcp
sudo firewall-cmd --reload
```

### Fail2Ban Configuration
```bash
# /etc/fail2ban/jail.local
[sshd]
enabled = true
port = 2222
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 3600
findtime = 600
```

## Troubleshooting

### Common Issues and Solutions

1. **Permission Issues**
   ```bash
   # Check and fix permissions
   chmod 700 ~/.ssh
   chmod 600 ~/.ssh/id_ed25519
   chmod 644 ~/.ssh/id_ed25519.pub
   chmod 600 ~/.ssh/config
   ```

2. **Connection Issues**
   ```bash
   # Debug connection
   ssh -vvv username@server_ip

   # Test SSH server
   nc -vz server_ip 22

   # Check SSH service status
   sudo systemctl status sshd
   ```

3. **Key Issues**
   ```bash
   # Verify key is added to agent
   ssh-add -l

   # Add key to agent
   ssh-add ~/.ssh/id_ed25519

   # Check server authorized keys
   cat ~/.ssh/authorized_keys
   ```

### SSH Monitoring and Logging
```bash
# Monitor SSH attempts
sudo tail -f /var/log/auth.log | grep sshd

# Check failed login attempts
sudo grep "Failed password" /var/log/auth.log

# Monitor active SSH connections
who | grep pts
```

## Next Steps
- Configure SSH certificates
- Set up SSH jump hosts
- Implement SSH audit logging
- Configure SSH tunneling for services

---
*This guide will be updated with more detailed sections as we progress.* 