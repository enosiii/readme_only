# **Step-by-Step: Install Splunk Enterprise from USB (No Internet)**

## **Phase 1: Copy Splunk from USB to Server**

### **Step 1: Insert USB and Mount It**
```bash
# 1. Insert USB drive
# 2. Find USB device name
sudo fdisk -l
# Look for something like: /dev/sdb1

# 3. Create mount point and mount USB
sudo mkdir /mnt/usb
sudo mount /dev/sdb1 /mnt/usb  # Replace sdb1 with your actual USB

# 4. Verify files are there
ls -la /mnt/usb/
# You should see: splunk.tgz and maybe other files
```

### **Step 2: Copy Splunk to Server**
```bash
# Copy Splunk archive to server
sudo cp /mnt/usb/splunk.tgz /tmp/

# Optional: Copy Splunk apps too if you have them
sudo cp /mnt/usb/*.spl /tmp/ 2>/dev/null || true

# Unmount USB when done
sudo umount /mnt/usb
```

## **Phase 2: Install Splunk Enterprise**

### **Step 1: Install Dependencies (Offline Method)**
```bash
# First, check if dependencies are already installed
sudo apt update  # This will fail without internet, but try anyway

# If you have apt-offline setup, use it. Otherwise, we need to install manually:
# Most Ubuntu servers already have these, but just in case:

# Create a script to check and install dependencies from local repo if available
cat > check-deps.sh << 'EOF'
#!/bin/bash
# Check for required packages
REQUIRED_PACKAGES="libgssapi-krb5-2 libssl3"

for pkg in $REQUIRED_PACKAGES; do
    if ! dpkg -l | grep -q "^ii.*$pkg"; then
        echo "WARNING: $pkg not installed. Splunk may work anyway."
    fi
done
EOF

chmod +x check-deps.sh
sudo ./check-deps.sh
```

### **Step 2: Extract and Install Splunk**
```bash
# 1. Extract Splunk to /opt
sudo tar -xzf /tmp/splunk.tgz -C /opt/

# 2. Create splunk user
sudo useradd -m -d /opt/splunk -s /bin/bash splunk

# 3. Set ownership
sudo chown -R splunk:splunk /opt/splunk

# 4. Verify extraction
ls -la /opt/splunk/
# Should see: bin/, etc/, var/, etc.
```

### **Step 3: Initial Splunk Configuration**
```bash
# 1. Start Splunk for the first time (accepts license)
sudo -u splunk_fpi /opt/splunk/bin/splunk start --accept-license --answer-yes

# 2. Set admin password immediately (IMPORTANT!)
# Create password file
echo -e "[user_info]\nUSERNAME=admin\nPASSWORD=Innolab@2026" | sudo tee /opt/splunk/etc/system/local/user-seed.conf

# 3. Enable Splunk to start at boot
sudo /opt/splunk/bin/splunk enable boot-start -user splunk

# 4. Restart Splunk to apply password
sudo systemctl stop splunk
sudo systemctl start splunk
```

## **Phase 3: Configure Splunk for Log Collection**

### **Step 1: Enable Receiving Ports**
```bash
# 1. Enable listening on port 9997 (for syslog-ng)
sudo -u splunk /opt/splunk/bin/splunk enable listen 9997 -auth admin:Innolab@2026

# 2. Enable HTTP Event Collector (for future use)
sudo -u splunk /opt/splunk/bin/splunk enable http-event-collector -port 8088 -enable-ssl 0 -auth admin:Innolab@2026

# 3. Verify ports are open
sudo netstat -tulpn | grep -E '(9997|8088|8000)'
# Should see:
# 0.0.0.0:9997    - Splunk receiving port
# 0.0.0.0:8088    - HEC port
# 0.0.0.0:8000    - Web UI
```

### **Step 2: Create Network Logs Index**
```bash
# Create index for network device logs
sudo -u splunk /opt/splunk/bin/splunk add index network_logs -auth admin:Innolab@2026

# Set index properties
sudo tee /opt/splunk/etc/system/local/indexes.conf << 'EOF'
[network_logs]
homePath = $SPLUNK_DB/network_logs/db
coldPath = $SPLUNK_DB/network_logs/colddb
thawedPath = $SPLUNK_DB/network_logs/thaweddb
maxTotalDataSizeMB = 1000000  # 1TB limit
frozenTimePeriodInSecs = 7776000  # 90 days retention
EOF
```

### **Step 3: Configure Data Inputs**
```bash
# Configure the receiving port 9997 to use our network_logs index
sudo tee /opt/splunk/etc/system/local/inputs.conf << 'EOF'
[udp://9997]
sourcetype = network_device
index = network_logs
disabled = 0

[tcp://9997]
sourcetype = network_device
index = network_logs
disabled = 0

[monitor:///var/log/network-devices/*/*.log]
sourcetype = network_device
index = network_logs
disabled = 0
EOF

# Restart Splunk to apply changes
sudo systemctl restart splunk
```

