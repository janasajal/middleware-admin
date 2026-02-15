# Squid Proxy Server Configuration Guide

**Complete Reference for Installation, Configuration & Best Practices**

**Author:** Sajal Jana  
**Version:** Squid 5.x/6.x  
**Date:** February 15, 2026

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Installation](#2-installation)
3. [Basic Configuration](#3-basic-configuration)
4. [Access Control Lists (ACLs)](#4-access-control-lists-acls)
5. [Port and Protocol Configuration](#5-port-and-protocol-configuration)
6. [Website Filtering](#6-website-filtering)
7. [SSL/HTTPS Configuration](#7-sslhttps-configuration)
8. [Authentication](#8-authentication)
9. [Logging & Monitoring](#9-logging--monitoring)
10. [Cache Configuration](#10-cache-configuration)
11. [Performance Tuning](#11-performance-tuning)
12. [Security Best Practices](#12-security-best-practices)
13. [Transparent Proxy Setup](#13-transparent-proxy-setup)
14. [Common Commands & Troubleshooting](#14-common-commands--troubleshooting)

---

## 1. Introduction

Squid is a high-performance proxy caching server for web clients, supporting HTTP, HTTPS, FTP, and more. It reduces bandwidth usage and improves response times by caching frequently requested web pages.

### Key Features

- **HTTP/HTTPS Proxy:** Forward and reverse proxy capabilities
- **Caching:** Reduces bandwidth and improves performance
- **Access Control:** Fine-grained control over who can access what
- **Content Filtering:** Block or allow specific websites and content
- **Authentication:** Multiple authentication methods (Basic, NTLM, LDAP)
- **SSL Bumping:** Inspect encrypted HTTPS traffic
- **High Performance:** Handles thousands of concurrent connections

### Use Cases

- **Corporate Internet Gateway:** Control and monitor employee internet access
- **Content Filtering:** Block malicious or inappropriate websites
- **Bandwidth Optimization:** Cache frequently accessed content
- **Security:** Filter malware, inspect traffic, enforce policies
- **Anonymity:** Hide client IP addresses from destination servers
- **Load Balancing:** Distribute traffic across multiple backend servers

### Architecture Overview

```
┌─────────────┐         ┌─────────────┐         ┌─────────────┐
│   Client    │ ──────> │    Squid    │ ──────> │  Internet   │
│  (Browser)  │ <────── │   Proxy     │ <────── │   Servers   │
└─────────────┘         └─────────────┘         └─────────────┘
                              │
                              ▼
                        ┌─────────────┐
                        │    Cache    │
                        │   Storage   │
                        └─────────────┘
```

---

## 2. Installation

### System Requirements

- RHEL/CentOS 8+ or compatible Linux distribution
- Minimum 512MB RAM (2GB+ recommended for production)
- Adequate disk space for cache (depends on usage)
- Root or sudo access

### Install Squid

```bash
# Install Squid proxy server
dnf install squid -y

# Or using yum (older systems)
yum install squid -y
```

### Verify Installation

```bash
# Check installed version
squid -v

# Check package information
rpm -qi squid
```

**Expected Output:**
```
Squid Cache: Version 5.x or 6.x
Service Name: squid
```

### Enable and Start Squid

```bash
# Enable Squid to start on boot and start it now
systemctl enable --now squid

# Alternative (separate commands):
systemctl enable squid
systemctl start squid

# Check service status
systemctl status squid
```

### Verify Squid is Running

```bash
# Check if Squid is listening on port 3128 (default)
netstat -tulpn | grep squid
ss -tulpn | grep squid

# Check Squid process
ps aux | grep squid
```

### Firewall Configuration

```bash
# Allow Squid proxy port (8080 in our configuration)
firewall-cmd --permanent --add-port=8080/tcp

# For HTTPS proxy (if configured)
firewall-cmd --permanent --add-port=8443/tcp

# Reload firewall
firewall-cmd --reload

# Verify rules
firewall-cmd --list-ports
```

---

## 3. Basic Configuration

The main Squid configuration file is located at `/etc/squid/squid.conf`.

### Configuration File Location

```bash
# Main configuration file
/etc/squid/squid.conf

# Default configuration backup
/etc/squid/squid.conf.default

# Additional configuration directory
/etc/squid/conf.d/
```

### Basic Forward Proxy Configuration

Edit the configuration file:

```bash
vi /etc/squid/squid.conf
```

**Complete Basic Configuration:**

```squid
#=========================================================================
# SQUID FORWARD PROXY CONFIGURATION
# Author: Sajal Jana
# Purpose: Forward proxy with content filtering and access control
#=========================================================================

#-------------------------------------------------------------------------
# Network Configuration
#-------------------------------------------------------------------------

# HTTP proxy port
http_port 8080
# Squid listens on port 8080 for HTTP proxy requests
# Clients must configure this port in their browser/system proxy settings

# HTTPS proxy port (optional - requires SSL certificate)
# https_port 8443 cert=/etc/squid/ssl_cert/squid.pem key=/etc/squid/ssl_cert/squid.key
# Uncomment to enable HTTPS proxy support
# Requires valid SSL certificate and private key

#-------------------------------------------------------------------------
# Access Control Lists (ACLs) - Define Rules
#-------------------------------------------------------------------------

# SSL/HTTPS ports
acl SSL_ports port 443
# Define port 443 as the standard HTTPS port

# Safe ports - ports that are generally considered safe
acl Safe_ports port 80          # HTTP
acl Safe_ports port 21          # FTP
acl Safe_ports port 443         # HTTPS
acl Safe_ports port 70          # Gopher
acl Safe_ports port 210         # WAIS
acl Safe_ports port 1025-65535  # Unregistered ports (for applications)
acl Safe_ports port 280         # HTTP management
acl Safe_ports port 488         # GSS-HTTP
acl Safe_ports port 591         # FileMaker
acl Safe_ports port 777         # Multiling HTTP
acl Safe_ports port 8443        # Alternative HTTPS

# CONNECT method ACL
acl CONNECT method CONNECT
# CONNECT is used for SSL/HTTPS tunneling

#-------------------------------------------------------------------------
# Security Rules - Deny Unsafe Traffic
#-------------------------------------------------------------------------

# Deny requests to unsafe ports
http_access deny !Safe_ports
# Block access to any port not defined in Safe_ports
# Protects against connections to potentially dangerous services

# Deny CONNECT to non-SSL ports
http_access deny CONNECT !SSL_ports
# Only allow CONNECT method (used for HTTPS) to SSL ports
# Prevents abuse of CONNECT for non-SSL tunneling

#-------------------------------------------------------------------------
# Cache Manager Access
#-------------------------------------------------------------------------

# Only allow cache manager access from localhost
http_access allow localhost manager
http_access deny manager
# Cache manager (cachemgr.cgi) should only be accessible locally
# Prevents unauthorized access to Squid statistics and controls

#-------------------------------------------------------------------------
# Content Filtering - Website Blocking
#-------------------------------------------------------------------------

# DEMO: Block social media and time-wasting websites
acl blocked_sites dstdomain .facebook.com .twitter.com .instagram.com
http_access deny blocked_sites
# Blocks access to Facebook, Twitter, and Instagram
# The leading dot (.) matches all subdomains (www.facebook.com, m.facebook.com, etc.)

#-------------------------------------------------------------------------
# Allowed Websites (Whitelist Approach)
#-------------------------------------------------------------------------

# Allow specific websites only
acl allowed_sites dstdomain .bing.com .google.com
http_access allow allowed_sites
# Only allows access to Bing and Google
# All other websites will be blocked (unless other rules permit)

#-------------------------------------------------------------------------
# Default Access Policy
#-------------------------------------------------------------------------

# WARNING: This allows ALL traffic - FOR DEMO ONLY!
# http_access allow all
# NEVER use "allow all" in production without proper ACLs above it
# This would bypass all security controls

# Deny all other access (default deny policy)
http_access deny all
# Final rule: deny everything not explicitly allowed
# This is security best practice (whitelist approach)

#-------------------------------------------------------------------------
# Logging Configuration
#-------------------------------------------------------------------------

# Access log - records all client requests
access_log stdio:/var/log/squid/access.log squid
# Uses stdio method for better reliability (avoids pipe issues)
# Format: stdio:<path> <format>

# Cache log - records cache operations and errors
cache_log /var/log/squid/cache.log
# Contains cache statistics, errors, and operational messages

#-------------------------------------------------------------------------
# Cache Directory
#-------------------------------------------------------------------------

# Core dump directory (for debugging)
coredump_dir /var/spool/squid
# Location where Squid writes core dumps if it crashes

# Cache directory configuration (optional - add if caching is needed)
# cache_dir ufs /var/spool/squid 10000 16 256
# Format: cache_dir <type> <path> <size_mb> <L1_dirs> <L2_dirs>
# This would create a 10GB cache with 16 first-level and 256 second-level directories

#=========================================================================
# END OF CONFIGURATION
#=========================================================================
```

### Configuration Validation

```bash
# Test configuration for syntax errors
squid -k parse

# Or use:
squid -k check

# If successful, no output means configuration is valid
```

### Apply Configuration

```bash
# Reload configuration without dropping connections
systemctl reload squid

# Or restart (drops active connections)
systemctl restart squid

# Verify service is running
systemctl status squid
```

---

## 4. Access Control Lists (ACLs)

ACLs are the foundation of Squid's access control system. They define conditions that can be used in access rules.

### ACL Syntax

```squid
acl <name> <type> <value(s)>
```

### Common ACL Types

#### Source IP/Network ACLs

```squid
# Single IP address
acl admin_pc src 192.168.1.100

# IP range (CIDR notation)
acl internal_network src 192.168.1.0/24

# Multiple networks
acl company_network src 192.168.1.0/24 10.0.0.0/8

# Exclude specific IP
acl blocked_client src 192.168.1.50
```

#### Destination Domain ACLs

```squid
# Single domain
acl google dstdomain google.com

# Domain with all subdomains (leading dot)
acl google_all dstdomain .google.com

# Multiple domains
acl social_media dstdomain .facebook.com .twitter.com .instagram.com .tiktok.com

# Domain from file
acl blocked_domains dstdomain "/etc/squid/blocked_domains.txt"
```

#### Port ACLs

```squid
# Single port
acl http_port port 80

# Multiple ports
acl web_ports port 80 443 8080

# Port range
acl high_ports port 1024-65535
```

#### Time-Based ACLs

```squid
# Business hours (Monday-Friday, 9 AM - 5 PM)
acl business_hours time MTWHF 09:00-17:00

# Lunch break
acl lunch_time time MTWHFAS 12:00-13:00

# Weekends
acl weekends time SA

# Specific days and hours
acl maintenance_window time T 02:00-04:00
```

#### URL Pattern ACLs

```squid
# URL regex pattern
acl streaming_sites url_regex -i youtube|vimeo|dailymotion
# -i makes it case-insensitive

# URL path contains
acl downloads urlpath_regex -i \.(exe|zip|mp3|mp4|avi)$

# URL from file
acl video_sites url_regex "/etc/squid/video_sites.txt"
```

#### Method ACLs

```squid
# HTTP methods
acl POST_requests method POST
acl GET_requests method GET
acl CONNECT_method method CONNECT
```

#### Protocol ACLs

```squid
# HTTP protocol
acl http_protocol proto HTTP

# FTP protocol
acl ftp_protocol proto FTP

# SSL/HTTPS (via CONNECT)
acl SSL_protocol proto HTTPS
```

### ACL Combination Examples

```squid
# Allow admin from specific IP during business hours
acl admin_ip src 192.168.1.100
acl business_hours time MTWHF 09:00-17:00
http_access allow admin_ip business_hours

# Block social media except during lunch
acl social_media dstdomain .facebook.com .twitter.com .instagram.com
acl lunch_time time MTWHFAS 12:00-13:00
http_access deny social_media !lunch_time
http_access allow social_media lunch_time

# Allow downloads only from internal network
acl internal_network src 192.168.1.0/24
acl downloads urlpath_regex -i \.(exe|zip|pdf)$
http_access allow downloads internal_network
http_access deny downloads
```

### External ACL Files

Create `/etc/squid/blocked_domains.txt`:

```
facebook.com
twitter.com
instagram.com
tiktok.com
reddit.com
```

Reference in squid.conf:

```squid
acl blocked_sites dstdomain "/etc/squid/blocked_domains.txt"
http_access deny blocked_sites
```

---

## 5. Port and Protocol Configuration

### Standard HTTP Proxy

```squid
# Listen on port 8080 for HTTP proxy
http_port 8080
```

### Multiple Ports

```squid
# Listen on multiple ports
http_port 3128
http_port 8080
http_port 8888
```

### Listen on Specific Interface

```squid
# Listen only on specific IP address
http_port 192.168.1.10:8080

# Listen on all interfaces (default)
http_port 0.0.0.0:8080
```

### Transparent Proxy Mode

```squid
# Transparent proxy (no browser configuration needed)
http_port 3128 intercept

# For IPv6
http_port [::]:3128 intercept
```

### SSL Bump (HTTPS Interception)

```squid
# HTTPS interception port
https_port 3129 intercept ssl-bump \
    cert=/etc/squid/ssl_cert/squid.pem \
    key=/etc/squid/ssl_cert/squid.key \
    generate-host-certificates=on \
    dynamic_cert_mem_cache_size=4MB

# SSL bump configuration
acl step1 at_step SslBump1
ssl_bump peek step1
ssl_bump bump all

# SSL certificate database
sslcrtd_program /usr/lib64/squid/security_file_certgen -s /var/lib/squid/ssl_db -M 4MB
```

---

## 6. Website Filtering

### Block Specific Websites

```squid
# Block social media
acl social_media dstdomain .facebook.com .twitter.com .instagram.com .tiktok.com
http_access deny social_media

# Block streaming sites
acl streaming dstdomain .youtube.com .netflix.com .hulu.com .twitch.tv
http_access deny streaming

# Block adult content (example)
acl adult_content dstdomain .pornhub.com .xvideos.com
http_access deny adult_content
```

### Allow Only Specific Websites (Whitelist)

```squid
# Define allowed sites
acl allowed_sites dstdomain .google.com .bing.com .wikipedia.org .github.com

# Allow only these sites
http_access allow allowed_sites

# Deny everything else
http_access deny all
```

### Block by URL Pattern

```squid
# Block URLs containing specific keywords
acl blocked_keywords url_regex -i porn|adult|casino|gambling|warez

# Block file downloads by extension
acl executables urlpath_regex -i \.(exe|msi|bat|com|scr)$

# Apply rules
http_access deny blocked_keywords
http_access deny executables
```

### Time-Based Filtering

```squid
# Block social media during work hours
acl social_media dstdomain .facebook.com .twitter.com .instagram.com
acl work_hours time MTWHF 09:00-17:00

http_access deny social_media work_hours
http_access allow social_media
```

### Category-Based Filtering

Create category files:

**/etc/squid/categories/social_media.txt:**
```
facebook.com
twitter.com
instagram.com
tiktok.com
snapchat.com
```

**/etc/squid/categories/streaming.txt:**
```
youtube.com
netflix.com
hulu.com
twitch.tv
vimeo.com
```

**squid.conf:**
```squid
acl social_media dstdomain "/etc/squid/categories/social_media.txt"
acl streaming dstdomain "/etc/squid/categories/streaming.txt"

http_access deny social_media
http_access deny streaming
```

### Bandwidth Throttling for Specific Sites

```squid
# Define streaming sites
acl streaming dstdomain .youtube.com .netflix.com

# Create delay pool
delay_pools 1
delay_class 1 1
delay_parameters 1 32000/32000
delay_access 1 allow streaming
delay_access 1 deny all
```

---

## 7. SSL/HTTPS Configuration

### Generate SSL Certificate for HTTPS Proxy

```bash
# Create certificate directory
mkdir -p /etc/squid/ssl_cert

# Generate self-signed certificate
openssl req -new -newkey rsa:2048 -days 365 -nodes -x509 \
    -keyout /etc/squid/ssl_cert/squid.key \
    -out /etc/squid/ssl_cert/squid.pem \
    -subj "/C=US/ST=State/L=City/O=Organization/CN=proxy.example.com"

# Combine certificate and key (if needed)
cat /etc/squid/ssl_cert/squid.key /etc/squid/ssl_cert/squid.pem > /etc/squid/ssl_cert/squid-combined.pem

# Set permissions
chmod 600 /etc/squid/ssl_cert/*
chown squid:squid /etc/squid/ssl_cert/*
```

### Enable HTTPS Proxy Port

```squid
# HTTPS proxy listener
https_port 8443 cert=/etc/squid/ssl_cert/squid.pem key=/etc/squid/ssl_cert/squid.key
```

### SSL Bump (Decrypt and Inspect HTTPS)

```bash
# Initialize SSL certificate database
/usr/lib64/squid/security_file_certgen -c -s /var/lib/squid/ssl_db -M 4MB
chown -R squid:squid /var/lib/squid/ssl_db
```

**squid.conf:**
```squid
# SSL bump port
https_port 3129 intercept ssl-bump \
    cert=/etc/squid/ssl_cert/squid.pem \
    generate-host-certificates=on \
    dynamic_cert_mem_cache_size=4MB

# SSL certificate generation program
sslcrtd_program /usr/lib64/squid/security_file_certgen -s /var/lib/squid/ssl_db -M 4MB
sslcrtd_children 5

# SSL bump rules
acl step1 at_step SslBump1
acl step2 at_step SslBump2
acl step3 at_step SslBump3

# Peek at step 1 to get server certificate
ssl_bump peek step1

# Bump (intercept) all other traffic
ssl_bump bump all

# Don't bump certain sites (banking, etc.)
acl nobump_sites dstdomain .bank.com .paypal.com
ssl_bump splice nobump_sites
```

> **Important:** SSL bumping requires installing Squid's CA certificate on client machines.

### Export CA Certificate for Clients

```bash
# Copy CA certificate for distribution
cp /etc/squid/ssl_cert/squid.pem /var/www/html/squid-ca.pem

# Clients should import this certificate as a trusted root CA
```

---

## 8. Authentication

### Basic Authentication (htpasswd)

#### Setup

```bash
# Install httpd-tools for htpasswd
dnf install httpd-tools -y

# Create password file
htpasswd -c /etc/squid/passwd username1
# Enter password when prompted

# Add more users (without -c flag)
htpasswd /etc/squid/passwd username2

# Set permissions
chmod 640 /etc/squid/passwd
chown squid:squid /etc/squid/passwd
```

#### Configure Squid

```squid
# Basic authentication
auth_param basic program /usr/lib64/squid/basic_ncsa_auth /etc/squid/passwd
auth_param basic children 5
auth_param basic realm Squid Proxy Server
auth_param basic credentialsttl 2 hours

# ACL for authenticated users
acl authenticated_users proxy_auth REQUIRED

# Require authentication
http_access allow authenticated_users
http_access deny all
```

### LDAP Authentication

```squid
# LDAP authentication
auth_param basic program /usr/lib64/squid/basic_ldap_auth \
    -b "dc=example,dc=com" \
    -D "cn=admin,dc=example,dc=com" \
    -w "admin_password" \
    -f "uid=%s" \
    -h ldap.example.com

auth_param basic children 5
auth_param basic realm Squid Proxy
auth_param basic credentialsttl 2 hours

acl authenticated_users proxy_auth REQUIRED
http_access allow authenticated_users
```

### Active Directory Authentication (NTLM)

```squid
# NTLM authentication (for Windows AD)
auth_param ntlm program /usr/bin/ntlm_auth --helper-protocol=squid-2.5-ntlmssp
auth_param ntlm children 5
auth_param ntlm keep_alive on

# Fallback to basic auth
auth_param basic program /usr/bin/ntlm_auth --helper-protocol=squid-2.5-basic
auth_param basic children 5
auth_param basic realm Squid Proxy

acl authenticated_users proxy_auth REQUIRED
http_access allow authenticated_users
```

### Per-User Access Control

```squid
# Define users
acl admin_users proxy_auth admin root sajal
acl regular_users proxy_auth user1 user2 user3

# Admins have full access
http_access allow admin_users

# Regular users limited access
acl allowed_sites dstdomain .google.com .wikipedia.org
http_access allow regular_users allowed_sites
http_access deny regular_users
```

---

## 9. Logging & Monitoring

### Log File Locations

```bash
# Access log (all HTTP requests)
/var/log/squid/access.log

# Cache log (cache operations and errors)
/var/log/squid/cache.log

# Store log (object storage information)
/var/log/squid/store.log
```

### Access Log Configuration

```squid
# Standard access log
access_log /var/log/squid/access.log squid

# Using stdio (more reliable)
access_log stdio:/var/log/squid/access.log squid

# Multiple log formats
access_log /var/log/squid/access.log squid
access_log /var/log/squid/custom.log combined

# Disable access logging
# access_log none
```

### Access Log Formats

```squid
# Common format
logformat common %>a %[ui %[un [%tl] "%rm %ru HTTP/%rv" %>Hs %<st %Ss:%Sh

# Combined format (Apache-like)
logformat combined %>a %[ui %[un [%tl] "%rm %ru HTTP/%rv" %>Hs %<st "%{Referer}>h" "%{User-Agent}>h" %Ss:%Sh

# Custom format
logformat custom %ts.%03tu %>a %Ss/%03>Hs %<st %rm %ru %[un %Sh/%<a %mt
```

### Cache Log Configuration

```squid
# Cache log location
cache_log /var/log/squid/cache.log

# Debug level (0-9, 0=critical only, 9=verbose)
debug_options ALL,1

# Specific section debugging
debug_options ALL,1 28,3 33,5
# Section 28 (access control) at level 3
# Section 33 (client side) at level 5
```

### View Logs in Real-Time

```bash
# Access log
tail -f /var/log/squid/access.log

# Cache log
tail -f /var/log/squid/cache.log

# Both logs
tail -f /var/log/squid/*.log

# Filter for specific domain
tail -f /var/log/squid/access.log | grep facebook.com

# Count requests per IP
cat /var/log/squid/access.log | awk '{print $3}' | sort | uniq -c | sort -nr
```

### Log Rotation

Create `/etc/logrotate.d/squid`:

```
/var/log/squid/*.log {
    daily
    rotate 30
    compress
    delaycompress
    notifempty
    missingok
    sharedscripts
    postrotate
        /usr/sbin/squid -k rotate
    endscript
}
```

### SNMP Monitoring

```squid
# Enable SNMP
acl snmppublic snmp_community public
snmp_port 3401
snmp_access allow snmppublic localhost
snmp_access deny all
```

### Cache Manager (Web Interface)

```squid
# Allow cache manager from localhost
http_access allow localhost manager
http_access deny manager

# Set cache manager email
cache_mgr admin@example.com
```

Access via: `http://proxy-server:8080/squid-internal-mgr/`

---

## 10. Cache Configuration

### Basic Cache Setup

```squid
# Cache directory
# Format: cache_dir <type> <directory> <size_MB> <L1_dirs> <L2_dirs>
cache_dir ufs /var/spool/squid 10000 16 256

# Maximum object size in cache
maximum_object_size 4 MB

# Minimum object size in cache
minimum_object_size 0 KB

# Maximum object size in memory
maximum_object_size_in_memory 512 KB

# Memory cache size
cache_mem 256 MB
```

### Initialize Cache Directory

```bash
# Create and initialize cache directories
squid -z

# This must be run after changing cache_dir configuration
```

### Cache Replacement Policies

```squid
# LRU (Least Recently Used) - default
cache_replacement_policy lru

# Heap GDSF (Greedy-Dual Size Frequency)
cache_replacement_policy heap GDSF

# Heap LFUDA (Least Frequently Used with Dynamic Aging)
cache_replacement_policy heap LFUDA
```

### Refresh Patterns

```squid
# Refresh patterns control caching behavior
# Format: refresh_pattern <regex> <min> <percent> <max> [options]

# Don't cache CGI scripts
refresh_pattern ^ftp: 1440 20% 10080
refresh_pattern ^gopher: 1440 0% 1440

# Cache images for 1 week
refresh_pattern -i \.(jpg|jpeg|png|gif|bmp)$ 10080 90% 43200 override-expire ignore-no-cache

# Cache static content
refresh_pattern -i \.(css|js)$ 1440 50% 2880 override-expire

# Default pattern
refresh_pattern . 0 20% 4320
```

### Cache Control

```squid
# Don't cache certain sites
acl no_cache_sites dstdomain .facebook.com .twitter.com
cache deny no_cache_sites

# Don't cache dynamic content
acl dynamic_content urlpath_regex cgi-bin \?
cache deny dynamic_content

# Cache everything else
cache allow all
```

### Memory Cache Tuning

```squid
# Total memory for caching
cache_mem 512 MB

# Maximum single object in memory
maximum_object_size_in_memory 512 KB

# Memory pools
memory_pools on
memory_pools_limit 64 MB
```

---

## 11. Performance Tuning

### Connection Limits

```squid
# Maximum number of client connections
http_access allow all
request_header_max_size 64 KB
request_body_max_size 5 MB

# Client connection limits
client_lifetime 1 day
pconn_timeout 1 minute
```

### DNS Performance

```squid
# DNS nameserver configuration
dns_nameservers 8.8.8.8 8.8.4.4

# DNS timeout
dns_timeout 2 minutes

# Number of DNS lookup children
dns_children 32
```

### File Descriptor Limits

```squid
# Maximum file descriptors
max_filedescriptors 4096

# Increase system limits if needed
```

**Edit `/etc/security/limits.conf`:**
```
squid soft nofile 4096
squid hard nofile 8192
```

### Worker Processes (SMP)

```squid
# Enable multiple worker processes
workers 4

# CPU affinity (optional)
cpu_affinity_map process_numbers=1,2,3,4 cores=0,1,2,3
```

### Connection Pooling

```squid
# Client-side connections
client_persistent_connections on
server_persistent_connections on

# Keep-alive timeout
pconn_timeout 60 seconds

# Maximum persistent connections
client_pconn_lifetime 300 seconds
server_pconn_lifetime 300 seconds
```

### Optimization Example

```squid
# High-performance configuration
workers 4
cache_mem 2048 MB
maximum_object_size_in_memory 1024 KB
maximum_object_size 512 MB
cache_dir ufs /var/spool/squid 50000 16 256

# Connection optimization
client_persistent_connections on
server_persistent_connections on
pconn_timeout 120 seconds

# DNS optimization
dns_nameservers 8.8.8.8 8.8.4.4
dns_children 64

# File descriptors
max_filedescriptors 8192
```

---

## 12. Security Best Practices

### Restrict Access by IP/Network

```squid
# Define allowed networks
acl internal_network src 192.168.1.0/24
acl vpn_network src 10.8.0.0/24

# Allow only from these networks
http_access allow internal_network
http_access allow vpn_network
http_access deny all
```

### Disable Unsafe Methods

```squid
# Define unsafe methods
acl unsafe_methods method CONNECT TRACE DELETE

# Block unsafe methods (except CONNECT for HTTPS)
acl SSL_ports port 443
http_access deny unsafe_methods !SSL_ports
```

### Hide Client IP Address

```squid
# Don't reveal client IP to destination servers
forwarded_for off
request_header_access X-Forwarded-For deny all
request_header_access Via deny all
request_header_access Cache-Control deny all
```

### Prevent Information Disclosure

```squid
# Hide Squid version
httpd_suppress_version_string on

# Custom error pages
error_directory /etc/squid/errors/en

# Disable unnecessary information
request_header_access Server deny all
reply_header_access X-Cache deny all
reply_header_access X-Cache-Lookup deny all
```

### Rate Limiting

```squid
# Delay pools for rate limiting
delay_pools 1
delay_class 1 2

# 256 Kbps overall, 64 Kbps per IP
delay_parameters 1 32000/32000 8000/8000

# Apply to all
delay_access 1 allow all
```

### Malware Protection

```squid
# Block known malware domains (use external list)
acl malware_domains dstdomain "/etc/squid/malware_domains.txt"
http_access deny malware_domains

# Block malicious file types
acl dangerous_files urlpath_regex -i \.(exe|bat|com|scr|pif|vbs)$
http_access deny dangerous_files
```

### Security Headers

```squid
# Add security headers to responses
request_header_access X-Frame-Options deny all
request_header_access X-Content-Type-Options deny all
request_header_access X-XSS-Protection deny all

reply_header_access X-Frame-Options deny all
reply_header_access X-Content-Type-Options deny all
reply_header_access X-XSS-Protection deny all
```

---

## 13. Transparent Proxy Setup

Transparent proxy requires no browser configuration - traffic is automatically redirected to Squid.

### Squid Configuration

```squid
# Transparent proxy mode
http_port 3128 intercept

# Allow intercepted traffic
acl localnet src 192.168.1.0/24
http_access allow localnet
```

### IPTables Rules (Redirect Traffic)

```bash
# Enable IP forwarding
echo 1 > /proc/sys/net/ipv4/ip_forward

# Make permanent
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p

# Redirect HTTP traffic to Squid
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 -j REDIRECT --to-port 3128

# Redirect HTTPS traffic (if SSL bumping is configured)
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 443 -j REDIRECT --to-port 3129

# Save rules
iptables-save > /etc/sysconfig/iptables
```

### Firewalld Configuration (Alternative)

```bash
# Add rich rules for transparent proxy
firewall-cmd --permanent --direct --add-rule ipv4 nat PREROUTING 0 -i eth0 -p tcp --dport 80 -j REDIRECT --to-ports 3128
firewall-cmd --permanent --direct --add-rule ipv4 nat PREROUTING 0 -i eth0 -p tcp --dport 443 -j REDIRECT --to-ports 3129

# Reload firewall
firewall-cmd --reload
```

### Router/Gateway Configuration

If Squid is on a separate machine from the gateway:

```bash
# On the gateway/router
iptables -t nat -A PREROUTING -s 192.168.1.0/24 -p tcp --dport 80 -j DNAT --to-destination 192.168.1.10:3128
iptables -t nat -A PREROUTING -s 192.168.1.0/24 -p tcp --dport 443 -j DNAT --to-destination 192.168.1.10:3129
iptables -t nat -A POSTROUTING -j MASQUERADE
```

---

## 14. Common Commands & Troubleshooting

### Essential Commands

| Command | Description |
|---------|-------------|
| `systemctl start squid` | Start Squid service |
| `systemctl stop squid` | Stop Squid service |
| `systemctl restart squid` | Restart Squid service |
| `systemctl reload squid` | Reload configuration (no downtime) |
| `systemctl status squid` | Check service status |
| `squid -k parse` | Test configuration syntax |
| `squid -k check` | Verify configuration |
| `squid -k reconfigure` | Reload configuration |
| `squid -k rotate` | Rotate log files |
| `squid -z` | Initialize cache directories |
| `squid -v` | Show version |

### Configuration Testing

```bash
# Test configuration file
squid -k parse

# Verify configuration
squid -k check

# Test and show errors
squid -k parse -f /etc/squid/squid.conf
```

### View Squid Status

```bash
# Service status
systemctl status squid

# Check if Squid is running
ps aux | grep squid

# Check listening ports
netstat -tulpn | grep squid
ss -tulpn | grep squid

# Check cache directory
du -sh /var/spool/squid
```

### Monitoring Active Connections

```bash
# Show current connections
squidclient -h localhost -p 8080 mgr:active_requests

# Show cache statistics
squidclient mgr:info

# Show connection statistics
squidclient mgr:client_list
```

### Log Analysis

```bash
# View real-time access log
tail -f /var/log/squid/access.log

# Count requests per IP
awk '{print $3}' /var/log/squid/access.log | sort | uniq -c | sort -nr | head -20

# Most accessed domains
awk '{print $7}' /var/log/squid/access.log | sort | uniq -c | sort -nr | head -20

# Find denied requests
grep "TCP_DENIED" /var/log/squid/access.log

# Bandwidth usage by IP
awk '{ip[$3]+=$5} END {for (i in ip) print ip[i]/1024/1024, i}' /var/log/squid/access.log | sort -nr
```

### Common Issues & Solutions

#### Issue: Squid won't start

```bash
# Check configuration
squid -k parse

# Check logs for errors
tail -50 /var/log/squid/cache.log

# Check file permissions
ls -la /var/log/squid/
ls -la /var/spool/squid/

# Common causes:
# - Syntax errors in squid.conf
# - Permission issues on cache directory
# - Port already in use
```

#### Issue: Cache directory errors

```bash
# Reinitialize cache directories
rm -rf /var/spool/squid/*
squid -z

# Set correct permissions
chown -R squid:squid /var/spool/squid/
chmod 750 /var/spool/squid/
```

#### Issue: Port already in use

```bash
# Find process using port
lsof -i :8080
netstat -tulpn | grep :8080

# Kill process or change Squid port
kill -9 <PID>
```

#### Issue: Authentication not working

```bash
# Test password file
/usr/lib64/squid/basic_ncsa_auth /etc/squid/passwd
# Enter: username password
# Should return OK or ERR

# Check permissions
ls -la /etc/squid/passwd
chmod 640 /etc/squid/passwd
chown squid:squid /etc/squid/passwd

# Check Squid logs
tail -f /var/log/squid/cache.log | grep auth
```

#### Issue: SSL/HTTPS not working

```bash
# Verify certificate files exist
ls -la /etc/squid/ssl_cert/

# Check certificate validity
openssl x509 -in /etc/squid/ssl_cert/squid.pem -text -noout

# Verify permissions
chmod 600 /etc/squid/ssl_cert/*
chown squid:squid /etc/squid/ssl_cert/*

# For SSL bump, check certificate database
ls -la /var/lib/squid/ssl_db/
```

#### Issue: Website not blocked

```bash
# Test ACL matching
grep "facebook" /var/log/squid/access.log

# Verify ACL is before any "allow all" rule
cat /etc/squid/squid.conf | grep -A 5 "blocked_sites"

# Check if bypass is possible (direct IP access)
# Clients may bypass proxy if not configured correctly
```

#### Issue: Poor performance

```bash
# Check cache hit ratio
squidclient mgr:info | grep "Hit Ratios"

# Monitor memory usage
free -h
top -p $(pidof squid)

# Check disk I/O
iostat -x 2

# Consider:
# - Increase cache_mem
# - Use SSD for cache directory
# - Enable more worker processes
# - Tune DNS settings
```

### Client Configuration

#### Windows Proxy Settings

```
1. Settings → Network & Internet → Proxy
2. Enable "Use a proxy server"
3. Address: 192.168.1.10 (Squid server IP)
4. Port: 8080
```

#### Linux Proxy Settings

**Environment variables:**
```bash
export http_proxy=http://192.168.1.10:8080
export https_proxy=http://192.168.1.10:8080
export ftp_proxy=http://192.168.1.10:8080
export no_proxy=localhost,127.0.0.1
```

**System-wide (/etc/environment):**
```
http_proxy=http://192.168.1.10:8080
https_proxy=http://192.168.1.10:8080
```

#### Firefox Proxy Settings

```
1. Settings → Network Settings
2. Manual proxy configuration
3. HTTP Proxy: 192.168.1.10 Port: 8080
4. Check "Use this proxy server for all protocols"
```

---

## Quick Reference Card

### Configuration Workflow

1. Edit configuration: `vi /etc/squid/squid.conf`
2. Test syntax: `squid -k parse`
3. Reload Squid: `systemctl reload squid`
4. Check logs: `tail -f /var/log/squid/access.log`
5. Monitor: `squidclient mgr:info`

### Key Configuration Sections

- **http_port** - Listening port and options
- **acl** - Access control list definitions
- **http_access** - Access control rules
- **cache_dir** - Cache storage location and size
- **auth_param** - Authentication configuration

### Common ACL Types

- `src` - Source IP/network
- `dst` - Destination IP/network
- `dstdomain` - Destination domain
- `port` - Port number
- `method` - HTTP method
- `time` - Time of day
- `url_regex` - URL pattern
- `proxy_auth` - Authenticated user

### Log File Locations

- `/var/log/squid/access.log` - HTTP requests
- `/var/log/squid/cache.log` - Errors and cache operations
- `/var/spool/squid/` - Cache storage

---

## Production Configuration Example

**Complete production-ready configuration:**

```squid
#=========================================================================
# SQUID PROXY - PRODUCTION CONFIGURATION
# Author: Sajal Jana
# Organization: Example Corp
#=========================================================================

#-------------------------------------------------------------------------
# Network Ports
#-------------------------------------------------------------------------
http_port 8080
# https_port 8443 cert=/etc/squid/ssl_cert/squid.pem key=/etc/squid/ssl_cert/squid.key

#-------------------------------------------------------------------------
# Memory and Performance
#-------------------------------------------------------------------------
cache_mem 1024 MB
maximum_object_size_in_memory 512 KB
maximum_object_size 512 MB
minimum_object_size 0 KB

#-------------------------------------------------------------------------
# Cache Storage
#-------------------------------------------------------------------------
cache_dir ufs /var/spool/squid 20000 16 256
coredump_dir /var/spool/squid

#-------------------------------------------------------------------------
# ACLs - Network and Port Definitions
#-------------------------------------------------------------------------
acl SSL_ports port 443
acl Safe_ports port 80 21 443 70 210 1025-65535 280 488 591 777 8443
acl CONNECT method CONNECT
acl localnet src 192.168.1.0/24
acl localhost src 127.0.0.1/32

#-------------------------------------------------------------------------
# ACLs - Time-Based
#-------------------------------------------------------------------------
acl business_hours time MTWHF 08:00-18:00
acl weekend time SA

#-------------------------------------------------------------------------
# ACLs - Content Categories
#-------------------------------------------------------------------------
acl social_media dstdomain .facebook.com .twitter.com .instagram.com .tiktok.com
acl streaming dstdomain .youtube.com .netflix.com .hulu.com .twitch.tv
acl file_sharing dstdomain .dropbox.com .mega.nz .mediafire.com
acl allowed_domains dstdomain .google.com .bing.com .wikipedia.org .github.com .stackoverflow.com

#-------------------------------------------------------------------------
# Security Rules
#-------------------------------------------------------------------------
http_access deny !Safe_ports
http_access deny CONNECT !SSL_ports
http_access allow localhost manager
http_access deny manager

#-------------------------------------------------------------------------
# Content Filtering
#-------------------------------------------------------------------------
http_access deny social_media business_hours
http_access deny streaming business_hours
http_access deny file_sharing

#-------------------------------------------------------------------------
# Authentication (Optional)
#-------------------------------------------------------------------------
# auth_param basic program /usr/lib64/squid/basic_ncsa_auth /etc/squid/passwd
# auth_param basic children 5
# auth_param basic realm Squid Proxy Server
# acl authenticated_users proxy_auth REQUIRED
# http_access allow authenticated_users

#-------------------------------------------------------------------------
# Access Policy
#-------------------------------------------------------------------------
http_access allow localnet
http_access deny all

#-------------------------------------------------------------------------
# Logging
#-------------------------------------------------------------------------
access_log stdio:/var/log/squid/access.log squid
cache_log /var/log/squid/cache.log

logformat combined %>a %[ui %[un [%tl] "%rm %ru HTTP/%rv" %>Hs %<st "%{Referer}>h" "%{User-Agent}>h" %Ss:%Sh

#-------------------------------------------------------------------------
# Cache Tuning
#-------------------------------------------------------------------------
refresh_pattern ^ftp: 1440 20% 10080
refresh_pattern ^gopher: 1440 0% 1440
refresh_pattern -i \.(jpg|jpeg|png|gif|bmp)$ 10080 90% 43200 override-expire
refresh_pattern -i \.(css|js)$ 1440 50% 2880 override-expire
refresh_pattern . 0 20% 4320

#-------------------------------------------------------------------------
# Performance Tuning
#-------------------------------------------------------------------------
dns_nameservers 8.8.8.8 8.8.4.4
dns_timeout 2 minutes
client_persistent_connections on
server_persistent_connections on
pconn_timeout 120 seconds

#-------------------------------------------------------------------------
# Security
#-------------------------------------------------------------------------
httpd_suppress_version_string on
forwarded_for delete
request_header_access Via deny all
reply_header_access X-Cache deny all
reply_header_access X-Cache-Lookup deny all

#=========================================================================
# END OF CONFIGURATION
#=========================================================================
```

---

## Additional Resources

- **Official Documentation:** http://www.squid-cache.org/Doc/
- **Configuration Reference:** http://www.squid-cache.org/Doc/config/
- **Wiki:** https://wiki.squid-cache.org/
- **Mailing Lists:** http://www.squid-cache.org/Support/mailing-lists.html
- **Community Forum:** https://www.squid-cache.org/Support/

---

**Document prepared by Sajal Jana**

*End of Document*
