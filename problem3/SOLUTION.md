# NGINX VM Storage Troubleshooting

### 1. Verify Current Storage Usage
```bash
# Check overall disk usage
df -h

# Check inode usage (sometimes the issue is inode exhaustion)
df -i

# Identify largest directories
du -h --max-depth=1 / 2>/dev/null | sort -hr | head -20

# Check specific NGINX-related directories
du -sh /var/log/nginx/
du -sh /var/lib/nginx/
du -sh /tmp/
du -sh /var/tmp/
```

### 2. Identify Large Files and Directories
```bash
# Find largest files system-wide
find / -type f -size +100M -exec ls -lh {} \; 2>/dev/null | sort -k5 -hr

# Find largest directories
du -ahx / 2>/dev/null | sort -rh | head -30

# Check for deleted files still held by processes
lsof +L1 | grep deleted
```
---

## Possible Root Cause

### Root Cause #1: NGINX Log Files Accumulation

#### **Troubleshooting Commands**
```bash
# Check log file sizes
ls -lh /var/log/nginx/

# Check log rotation status
cat /etc/logrotate.d/nginx
systemctl status logrotate

# Check when logs were last rotated
ls -lt /var/log/nginx/

# Check current log file growth
tail -f /var/log/nginx/access.log
```

#### **Expected Scenario**
- Access logs growing rapidly due to high traffic volume
- Error logs accumulating from upstream server failures
- Log rotation not configured or failing
- Compressed old logs not being deleted

#### **Impact**
- When logs fill disk to 100%, NGINX cannot write new logs
- Disk I/O slowdown affects request processing
- Loss of audit trail and troubleshooting data
- NGINX may become unresponsive or crash

#### **Recovery Steps**
```bash
# IMMEDIATE: Truncate current logs (preserve recent entries)
tail -n 10000 /var/log/nginx/access.log > /tmp/access.log.recent
mv /tmp/access.log.recent /var/log/nginx/access.log

tail -n 10000 /var/log/nginx/error.log > /tmp/error.log.recent
mv /tmp/error.log.recent /var/log/nginx/error.log

# Or completely clear
truncate -s 0 /var/log/nginx/access.log
truncate -s 0 /var/log/nginx/error.log

# Signal NGINX to reopen log files
nginx -s reopen

# Delete old compressed logs
find /var/log/nginx/ -name "*.gz" -mtime +7 -delete

# Configure proper log rotation
cat > /etc/logrotate.d/nginx << 'EOF'
/var/log/nginx/*.log {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    create 0640 www-data adm
    sharedscripts
    postrotate
        [ -f /var/run/nginx.pid ] && kill -USR1 `cat /var/run/nginx.pid`
    endscript
}
EOF

# Force log rotation
logrotate -f /etc/logrotate.d/nginx
```

---

### Root Cause #2: Temporary Files Accumulation

#### **Troubleshooting Commands**
```bash
# Check temp directories
du -sh /tmp /var/tmp /var/lib/nginx/body /var/lib/nginx/proxy

# Find old temp files
find /tmp -type f -mtime +7
find /var/tmp -type f -mtime +7

# Check NGINX temp paths
grep -E "client_body_temp_path|proxy_temp_path" /etc/nginx/nginx.conf
```

#### **Expected Scenario**
- Large file uploads stored in client_body_temp_path
- Proxy buffering creating large temp files
- Failed uploads leaving orphaned files
- systemd-tmpfiles not cleaning /tmp properly

#### **Impact**
- Gradual accumulation over time
- Disk I/O affected if temp is on same partition
- Upload failures when disk full
- Old temp files may contain sensitive data

#### **Recovery Steps**
```bash
# IMMEDIATE: Clean old temp files
find /tmp -type f -mtime +2 -delete
find /var/tmp -type f -mtime +7 -delete

# Clean NGINX temp directories
systemctl stop nginx
rm -rf /var/lib/nginx/body/*
rm -rf /var/lib/nginx/proxy/*
systemctl start nginx

# Configure NGINX temp file size limits
cat >> /etc/nginx/nginx.conf << 'EOF'
http {
    client_body_temp_path /var/lib/nginx/body 1 2;
    client_max_body_size 10M;
    proxy_temp_path /var/lib/nginx/proxy;
}
EOF

# Configure systemd-tmpfiles cleanup
cat > /etc/tmpfiles.d/tmp.conf << 'EOF'
# Delete files in /tmp older than 3 days
d /tmp 1777 root root 3d
EOF

nginx -t && systemctl reload nginx
```

