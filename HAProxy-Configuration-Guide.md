# HAProxy Configuration Guide

**Complete Reference for Installation, Configuration & Best Practices**

**Author:** Sajal Jana  
**Version:** HAProxy 3.0.5  
**Date:** February 15, 2026

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Prerequisites & Environment Setup](#2-prerequisites--environment-setup)
3. [Installation](#3-installation)
4. [Configuration File Structure](#4-configuration-file-structure)
5. [Kubernetes API Load Balancing](#5-kubernetes-api-load-balancing)
6. [Advanced Load Balancing Algorithms](#6-advanced-load-balancing-algorithms)
7. [Health Checks & Monitoring](#7-health-checks--monitoring)
8. [High Availability Setup](#8-high-availability-setup)
9. [Security Best Practices](#9-security-best-practices)
10. [SSL/TLS Termination](#10-ssltls-termination)
11. [Performance Tuning](#11-performance-tuning)
12. [Logging & Statistics](#12-logging--statistics)
13. [Common Commands & Troubleshooting](#13-common-commands--troubleshooting)

---

## 1. Introduction

HAProxy (High Availability Proxy) is a free, open-source software that provides a high availability load balancer and reverse proxy for TCP and HTTP-based applications. It's particularly well-suited for high traffic websites and is used by many high-profile organizations.

### Key Features

- **High Performance:** Can handle tens of thousands of connections with low resource usage
- **Load Balancing:** Multiple algorithms including round-robin, least connections, and source IP hashing
- **Health Checks:** Active and passive health monitoring of backend servers
- **SSL/TLS Support:** Native SSL termination and offloading
- **High Availability:** Session persistence and failover capabilities
- **Statistics:** Built-in web interface for monitoring

### Use Cases

- Load balancing Kubernetes API servers
- Web application load balancing
- Database read replica distribution
- Microservices traffic management
- SSL/TLS termination

---

## 2. Prerequisites & Environment Setup

### System Requirements

- RHEL/CentOS 8+ or compatible Linux distribution
- Minimum 1GB RAM (2GB+ recommended for production)
- Network connectivity between HAProxy and backend servers
- Root or sudo access

### Network Configuration

Before installing HAProxy, configure hostname resolution on all servers in your cluster.

#### Configure /etc/hosts

Add the following entries to `/etc/hosts` on **all servers** in your infrastructure:

```bash
# Edit hosts file
vi /etc/hosts
```

```
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

# HAProxy Load Balancer
192.168.1.34  loadbalancer.example.com loadbalancer

# Kubernetes Master Nodes
192.168.1.30  master2.example.com      master2
192.168.1.31  master1.example.com      master1

# Kubernetes Worker Nodes
192.168.1.32  workernode1.example.com  workernode1
192.168.1.33  workernode2.example.com  workernode2
```

#### Verify Hostname Resolution

```bash
# Test DNS resolution
ping -c 2 master1.example.com
ping -c 2 master2.example.com
ping -c 2 loadbalancer.example.com
```

### Firewall & SELinux Configuration

> **Note:** For development/testing environments, you may disable firewall and SELinux. For production, configure appropriate rules instead.

#### Development/Testing (Disable Security)

```bash
# Disable firewall
systemctl stop firewalld
systemctl disable firewalld

# Disable SELinux temporarily
setenforce 0

# Disable SELinux permanently
sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
```

#### Production (Configure Firewall Rules)

```bash
# Allow HAProxy ports
firewall-cmd --permanent --add-port=6443/tcp  # Kubernetes API
firewall-cmd --permanent --add-port=80/tcp    # HTTP
firewall-cmd --permanent --add-port=443/tcp   # HTTPS
firewall-cmd --permanent --add-port=8404/tcp  # HAProxy Stats
firewall-cmd --reload

# Configure SELinux for HAProxy
setsebool -P haproxy_connect_any 1
```

---

## 3. Installation

### Check Available Versions

```bash
# List available HAProxy packages
dnf list available 'haproxy*'
```

### Install HAProxy

```bash
# Install HAProxy
dnf install haproxy -y

# Verify installation
rpm -qi haproxy
```

**Expected Output:**
```
Name        : haproxy
Version     : 3.0.5
Release     : ...
Architecture: x86_64
Install Date: ...
Group       : System Environment/Daemons
Size        : ...
License     : GPLv2+
Signature   : ...
Source RPM  : haproxy-3.0.5...
Summary     : HAProxy - High Performance TCP/HTTP Load Balancer
```

### Enable and Start HAProxy

```bash
# Enable HAProxy to start on boot
systemctl enable haproxy

# Start HAProxy service
systemctl start haproxy

# Check status
systemctl status haproxy
```

### Verify Installation

```bash
# Check HAProxy version
haproxy -v

# Test configuration syntax
haproxy -c -f /etc/haproxy/haproxy.cfg
```

---

## 4. Configuration File Structure

HAProxy uses a single configuration file located at `/etc/haproxy/haproxy.cfg`. The file is divided into several sections:

### Configuration Sections

```
┌─────────────────────────────────────┐
│         global                      │  ← Global settings
├─────────────────────────────────────┤
│         defaults                    │  ← Default parameters
├─────────────────────────────────────┤
│         frontend                    │  ← Entry points
├─────────────────────────────────────┤
│         backend                     │  ← Server pools
├─────────────────────────────────────┤
│         listen (optional)           │  ← Combined frontend+backend
└─────────────────────────────────────┘
```

### Section Descriptions

| Section | Purpose | Required |
|---------|---------|----------|
| `global` | Process-wide settings (user, logging, performance) | Yes |
| `defaults` | Default settings inherited by frontend/backend | Yes |
| `frontend` | Client-facing entry points | Yes* |
| `backend` | Server pools for load balancing | Yes* |
| `listen` | Simplified frontend+backend combined | Optional |

*Either frontend/backend or listen sections required

### Basic Configuration Template

```haproxy
#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    log         127.0.0.1 local2
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

#---------------------------------------------------------------------
# Common defaults
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option                  http-server-close
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

#---------------------------------------------------------------------
# Frontend configuration
#---------------------------------------------------------------------
frontend <name>
    bind <ip>:<port>
    default_backend <backend-name>

#---------------------------------------------------------------------
# Backend configuration
#---------------------------------------------------------------------
backend <name>
    balance roundrobin
    server <name> <ip>:<port> check
```

---

## 5. Kubernetes API Load Balancing

This is the most common use case for HAProxy in Kubernetes environments - load balancing the Kubernetes API server across multiple control plane nodes.

### Complete Configuration

Edit the HAProxy configuration file:

```bash
vi /etc/haproxy/haproxy.cfg
```

Add the following configuration:

```haproxy
#---------------------------------------------------------------------
# FRONTEND: Entry point for Kubernetes API requests
#---------------------------------------------------------------------
frontend kubernetes-frontend
    bind 0.0.0.0:6443
    # Listen on all network interfaces on port 6443
    # This makes the load balancer accessible from any network
    
    option tcplog
    # Enable detailed TCP connection logging
    # Logs include connection timestamps, bytes transferred, and connection state
    
    mode tcp
    # Operate in raw TCP mode (Layer 4)
    # Required for HTTPS traffic passthrough without SSL termination
    # HAProxy forwards encrypted traffic without inspecting contents
    
    default_backend kubernetes-backend
    # Forward all incoming requests to the kubernetes-backend server pool

#---------------------------------------------------------------------
# BACKEND: Load-balanced pool of Kubernetes API servers
#---------------------------------------------------------------------
backend kubernetes-backend
    mode tcp
    # Backend also processes connections in TCP mode
    # Maintains consistency with frontend configuration
    
    balance roundrobin
    # Load balancing algorithm: round-robin
    # Distributes requests evenly across all healthy backend servers
    # Each new connection goes to the next server in rotation
    
    option tcp-check
    # Perform TCP-level health checks
    # Verifies that backend servers are alive and accepting connections
    # Failed servers are automatically removed from the pool
    
    # Backend server definitions
    server master01 172.24.42.229:6443 check
    server master02 172.24.41.95:6443 check
    # Format: server <name> <ip>:<port> <options>
    # 'check' enables active health monitoring for each server
    # HAProxy will periodically connect to verify server availability
```

### Configuration Breakdown

#### Frontend Section (`kubernetes-frontend`)

| Directive | Value | Purpose |
|-----------|-------|---------|
| `bind` | `0.0.0.0:6443` | Listen on all interfaces, port 6443 |
| `option tcplog` | - | Enable TCP connection logging |
| `mode` | `tcp` | Layer 4 (TCP) load balancing |
| `default_backend` | `kubernetes-backend` | Route traffic to backend pool |

#### Backend Section (`kubernetes-backend`)

| Directive | Value | Purpose |
|-----------|-------|---------|
| `mode` | `tcp` | Layer 4 operation |
| `balance` | `roundrobin` | Even distribution algorithm |
| `option tcp-check` | - | Enable health checks |
| `server` | `master01 172.24.42.229:6443 check` | Backend server definition |

### Using Hostname-Based Configuration

Instead of IP addresses, you can use hostnames configured in `/etc/hosts`:

```haproxy
backend kubernetes-backend
    mode tcp
    balance roundrobin
    option tcp-check
    
    server master01 master1.example.com:6443 check
    server master02 master2.example.com:6443 check
```

### Apply Configuration

```bash
# Test configuration syntax
haproxy -c -f /etc/haproxy/haproxy.cfg

# Restart HAProxy to apply changes
systemctl restart haproxy

# Verify service is running
systemctl status haproxy
```

### Verify Load Balancing

```bash
# Test connection to load balancer
curl -k https://192.168.1.34:6443

# Check HAProxy logs
tail -f /var/log/messages | grep haproxy

# Test from kubectl (if configured)
kubectl --server=https://192.168.1.34:6443 get nodes
```

---

## 6. Advanced Load Balancing Algorithms

HAProxy supports multiple load balancing algorithms. Choose based on your application requirements.

### Round Robin (Default)

Distributes requests evenly across all servers in sequential order.

```haproxy
backend app-backend
    balance roundrobin
    server app01 192.168.1.10:80 check
    server app02 192.168.1.11:80 check
    server app03 192.168.1.12:80 check
```

**Best for:** Servers with equal capacity, stateless applications

### Least Connections

Routes new connections to the server with the fewest active connections.

```haproxy
backend app-backend
    balance leastconn
    server app01 192.168.1.10:80 check
    server app02 192.168.1.11:80 check
```

**Best for:** Long-lived connections, varying processing times

### Source IP Hash

Routes requests from the same source IP to the same backend server.

```haproxy
backend app-backend
    balance source
    hash-type consistent
    server app01 192.168.1.10:80 check
    server app02 192.168.1.11:80 check
```

**Best for:** Session persistence without cookies

### URI Hash

Routes requests for the same URI to the same backend server.

```haproxy
backend cache-backend
    balance uri
    hash-type consistent
    server cache01 192.168.1.20:80 check
    server cache02 192.168.1.21:80 check
```

**Best for:** Caching layers, content distribution

### Weighted Round Robin

Assigns different weights to servers based on capacity.

```haproxy
backend app-backend
    balance roundrobin
    server app01 192.168.1.10:80 check weight 100
    server app02 192.168.1.11:80 check weight 50
    server app03 192.168.1.12:80 check weight 150
```

**Best for:** Heterogeneous server capacities

### Algorithm Comparison

| Algorithm | Session Persistence | Load Distribution | Use Case |
|-----------|-------------------|------------------|----------|
| `roundrobin` | No | Even | Stateless apps |
| `leastconn` | No | Based on load | Long connections |
| `source` | Yes (by IP) | Even | Session-based apps |
| `uri` | Yes (by URI) | Even | Caching |
| `rdp-cookie` | Yes (by cookie) | Even | RDP sessions |

---

## 7. Health Checks & Monitoring

### TCP Health Checks (Default)

Basic connectivity check - verifies server accepts connections.

```haproxy
backend app-backend
    mode tcp
    option tcp-check
    server app01 192.168.1.10:80 check
```

### HTTP Health Checks

Performs HTTP GET request to verify application is responding.

```haproxy
backend web-backend
    mode http
    option httpchk GET /health
    http-check expect status 200
    server web01 192.168.1.10:80 check
    server web02 192.168.1.11:80 check
```

### Advanced HTTP Health Checks

```haproxy
backend api-backend
    mode http
    option httpchk GET /api/health HTTP/1.1\r\nHost:\ api.example.com
    http-check expect string "healthy"
    server api01 192.168.1.10:8080 check
    server api02 192.168.1.11:8080 check
```

### SSL/HTTPS Health Checks

```haproxy
backend secure-backend
    mode tcp
    option ssl-hello-chk
    server secure01 192.168.1.10:443 check
```

### Custom Health Check Parameters

```haproxy
backend app-backend
    option httpchk GET /health
    server app01 192.168.1.10:80 check inter 5s rise 2 fall 3
```

**Parameters:**
- `inter 5s` - Check interval (every 5 seconds)
- `rise 2` - Server is considered UP after 2 successful checks
- `fall 3` - Server is considered DOWN after 3 failed checks

### Health Check Best Practices

```haproxy
backend production-backend
    # Health check configuration
    option httpchk GET /healthz
    http-check expect status 200
    
    # Server definitions with comprehensive checks
    server prod01 192.168.1.10:80 check inter 3s rise 2 fall 3 maxconn 500
    server prod02 192.168.1.11:80 check inter 3s rise 2 fall 3 maxconn 500
    server prod03 192.168.1.12:80 check inter 3s rise 2 fall 3 maxconn 500 backup
```

**Explanation:**
- `check` - Enable health checks
- `inter 3s` - Check every 3 seconds
- `rise 2` - 2 successes = server UP
- `fall 3` - 3 failures = server DOWN
- `maxconn 500` - Limit connections per server
- `backup` - Use only when other servers are down

---

## 8. High Availability Setup

### Multi-Master Configuration

For true high availability, run HAProxy on multiple nodes with Keepalived for failover.

#### HAProxy Configuration (Same on Both Nodes)

```haproxy
frontend kubernetes-frontend
    bind 0.0.0.0:6443
    option tcplog
    mode tcp
    default_backend kubernetes-backend

backend kubernetes-backend
    mode tcp
    balance roundrobin
    option tcp-check
    server master01 192.168.1.30:6443 check
    server master02 192.168.1.31:6443 check
```

#### Keepalived Configuration (Node 1)

Create `/etc/keepalived/keepalived.conf`:

```
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    
    authentication {
        auth_type PASS
        auth_pass secretpass
    }
    
    virtual_ipaddress {
        192.168.1.100/24
    }
}
```

#### Keepalived Configuration (Node 2)

```
vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 51
    priority 90
    advert_int 1
    
    authentication {
        auth_type PASS
        auth_pass secretpass
    }
    
    virtual_ipaddress {
        192.168.1.100/24
    }
}
```

### Backup Servers

Designate backup servers that only receive traffic when primary servers are down:

```haproxy
backend app-backend
    balance roundrobin
    server app01 192.168.1.10:80 check
    server app02 192.168.1.11:80 check
    server app03 192.168.1.12:80 check backup
```

### Connection Limits & Queuing

```haproxy
backend app-backend
    balance roundrobin
    option httpchk
    
    # Limit connections per server
    server app01 192.168.1.10:80 check maxconn 1000
    server app02 192.168.1.11:80 check maxconn 1000
    
    # Queue connections when servers are full
    timeout queue 30s
```

---

## 9. Security Best Practices

### Access Control Lists (ACLs)

Restrict access to specific IP ranges or networks:

```haproxy
frontend web-frontend
    bind *:80
    
    # Define ACLs
    acl network_allowed src 192.168.1.0/24
    acl network_allowed src 10.0.0.0/8
    acl admin_path path_beg /admin
    
    # Apply rules
    http-request deny if admin_path !network_allowed
    
    default_backend web-backend
```

### Rate Limiting

Protect against DDoS and abuse:

```haproxy
frontend web-frontend
    bind *:80
    
    # Track client request rate
    stick-table type ip size 100k expire 30s store http_req_rate(10s)
    http-request track-sc0 src
    http-request deny if { sc_http_req_rate(0) gt 100 }
    
    default_backend web-backend
```

### Header Security

Add security headers to responses:

```haproxy
frontend web-frontend
    bind *:443 ssl crt /etc/haproxy/certs/cert.pem
    
    # Security headers
    http-response set-header X-Frame-Options "SAMEORIGIN"
    http-response set-header X-Content-Type-Options "nosniff"
    http-response set-header X-XSS-Protection "1; mode=block"
    http-response set-header Strict-Transport-Security "max-age=31536000"
    
    default_backend web-backend
```

### Hide HAProxy Version

```haproxy
defaults
    option httplog
    option dontlognull
    option http-server-close
    option forwardfor except 127.0.0.0/8
    option redispatch
    retries 3
    timeout http-request 10s
    timeout queue 1m
    timeout connect 10s
    timeout client 1m
    timeout server 1m
    
    # Hide version information
    option hide-version
```

---

## 10. SSL/TLS Termination

### SSL Termination (Decrypt at HAProxy)

```haproxy
frontend https-frontend
    bind *:443 ssl crt /etc/haproxy/certs/cert.pem
    mode http
    
    # Redirect HTTP to HTTPS
    redirect scheme https if !{ ssl_fc }
    
    default_backend web-backend

backend web-backend
    mode http
    balance roundrobin
    server web01 192.168.1.10:80 check
    server web02 192.168.1.11:80 check
```

### SSL Passthrough (No Decryption)

```haproxy
frontend https-frontend
    bind *:443
    mode tcp
    option tcplog
    default_backend web-backend

backend web-backend
    mode tcp
    balance roundrobin
    server web01 192.168.1.10:443 check
    server web02 192.168.1.11:443 check
```

### Generate Self-Signed Certificate

```bash
# Create certificate directory
mkdir -p /etc/haproxy/certs

# Generate certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/haproxy/certs/cert.pem \
  -out /etc/haproxy/certs/cert.pem \
  -subj "/C=US/ST=State/L=City/O=Organization/CN=loadbalancer.example.com"

# Set permissions
chmod 600 /etc/haproxy/certs/cert.pem
```

### SSL Best Practices

```haproxy
frontend https-frontend
    bind *:443 ssl crt /etc/haproxy/certs/cert.pem ssl-min-ver TLSv1.2 ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384
    
    # Force HTTPS
    http-request redirect scheme https unless { ssl_fc }
    
    # HSTS header
    http-response set-header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
    
    default_backend web-backend
```

---

## 11. Performance Tuning

### Global Settings Optimization

```haproxy
global
    log 127.0.0.1 local0
    chroot /var/lib/haproxy
    pidfile /var/run/haproxy.pid
    
    # Performance tuning
    maxconn 50000
    nbproc 4
    cpu-map auto:1/1-4 0-3
    
    # Buffers
    tune.bufsize 32768
    tune.maxrewrite 1024
    
    # SSL optimization
    ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256
    ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets
    
    user haproxy
    group haproxy
    daemon
```

### Connection Timeout Optimization

```haproxy
defaults
    mode http
    log global
    option httplog
    option dontlognull
    
    # Optimized timeouts
    timeout connect 5s
    timeout client 50s
    timeout server 50s
    timeout http-keep-alive 5s
    timeout http-request 15s
    timeout queue 30s
    timeout tunnel 1h
    
    # Connection optimization
    option http-server-close
    option forwardfor
    maxconn 10000
```

### Backend Optimization

```haproxy
backend app-backend
    balance roundrobin
    
    # Connection pooling
    option http-keep-alive
    option prefer-last-server
    
    # Compression
    compression algo gzip
    compression type text/html text/plain text/css text/javascript application/javascript application/json
    
    server app01 192.168.1.10:80 check maxconn 5000
    server app02 192.168.1.11:80 check maxconn 5000
```

---

## 12. Logging & Statistics

### Enable Logging

```haproxy
global
    log 127.0.0.1 local2
    chroot /var/lib/haproxy
    pidfile /var/run/haproxy.pid
    maxconn 4000
    user haproxy
    group haproxy
    daemon

defaults
    log global
    option httplog
    option dontlognull
```

### Configure rsyslog

Create `/etc/rsyslog.d/haproxy.conf`:

```
$ModLoad imudp
$UDPServerRun 514
$UDPServerAddress 127.0.0.1

local2.* /var/log/haproxy.log
```

Restart rsyslog:

```bash
systemctl restart rsyslog
```

### Enable Statistics Page

```haproxy
frontend stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 30s
    stats admin if TRUE
    
    # Optional authentication
    stats auth admin:password123
```

Access statistics at: `http://192.168.1.34:8404/stats`

### Advanced Statistics Configuration

```haproxy
listen stats
    bind *:8404
    mode http
    stats enable
    stats uri /haproxy?stats
    stats realm HAProxy\ Statistics
    stats auth admin:SecurePassword123
    stats admin if TRUE
    
    # Customize appearance
    stats show-legends
    stats show-node
    stats refresh 10s
```

### View Logs

```bash
# Real-time log monitoring
tail -f /var/log/haproxy.log

# Search for errors
grep -i error /var/log/haproxy.log

# Connection statistics
grep "backend" /var/log/haproxy.log | tail -20
```

---

## 13. Common Commands & Troubleshooting

### Essential Commands

| Command | Description |
|---------|-------------|
| `systemctl start haproxy` | Start HAProxy service |
| `systemctl stop haproxy` | Stop HAProxy service |
| `systemctl restart haproxy` | Restart HAProxy service |
| `systemctl reload haproxy` | Reload configuration without downtime |
| `systemctl status haproxy` | Check service status |
| `haproxy -c -f /etc/haproxy/haproxy.cfg` | Test configuration syntax |
| `haproxy -v` | Show version |
| `haproxy -vv` | Show version and build options |

### Testing Configuration

```bash
# Test configuration syntax
haproxy -c -f /etc/haproxy/haproxy.cfg

# Test configuration with detailed output
haproxy -c -V -f /etc/haproxy/haproxy.cfg

# Show current configuration
cat /etc/haproxy/haproxy.cfg
```

### Connection Testing

```bash
# Test TCP connection
telnet 192.168.1.34 6443

# Test with netcat
nc -zv 192.168.1.34 6443

# Test with curl (HTTP/HTTPS)
curl -v http://192.168.1.34:80
curl -k -v https://192.168.1.34:443
```

### Monitoring & Debugging

```bash
# Check HAProxy is listening
netstat -tulpn | grep haproxy
ss -tulpn | grep haproxy

# View active connections
ss -tan | grep :6443

# Monitor logs in real-time
tail -f /var/log/haproxy.log
journalctl -u haproxy -f

# Check backend server status
echo "show servers state" | socat stdio /var/lib/haproxy/stats
```

### Common Issues & Solutions

#### Issue: Configuration test fails

```bash
# Check syntax errors
haproxy -c -f /etc/haproxy/haproxy.cfg

# Common causes:
# - Missing semicolons or brackets
# - Invalid directives
# - File permission issues
```

#### Issue: HAProxy won't start

```bash
# Check status and logs
systemctl status haproxy -l
journalctl -u haproxy --no-pager

# Common causes:
# - Port already in use
# - Permission denied
# - Invalid configuration
```

#### Issue: Backend servers showing as DOWN

```bash
# Check health check configuration
# Verify backend servers are running
systemctl status <backend-service>

# Test direct connection to backend
curl http://192.168.1.10:80

# Check HAProxy logs for health check failures
grep "Health check" /var/log/haproxy.log
```

#### Issue: Port already in use

```bash
# Find process using port
lsof -i :6443
ss -tulpn | grep :6443

# Kill process or change HAProxy port
kill -9 <PID>
```

#### Issue: Connection timeout

```bash
# Check firewall rules
firewall-cmd --list-all

# Verify network connectivity
ping 192.168.1.10
telnet 192.168.1.10 80

# Check timeout values in configuration
grep timeout /etc/haproxy/haproxy.cfg
```

### Performance Analysis

```bash
# Check HAProxy process stats
pidof haproxy | xargs ps -p

# Monitor connections
watch -n 1 'ss -tan | grep :6443 | wc -l'

# Check system resources
top -p $(pidof haproxy)
htop -p $(pidof haproxy)
```

---

## Quick Reference Card

### Configuration Workflow

1. Edit configuration: `vi /etc/haproxy/haproxy.cfg`
2. Test syntax: `haproxy -c -f /etc/haproxy/haproxy.cfg`
3. Reload configuration: `systemctl reload haproxy`
4. Check logs: `tail -f /var/log/haproxy.log`
5. Monitor stats: `http://server:8404/stats`

### Key Configuration Directives

**Global Section:**
- `maxconn` - Maximum connections
- `log` - Logging configuration
- `user/group` - Process ownership

**Frontend Section:**
- `bind` - Listening address/port
- `mode` - tcp or http
- `default_backend` - Target backend

**Backend Section:**
- `balance` - Load balancing algorithm
- `server` - Backend server definition
- `option httpchk` - Health check method

### Load Balancing Algorithms

- `roundrobin` - Even distribution
- `leastconn` - Least connections
- `source` - Source IP hash
- `uri` - URI hash

### Health Check Options

- `check` - Enable health checks
- `inter <time>` - Check interval
- `rise <count>` - Successful checks to mark UP
- `fall <count>` - Failed checks to mark DOWN

---

## Additional Resources

- **Official Documentation:** https://www.haproxy.org/documentation.html
- **Configuration Manual:** https://cbonte.github.io/haproxy-dconv/
- **Community Forum:** https://discourse.haproxy.org/
- **GitHub Repository:** https://github.com/haproxy/haproxy
- **HAProxy Technologies:** https://www.haproxy.com/

---

## Example Configurations

### Complete Production Example

```haproxy
#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    log 127.0.0.1 local2
    chroot /var/lib/haproxy
    pidfile /var/run/haproxy.pid
    maxconn 50000
    user haproxy
    group haproxy
    daemon
    
    # SSL settings
    ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256
    ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

#---------------------------------------------------------------------
# Defaults
#---------------------------------------------------------------------
defaults
    mode http
    log global
    option httplog
    option dontlognull
    option http-server-close
    option forwardfor except 127.0.0.0/8
    option redispatch
    retries 3
    timeout http-request 10s
    timeout queue 1m
    timeout connect 10s
    timeout client 1m
    timeout server 1m
    timeout http-keep-alive 10s
    timeout check 10s
    maxconn 30000

#---------------------------------------------------------------------
# Statistics
#---------------------------------------------------------------------
listen stats
    bind *:8404
    mode http
    stats enable
    stats uri /stats
    stats realm HAProxy\ Statistics
    stats auth admin:SecurePassword123
    stats refresh 30s

#---------------------------------------------------------------------
# Kubernetes API Frontend
#---------------------------------------------------------------------
frontend kubernetes-api
    bind *:6443
    mode tcp
    option tcplog
    default_backend kubernetes-masters

#---------------------------------------------------------------------
# Kubernetes API Backend
#---------------------------------------------------------------------
backend kubernetes-masters
    mode tcp
    balance roundrobin
    option tcp-check
    server master01 192.168.1.30:6443 check inter 3s rise 2 fall 3
    server master02 192.168.1.31:6443 check inter 3s rise 2 fall 3

#---------------------------------------------------------------------
# HTTP Frontend
#---------------------------------------------------------------------
frontend http-in
    bind *:80
    mode http
    
    # Redirect to HTTPS
    redirect scheme https code 301 if !{ ssl_fc }

#---------------------------------------------------------------------
# HTTPS Frontend
#---------------------------------------------------------------------
frontend https-in
    bind *:443 ssl crt /etc/haproxy/certs/cert.pem
    mode http
    
    # Security headers
    http-response set-header X-Frame-Options "SAMEORIGIN"
    http-response set-header X-Content-Type-Options "nosniff"
    http-response set-header X-XSS-Protection "1; mode=block"
    http-response set-header Strict-Transport-Security "max-age=31536000"
    
    default_backend web-servers

#---------------------------------------------------------------------
# Web Application Backend
#---------------------------------------------------------------------
backend web-servers
    mode http
    balance leastconn
    option httpchk GET /health
    http-check expect status 200
    
    # Compression
    compression algo gzip
    compression type text/html text/plain text/css application/javascript
    
    # Servers
    server web01 192.168.1.10:80 check inter 5s rise 2 fall 3 maxconn 1000
    server web02 192.168.1.11:80 check inter 5s rise 2 fall 3 maxconn 1000
    server web03 192.168.1.12:80 check inter 5s rise 2 fall 3 maxconn 1000 backup
```

---

**Document prepared by Sajal Jana**

*End of Document*
