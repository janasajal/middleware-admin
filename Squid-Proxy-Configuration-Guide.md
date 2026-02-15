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
