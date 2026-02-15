# NGINX Configuration Guide

**Complete Reference for Installation, Configuration & Best Practices**

**Author:** Sajal Jana  
**Version:** NGINX 1.26.3  
**Date:** February 15, 2026

---

## Table of Contents

1. [Installation](#1-installation)
2. [File Structure & Configuration Files](#2-file-structure--configuration-files)
3. [Static Website Configuration](#3-static-website-configuration)
4. [Reverse Proxy Configuration](#4-reverse-proxy-configuration)
5. [Load Balancing Methods](#5-load-balancing-methods)
6. [SSL/TLS Configuration](#6-ssltls-configuration)
7. [Security Best Practices](#7-security-best-practices)
8. [Performance Optimization](#8-performance-optimization)
9. [Logging & Monitoring](#9-logging--monitoring)
10. [Common Commands & Troubleshooting](#10-common-commands--troubleshooting)

---

## 1. Installation

NGINX can be easily installed on RHEL/CentOS-based systems using the EPEL repository.

### Installation Steps

```bash
# Install EPEL repository
sudo yum install epel-release -y

# Install NGINX
sudo yum install nginx -y
```

### Verify Installation

```bash
nginx -v
# Output: nginx version: nginx/1.26.3
```

### Service Management

```bash
# Start NGINX
sudo systemctl start nginx

# Enable NGINX to start on boot
sudo systemctl enable nginx

# Check status
sudo systemctl status nginx
```

---

## 2. File Structure & Configuration Files

Understanding NGINX's file structure is essential for effective configuration management.

### Key Directories and Files

| Path | Description |
|------|-------------|
| `/etc/nginx/nginx.conf` | Main configuration file |
| `/etc/nginx/conf.d/` | Additional server block configurations |
| `/etc/nginx/default.d/` | Default server configurations |
| `/usr/share/nginx/html/` | Default document root for static files |
| `/var/log/nginx/` | Log files (access.log, error.log) |

### Configuration File Structure

The main configuration file `/etc/nginx/nginx.conf` includes modular configuration files using the `include` directive:

```nginx
include /etc/nginx/conf.d/*.conf;
```

---

## 3. Static Website Configuration

NGINX excels at serving static content with high performance and low resource usage.

### Basic Static Site Configuration

The default server block in `/etc/nginx/nginx.conf`:

```nginx
server {
    listen       80;
    listen       [::]:80;
    server_name  _;
    root         /usr/share/nginx/html;

    # Load configuration files for default server
    include /etc/nginx/default.d/*.conf;
}
```

### Key Directives Explained

- **`listen`** - Specifies the port and IP address (IPv4 and IPv6)
- **`server_name`** - Domain name or IP address (underscore matches any)
- **`root`** - Document root directory for static files

### Custom Static Site

To host a custom website, create a new configuration file in `/etc/nginx/conf.d/`:

```nginx
# /etc/nginx/conf.d/mywebsite.conf
server {
    listen 80;
    server_name example.com www.example.com;
    root /var/www/mywebsite;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

---

## 4. Reverse Proxy Configuration

NGINX can act as a reverse proxy, forwarding client requests to backend servers and returning their responses to clients.

### Basic Reverse Proxy Setup

Create a configuration file in `/etc/nginx/conf.d/`:

```nginx
# /etc/nginx/conf.d/my_site.conf
upstream backend_app {
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
}

server {
    listen 80;
    server_name localhost;

    location / {
        proxy_pass http://backend_app;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Essential Proxy Headers

- **`Host`** - Preserves the original Host header
- **`X-Real-IP`** - Client's actual IP address
- **`X-Forwarded-For`** - Chain of proxy IP addresses
- **`X-Forwarded-Proto`** - Original protocol (http/https)

---

## 5. Load Balancing Methods

NGINX provides several load balancing algorithms to distribute traffic across backend servers.

### Round Robin (Default)

Distributes requests evenly across all backend servers in sequential order. No additional configuration required.

```nginx
upstream backend_app {
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
}
```

### Least Connections

Routes new requests to the backend server with the fewest active connections. Best for applications with long-lived connections or varying request processing times.

```nginx
upstream backend_app {
    least_conn;
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
}
```

### IP Hash

Ensures all requests from the same client IP address are always routed to the same backend server. Essential for session persistence.

```nginx
upstream backend_app {
    ip_hash;
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
}
```

### Weighted Load Balancing

Assign different weights to backend servers based on their capacity or performance.

```nginx
upstream backend_app {
    server 127.0.0.1:3001 weight=3;
    server 127.0.0.1:3002 weight=1;
}
```

### Health Checks & Backup Servers

```nginx
upstream backend_app {
    server 127.0.0.1:3001 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:3002 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:3003 backup;
}
```

---

## 6. SSL/TLS Configuration

Securing your NGINX server with SSL/TLS is essential for protecting data in transit and establishing trust with users.

### Generate Self-Signed Certificate (Testing Only)

```bash
# Create certificate and key
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/nginx-selfsigned.key \
  -out /etc/ssl/certs/nginx-selfsigned.crt
```

### Complete SSL/TLS Configuration

> **Important:** For production environments, use certificates from a trusted Certificate Authority (CA) like Let's Encrypt.

```nginx
# /etc/nginx/conf.d/my_site.conf
upstream backend_app {
    ip_hash;
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
}

# Redirect HTTP to HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name localhost;
    return 301 https://$host$request_uri;
}

# HTTPS server block
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name localhost;

    # SSL Certificate
    ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
    ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;

    # SSL Security
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384';
    ssl_prefer_server_ciphers off;

    # SSL Session
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    # Security Headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    location / {
        proxy_pass http://backend_app;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

## 7. Security Best Practices

### Security Headers

Add these headers to all server blocks for enhanced security:

```nginx
# HTTP Strict Transport Security (HSTS)
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

# Prevent clickjacking
add_header X-Frame-Options "SAMEORIGIN" always;

# Prevent MIME type sniffing
add_header X-Content-Type-Options "nosniff" always;

# Enable XSS protection
add_header X-XSS-Protection "1; mode=block" always;

# Content Security Policy (customize as needed)
add_header Content-Security-Policy "default-src 'self' https:; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'" always;
```

### Hide NGINX Version

Add to the `http` block in `/etc/nginx/nginx.conf`:

```nginx
server_tokens off;
```

### Rate Limiting

Protect against DDoS and brute-force attacks:

```nginx
# Define rate limit zone in http block
limit_req_zone $binary_remote_addr zone=one:10m rate=10r/s;

# Apply in server or location block
location / {
    limit_req zone=one burst=20 nodelay;
    proxy_pass http://backend_app;
}
```

### Restrict Access by IP

```nginx
location /admin {
    allow 192.168.1.0/24;
    deny all;
    proxy_pass http://backend_app;
}
```

---

## 8. Performance Optimization

### Worker Processes & Connections

Configure in the main context of `/etc/nginx/nginx.conf`:

```nginx
# Set to number of CPU cores
worker_processes auto;

events {
    worker_connections 1024;
    use epoll;  # Efficient for Linux
}
```

### Buffer Sizes

Add to the `http` block:

```nginx
client_body_buffer_size 10K;
client_header_buffer_size 1k;
client_max_body_size 8m;
large_client_header_buffers 2 1k;
```

### Gzip Compression

```nginx
gzip on;
gzip_vary on;
gzip_proxied any;
gzip_comp_level 6;
gzip_types text/plain text/css text/xml text/javascript
           application/json application/javascript application/xml+rss
           application/rss+xml font/truetype font/opentype
           application/vnd.ms-fontobject image/svg+xml;
```

### Caching Static Content

```nginx
location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
    expires 30d;
    add_header Cache-Control "public, immutable";
}
```

### Connection Keepalive

```nginx
keepalive_timeout 65;
keepalive_requests 100;
```

---

## 9. Logging & Monitoring

### Log File Locations

- **Access Log:** `/var/log/nginx/access.log`
- **Error Log:** `/var/log/nginx/error.log`

### Custom Log Format

Define in the `http` block:

```nginx
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '"$status" $body_bytes_sent "$http_referer" '
                '"$http_user_agent" "$http_x_forwarded_for"';

access_log /var/log/nginx/access.log main;
```

### Disable Logging for Specific Locations

```nginx
location /health {
    access_log off;
    return 200 "healthy\n";
}
```

### Viewing Logs

```bash
# View access log in real-time
sudo tail -f /var/log/nginx/access.log

# View error log
sudo tail -f /var/log/nginx/error.log

# Search for specific errors
sudo grep -i 'error' /var/log/nginx/error.log
```

---

## 10. Common Commands & Troubleshooting

### Essential Commands

| Command | Description |
|---------|-------------|
| `nginx -t` | Test configuration for syntax errors |
| `systemctl reload nginx` | Reload configuration without downtime |
| `systemctl restart nginx` | Restart NGINX service |
| `systemctl stop nginx` | Stop NGINX service |
| `nginx -V` | Show version and compiled modules |

### Common Issues & Solutions

#### Configuration Test Fails

- Run `nginx -t` to see detailed error messages
- Check for missing semicolons or brackets
- Verify file paths and permissions

#### 502 Bad Gateway

- Backend server is down or unreachable
- Check upstream server configuration
- Verify firewall rules and SELinux settings

#### Permission Denied

- Check file/directory ownership and permissions
- SELinux may be blocking access:

```bash
sudo setsebool -P httpd_can_network_connect 1
```

#### Port Already in Use

- Check what's using the port:

```bash
sudo ss -tlnp | grep :80
```

- Stop conflicting service or change NGINX port

---

## Quick Reference Card

### Configuration Workflow

1. Edit configuration files in `/etc/nginx/`
2. Test configuration: `nginx -t`
3. Reload NGINX: `systemctl reload nginx`
4. Check logs: `tail -f /var/log/nginx/error.log`

### Key Configuration Blocks

- **Main context:** worker_processes, events
- **http block:** Global HTTP settings, logging, gzip
- **upstream block:** Backend server definitions
- **server block:** Virtual host configuration
- **location block:** URL-specific configurations

### Important Variables

- `$host` - Request hostname
- `$remote_addr` - Client IP address
- `$request_uri` - Full request URI with arguments
- `$scheme` - Request scheme (http/https)
- `$proxy_add_x_forwarded_for` - X-Forwarded-For chain

---

## Additional Resources

- **Official Documentation:** https://nginx.org/en/docs/
- **Community Forum:** https://forum.nginx.org/
- **GitHub Repository:** https://github.com/nginx/nginx
- **SSL Configuration Generator:** https://ssl-config.mozilla.org/

---

**Document prepared by Sajal Jana**

*End of Document*