## **Phase 4: Install Splunk Apps (from USB)**

### **Step 1: Install Cisco Network Add-on**
```bash
# Check if you have the .spl files on USB
ls /mnt/usb/*.spl 2>/dev/null || echo "No .spl files found on USB"

# If you have them, copy from USB
sudo cp /mnt/usb/splunk-addon-for-cisco-networks_*.spl /tmp/
sudo cp /mnt/usb/splunk-addon-for-palo-alto-networks_*.spl /tmp/ 2>/dev/null || true
sudo cp /mnt/usb/splunk-addon-for-f5-networks_*.spl /tmp/ 2>/dev/null || true

# Install Cisco add-on
sudo -u splunk /opt/splunk/bin/splunk install app /tmp/splunk-addon-for-cisco-networks_*.spl -auth admin:Innolab@2026

# For other add-ons if available:
# sudo -u splunk /opt/splunk/bin/splunk install app /tmp/splunk-addon-for-palo-alto-networks_*.spl -auth admin:Innolab@2026
# sudo -u splunk /opt/splunk/bin/splunk install app /tmp/splunk-addon-for-f5-networks_*.spl -auth admin:Innolab@2026
```

### **Step 2: Verify Installation**
```bash
# Restart Splunk
sudo systemctl restart splunk

# Check if apps are installed
sudo -u splunk /opt/splunk/bin/splunk display app -auth admin:Innolab@2026
```

## **Phase 5: Test & Verify Installation**

### **Step 1: Check Splunk Status**
```bash
# Check if Splunk is running
sudo systemctl status splunk

# Check Splunk web interface
curl -k https://localhost:8000 2>/dev/null | head -5

# If curl not installed, use wget or just test in browser
wget --no-check-certificate -O- https://localhost:8000 2>/dev/null | head -5
```

### **Step 2: Create Test Log Entry**
```bash
# Test if Splunk is receiving logs on port 9997
echo "<14>$(date) - Test message from installer" | nc -u localhost 9997

# Wait a few seconds and check
sudo -u splunk /opt/splunk/bin/splunk search '"Test message"' -auth admin:Innolab@2026
```

### **Step 3: Access Splunk Web UI**
```
Open web browser on your workstation:
https://192.168.1.200:8000

Username: admin
Password: Innolab@2026

First things to do in web UI:
1. Check "Search & Reporting"
2. Search: index="network_logs" | head 10
3. Check Apps → Manage Apps
4. Create a simple dashboard
```

## **Phase 6: Post-Installation Configuration**

### **Step 1: Optimize Splunk for Performance**
```bash
# Edit system limits for Splunk
sudo tee /opt/splunk/etc/system/local/limits.conf << 'EOF'
[thruput]
maxKBps = 0

[search]
max_searches_per_cpu = 2

[indexing]
maxHotBuckets = 10
maxConcurrentOptimizes = 6

[license]
active_group = Free
EOF
```

### **Step 2: Configure Storage**
```bash
# Check disk space
df -h /opt/splunk

# If you have separate storage, symlink the indexes directory
# Example for separate /data partition:
# sudo mkdir -p /data/splunk_indexes
# sudo chown splunk:splunk /data/splunk_indexes
# sudo -u splunk ln -sf /data/splunk_indexes /opt/splunk/var/lib/splunk
```

### **Step 3: Create Backup Script**
```bash
# Create backup script for Splunk configuration
sudo tee /opt/splunk/scripts/backup-splunk.sh << 'EOF'
#!/bin/bash
BACKUP_DIR="/opt/splunk/backups"
DATE=$(date +%Y%m%d_%H%M%S)

# Create backup directory
mkdir -p $BACKUP_DIR

# Backup configuration
tar -czf $BACKUP_DIR/splunk-config-$DATE.tar.gz \
    /opt/splunk/etc \
    /opt/splunk/var/lib/splunk/apps \
    2>/dev/null

# Keep only last 7 backups
find $BACKUP_DIR -name "*.tar.gz" -mtime +7 -delete

echo "Backup completed: $DATE"
EOF

sudo chmod +x /opt/splunk/scripts/backup-splunk.sh
sudo chown splunk:splunk /opt/splunk/scripts/backup-splunk.sh

# Add to crontab for splunk user
sudo -u splunk crontab -l 2>/dev/null | { cat; echo "0 2 * * * /opt/splunk/scripts/backup-splunk.sh"; } | sudo -u splunk crontab -
```

## **Quick Installation Script**