---

### Root Cause #3: Misconfigured NGINX Buffering/Caching

#### **Troubleshooting Commands**
```bash
# Check NGINX configuration
grep -r "proxy_cache_path\|fastcgi_cache_path\|uwsgi_cache_path" /etc/nginx/

# Check cache directories
find /var/cache/nginx -type f | wc -l
du -sh /var/cache/nginx/

# Check proxy buffering settings
grep -r "proxy_buffer" /etc/nginx/
```

#### **Expected Scenario**
- Proxy cache enabled without size limits
- Cache not being purged properly
- Large responses being cached indefinitely
- Cache keys not properly configured leading to unlimited growth

#### **Impact**
- Can fill disk completely
- Initially improves, then degrades when disk full
- Cache writes fail, may affect proxying
- Old cached data may be stale

#### **Recovery Steps**
```bash
# IMMEDIATE: Clear cache
systemctl stop nginx
rm -rf /var/cache/nginx/*
systemctl start nginx

# Configure cache with limits
cat > /etc/nginx/conf.d/cache.conf << 'EOF'
proxy_cache_path /var/cache/nginx levels=1:2 
                 keys_zone=my_cache:10m 
                 max_size=2g 
                 inactive=60m 
                 use_temp_path=off;

# In server/location blocks:
# proxy_cache my_cache;
# proxy_cache_valid 200 10m;
# proxy_cache_valid 404 1m;
EOF

# Test and reload
nginx -t && systemctl reload nginx

# Set up cache cleanup cron
cat > /etc/cron.daily/nginx-cache-clean << 'EOF'
#!/bin/bash
find /var/cache/nginx -type f -mtime +7 -delete
EOF
chmod +x /etc/cron.daily/nginx-cache-clean
```

---

## Prevention and Monitoring

### 1. Set Up Disk Space Monitoring
```bash
# Install monitoring tools
apt-get install -y prometheus-node-exporter

# Create disk space alert script
cat > /usr/local/bin/disk-alert.sh << 'EOF'
#!/bin/bash
THRESHOLD=80
USAGE=$(df / | tail -1 | awk '{print $5}' | sed 's/%//')
if [ $USAGE -gt $THRESHOLD ]; then
    echo "ALERT: Disk usage at ${USAGE}%" | logger -t disk-alert
    # Add notification mechanism (email, Slack, PagerDuty, etc.)
fi
EOF
chmod +x /usr/local/bin/disk-alert.sh

# Add to crontab
echo "*/5 * * * * /usr/local/bin/disk-alert.sh" | crontab -
```

### 2. Implement Log Management Best Practices
```bash
# Reduce NGINX log verbosity for non-critical paths
# Add to nginx.conf location blocks:
# access_log off;  # For health check endpoints
# error_log /var/log/nginx/error.log warn;  # Only warnings and above

# Use syslog for centralized logging
# error_log syslog:server=logserver:514,tag=nginx,severity=info;
```

### 3. Regular Maintenance Checklist
```bash
# Create weekly maintenance script
cat > /etc/cron.weekly/disk-maintenance << 'EOF'
#!/bin/bash
# Rotate logs
logrotate -f /etc/logrotate.d/nginx

# Clean package cache
apt-get autoclean

# Clean old journals
journalctl --vacuum-time=7d

# Remove old temp files
find /tmp -type f -mtime +7 -delete

# Check for deleted but open files
lsof +L1 > /var/log/orphaned-files.log

# Report disk usage
df -h > /var/log/disk-usage-report.log
EOF
chmod +x /etc/cron.weekly/disk-maintenance
```

### 4. Configuration Hardening
```nginx
# Add to /etc/nginx/nginx.conf
http {
    # Limit request body size
    client_max_body_size 10M;
    
    # Optimize buffer sizes
    client_body_buffer_size 128k;
    
    # Limit proxy buffering
    proxy_buffering on;
    proxy_buffer_size 4k;
    proxy_buffers 8 4k;
    
    # Set temp file write delay
    proxy_max_temp_file_size 1024m;
    
    # Log format optimization (exclude unnecessary fields)
    log_format optimized '$remote_addr - $status [$time_local] '
                        '"$request" $body_bytes_sent';
    access_log /var/log/nginx/access.log optimized;
}
```