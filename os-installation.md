# Server OS Installation and Configuration Guide

## Table of Contents
1. [OS Selection](#os-selection)
2. [Installation Process](#installation-process)
3. [Initial Configuration](#initial-configuration)
4. [Security Hardening](#security-hardening)
5. [System Monitoring](#system-monitoring)

## OS Selection

### Recommended Enterprise Linux Distributions
- **Red Hat Enterprise Linux (RHEL)**
  - Commercial support available
  - 10-year lifecycle
  - Enhanced security features
- **Rocky Linux/AlmaLinux**
  - RHEL-compatible
  - Free alternative
  - Community support
- **Ubuntu Server LTS**
  - 5-year support
  - Large package repository
  - Strong community

### Selection Criteria
- Support lifecycle
- Security updates frequency
- Hardware compatibility
- Team expertise
- Available support options
- License costs

## Installation Process

### Pre-Installation Steps
1. Download OS installation media
2. Verify checksums
3. Create bootable USB or configure PXE boot
4. Gather required information:
   - Network configuration
   - Storage layout
   - Host naming scheme

### Installation Methods
1. **Manual Installation**
   ```bash
   # Example disk partitioning scheme
   /boot     - 1GB
   /         - 50GB
   /var      - 20GB
   /tmp      - 10GB
   /home     - 20GB
   swap      - 2x RAM up to 8GB
   ```

2. **Automated Installation**
   - Kickstart (RHEL/Rocky)
   - Preseed (Ubuntu)
   - Example Kickstart file:
   ```bash
   # Basic Kickstart configuration
   install
   cdrom
   lang en_US.UTF-8
   keyboard us
   network --bootproto=static --ip=10.0.10.100 --netmask=255.255.255.0 --gateway=10.0.10.1 --nameserver=10.0.10.53
   rootpw --iscrypted $6$randomsalt$encrypted_password
   firewall --enabled
   selinux --enforcing
   timezone America/New_York
   bootloader --location=mbr
   text
   skipx
   zerombr
   clearpart --all --initlabel
   autopart
   reboot
   ```

## Initial Configuration

### System Settings
1. Set hostname and timezone:
   ```bash
   hostnamectl set-hostname srv01.yourdomain.com
   timedatectl set-timezone UTC
   ```

2. Configure network interfaces:
   ```bash
   # Example netplan configuration (Ubuntu)
   cat > /etc/netplan/01-netcfg.yaml << EOF
   network:
     version: 2
     renderer: networkd
     ethernets:
       ens160:
         addresses:
           - 10.0.10.100/24
         gateway4: 10.0.10.1
         nameservers:
           addresses: [10.0.10.53]
   EOF
   ```

3. Update system packages:
   ```bash
   # RHEL/Rocky
   dnf update -y

   # Ubuntu
   apt update && apt upgrade -y
   ```

### User Management
1. Create admin user:
   ```bash
   useradd -m -s /bin/bash -G sudo admin
   passwd admin
   ```

2. Configure SSH access:
   ```bash
   mkdir -p /home/admin/.ssh
   chmod 700 /home/admin/.ssh
   touch /home/admin/.ssh/authorized_keys
   chmod 600 /home/admin/.ssh/authorized_keys
   ```

## Security Hardening

### Basic Security Steps
1. Configure SSH:
   ```bash
   # /etc/ssh/sshd_config
   PermitRootLogin no
   PasswordAuthentication no
   Protocol 2
   ```

2. Set up fail2ban:
   ```bash
   # RHEL/Rocky
   dnf install fail2ban
   
   # Ubuntu
   apt install fail2ban
   ```

3. Configure basic firewall:
   ```bash
   # RHEL/Rocky (firewalld)
   firewall-cmd --permanent --add-service=ssh
   firewall-cmd --reload

   # Ubuntu (ufw)
   ufw allow ssh
   ufw enable
   ```

### SELinux/AppArmor
1. Enable and configure SELinux (RHEL/Rocky):
   ```bash
   setenforce 1
   sed -i 's/SELINUX=disabled/SELINUX=enforcing/' /etc/selinux/config
   ```

2. Configure AppArmor (Ubuntu):
   ```bash
   aa-enforce /etc/apparmor.d/*
   ```

## System Monitoring

### Basic Monitoring Setup
1. Install monitoring tools:
   ```bash
   # RHEL/Rocky
   dnf install sysstat iotop htop

   # Ubuntu
   apt install sysstat iotop htop
   ```

2. Configure system logging:
   ```bash
   # /etc/rsyslog.conf
   *.info;mail.none;authpriv.none;cron.none /var/log/messages
   authpriv.* /var/log/secure
   ```

3. Set up log rotation:
   ```bash
   # /etc/logrotate.d/syslog
   /var/log/messages {
       rotate 7
       daily
       compress
       delaycompress
       missingok
       notifempty
   }
   ```

## Next Steps
- Web server installation
- Database server setup
- Application deployment
- Backup configuration
- Monitoring system implementation

---
*This guide will be updated with more detailed sections as we progress.* 