**Save this as `install-splunk.sh` and run it:**
```bash
#!/bin/bash
# install-splunk.sh - Complete Splunk installation from USB

set -e  # Exit on error

echo "=== Installing Splunk Enterprise from USB ==="

# Step 1: Mount USB
echo "1. Mounting USB..."
USB_DEVICE=$(sudo fdisk -l 2>/dev/null | grep -o "/dev/sd[b-z][0-9]" | head -1)
if [ -z "$USB_DEVICE" ]; then
    echo "ERROR: USB not found. Insert USB and try again."
    exit 1
fi

sudo mkdir -p /mnt/usb
sudo mount $USB_DEVICE /mnt/usb 2>/dev/null || true

# Step 2: Check for splunk.tgz
if [ ! -f "/mnt/usb/splunk.tgz" ]; then
    echo "ERROR: splunk.tgz not found on USB"
    echo "Files on USB:"
    ls -la /mnt/usb/
    exit 1
fi

echo "✅ Found splunk.tgz on USB"

# Step 3: Copy files
echo "2. Copying files..."
sudo cp /mnt/usb/splunk.tgz /tmp/

# Step 4: Extract
echo "3. Extracting Splunk..."
sudo tar -xzf /tmp/splunk.tgz -C /opt/

# Step 5: Create user
echo "4. Creating splunk user..."
sudo useradd -m -d /opt/splunk -s /bin/bash splunk 2>/dev/null || true
sudo chown -R splunk:splunk /opt/splunk

# Step 6: Start Splunk
echo "5. Starting Splunk..."
sudo -u splunk /opt/splunk/bin/splunk start --accept-license --answer-yes

# Step 7: Set password
echo "6. Setting admin password..."
echo -e "[user_info]\nUSERNAME=admin\nPASSWORD=Innolab@2026" | sudo tee /opt/splunk/etc/system/local/user-seed.conf

# Step 8: Enable boot start
echo "7. Enabling boot start..."
sudo /opt/splunk/bin/splunk enable boot-start -user splunk

# Step 9: Restart
echo "8. Restarting Splunk..."
sudo systemctl restart splunk

# Step 10: Configure
echo "9. Configuring Splunk..."
sudo -u splunk /opt/splunk/bin/splunk enable listen 9997 -auth admin:Innolab@2026
sudo -u splunk /opt/splunk/bin/splunk add index network_logs -auth admin:Innolab@2026

# Step 11: Create configs
sudo tee /opt/splunk/etc/system/local/inputs.conf << 'EOF'
[udp://9997]
sourcetype = network_device
index = network_logs
EOF

# Step 12: Final restart
sudo systemctl restart splunk

echo ""
echo "=== INSTALLATION COMPLETE ==="
echo ""
echo "✅ Splunk Enterprise installed successfully!"
echo ""
echo "Access Splunk:"
echo "  URL:      https://$(hostname -I | awk '{print $1}'):8000"
echo "  Username: admin"
echo "  Password: Innolab@2026"
echo ""
echo "Test with:"
echo "  echo '<14>Test' | nc -u localhost 9997"
echo "  Then search in Splunk: index=\"network_logs\" | head 10"
echo ""
```

## **Troubleshooting Common Issues**

### **Issue 1: "Permission denied" when starting Splunk**
```bash
sudo chown -R splunk:splunk /opt/splunk
sudo systemctl restart splunk
```

### **Issue 2: Port 8000 not accessible**
```bash
# Check firewall
sudo ufw allow 8000/tcp
sudo ufw allow 9997/tcp
sudo ufw allow 9997/udp

# Check if Splunk is listening
sudo netstat -tulpn | grep splunk
```

### **Issue 3: Splunk starts but web UI doesn't load**
```bash
# Check Splunk logs
sudo tail -f /opt/splunk/var/log/splunk/splunkd.log

# Common fix: Increase memory in limits.conf
sudo tee -a /opt/splunk/etc/system/local/limits.conf << 'EOF'
[search]
max_mem_usage_mb = 1024
EOF
sudo systemctl restart splunk
```

### **Issue 4: Can't find USB device**
```bash
# List all storage devices
sudo lsblk

# Try mounting different devices
sudo mount /dev/sdc1 /mnt/usb  # Try sdc1, sdd1, etc.
```

## **Verification Checklist**
```bash
# Run this after installation
echo "=== Splunk Verification ==="
echo "1. Service status:"
sudo systemctl status splunk | grep "Active:"

echo "2. Listening ports:"
sudo netstat -tulpn | grep -E "(8000|9997|8088)" | grep LISTEN

echo "3. Disk usage:"
du -sh /opt/splunk/

echo "4. Test log ingestion:"
echo "<14>Verification test $(date)" | nc -u localhost 9997
sleep 2
sudo -u splunk /opt/splunk/bin/splunk search 'Verification test' -auth admin:Innolab@2026 2>/dev/null | head -5

echo "✅ Verification complete!"
```

**Now you have Splunk Enterprise running!** Next step is to configure syslog-ng on your collector VM and start sending logs.
