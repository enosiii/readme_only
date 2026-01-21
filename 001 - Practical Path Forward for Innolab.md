# **Revised Strategy: Practical Path Forward for Innolab**

**AI & Automation - focus on the big picture!** Let me provide a **realistic, step-by-step approach** considering your constraints.

## **🎯 Revised Understanding & Recommendations:**

### **Key Constraints Identified:**
1. **No internet in office** (only local network)
2. **Target: End of month** (Jan 31, 2026 - 10 days!)
3. **Ultimate goal: On-prem for clients**
4. **Testing in lab first**
5. **Future AI automation**

### **🚨 RECOMMENDATION: Start with Splunk On-Prem NOW**

**Why NOT Cloud for your case:**
- No internet = Cloud Splunk impossible
- Clients want on-prem anyway
- Faster to deploy locally
- Control all data in-house

## **📋 Revised 10-Day Action Plan:**

### **Phase 1: TODAY (Day 1-2) - Infrastructure Setup**

#### **Step 1: Get Server from Logistics**
```bash
# Request this SPECIFICALLY:
- Server: Dell PowerEdge R740 or similar
- CPU: 16+ cores (for Splunk + Go collector)
- RAM: 64GB minimum (Splunk is memory hungry)
- Storage: 2TB SSD (500GB for OS, 1.5TB for logs)
- Network: 10GbE NIC
- OS: Ubuntu 22.04 LTS pre-installed
```

#### **Step 2: Install VMware ESXi (for future client flexibility)**
```bash
# Why ESXi? Clients likely use VMware
# Free version available for lab

1. Download ESXi 8.0: https://customerconnect.vmware.com
2. Install on bare metal
3. Create VMs:
   - VM1: Ubuntu 22.04 (Go Collector) - 4CPU, 8GB RAM
   - VM2: Ubuntu 22.04 (Splunk) - 8CPU, 32GB RAM
   - VM3: Ubuntu 22.04 (Ansible Control) - 2CPU, 4GB RAM
```

#### **Step 3: Basic Network Configuration**
```bash
# Static IPs for each VM:
- Go Collector: 192.168.1.100
- Splunk Server: 192.168.1.200  
- Ansible Control: 192.168.1.50
- Gateway: 192.168.1.1
- DNS: 8.8.8.8 (won't work without internet, but set anyway)
```

### **Phase 2: Day 3-5 - Core Implementation**

#### **Step 1: Install Splunk Enterprise (ON-PREM)**
```bash
# On Splunk VM (192.168.1.200)
wget -O splunk.tar.gz "INTERNAL_PATH/splunk-9.1.1-Linux-x86_64.tgz"
tar -xzf splunk.tar.gz -C /opt
sudo /opt/splunk/bin/splunk start --accept-license --answer-yes
sudo /opt/splunk/bin/splunk enable boot-start

# Set admin password
echo -e "[user_info]\nUSERNAME=admin\nPASSWORD=Innolab@2026" > /opt/splunk/etc/system/local/user-seed.conf

# Enable HEC for Go collector
sudo /opt/splunk/bin/splunk enable http-event-collector -port 8088 -enable-ssl 0
```

#### **Step 2: Create SIMPLE Go Collector (MVP)**
```go
// File: /opt/collector/main.go
package main

import (
    "fmt"
    "log"
    "net"
    "strings"
    "time"
)

func main() {
    fmt.Println("🚀 Innolab Collector Starting...")
    
    // Simple syslog receiver
    udpAddr, _ := net.ResolveUDPAddr("udp", ":514")
    conn, _ := net.ListenUDP("udp", udpAddr)
    defer conn.Close()
    
    buffer := make([]byte, 65535)
    
    for {
        n, addr, _ := conn.ReadFromUDP(buffer)
        msg := strings.TrimSpace(string(buffer[:n]))
        
        // Log to console and file
        log.Printf("📨 %s: %s", addr.IP, msg)
        
        // Write to file
        writeToFile(msg, addr.IP.String())
        
        // TODO: Send to Splunk later
    }
}

func writeToFile(msg, ip string) {
    // Simple file logging
    filename := fmt.Sprintf("/opt/collector/logs/%s.log", time.Now().Format("2006-01-02"))
    // ... file writing logic
}
```

#### **Step 3: Configure ONE Cisco Device (Test First)**
```cisco
! On your test Cisco switch/router:
enable
conf t
logging host 192.168.1.100
logging trap informational
logging source-interface vlan 1
end
write memory
```

### **Phase 3: Day 6-8 - Integration & Monitoring**

#### **Step 1: Connect Go to Splunk**
```go
// Add to collector:
func sendToSplunk(msg, deviceIP string) {
    // Simple HTTP POST to Splunk HEC
    // http://192.168.1.200:8088/services/collector
}
```

#### **Step 2: Install Splunk Apps OFFLINE**
```bash
# Download these at home with internet, bring to office:
1. Splunk Add-on for Cisco Networks
2. Splunk Add-on for Palo Alto Networks  
3. Splunk Add-on for F5 Networks
4. Splunk App for Infrastructure

# Install manually:
sudo cp cisco-addon.spl /opt/splunk/etc/apps/
sudo chown -R splunk:splunk /opt/splunk/etc/apps/cisco-addon
sudo /opt/splunk/bin/splunk restart
```

#### **Step 3: Create Basic Dashboard**
```
1. Login to Splunk: https://192.168.1.200:8000
2. Search → New Dashboard
3. Add panels:
   - Device Status (up/down)
   - Log Volume
   - Error Count
   - Top Talkers
```

### **Phase 4: Day 9-10 - Documentation & Demo**

#### **Step 1: Create Runbook**
```markdown
# Innolab Log Collection Runbook

## 1. Daily Checks
- Splunk: https://192.168.1.200:8000
- Collector: systemctl status innolab-collector
- Logs: tail -f /opt/collector/logs/current.log

## 2. Adding New Devices
1. Add IP to /opt/collector/config/devices.txt
2. Configure device: logging host 192.168.1.100
3. Verify: tail logs | grep "device-ip"

## 3. Troubleshooting
- No logs? Check: sudo tcpdump -i any port 514
- Splunk down? sudo systemctl restart splunk
```

#### **Step 2: Prepare Demo for Boss**
```bash
# Create demo script
cat > demo.sh << 'EOF'
echo "=== Innolab Log Collection Demo ==="
echo "1. Showing live logs..."
tail -f /opt/collector/logs/$(date +%Y-%m-%d).log &

echo "2. Generating test log..."
echo "<14>Demo: System working $(date)" | nc -u 192.168.1.100 514

echo "3. Opening Splunk dashboard..."
# Show dashboard URL
echo "Dashboard: https://192.168.1.200:8000"
EOF
```

## **📊 Updated Timeline (Jan 21-31, 2026):**

| Day | Task | Status |
|-----|------|--------|
| **Jan 21** | Get server, install ESXi | TODAY |
| **Jan 22** | Create VMs, basic network | |
| **Jan 23** | Install Splunk Enterprise | |
| **Jan 24** | Build Go collector MVP | |
| **Jan 25** | Configure first Cisco device | |
| **Jan 26** | Connect collector to Splunk | |
| **Jan 27** | Install Splunk apps | |
| **Jan 28** | Configure all devices | |
| **Jan 29** | Create dashboards | |
| **Jan 30** | Testing & documentation | |
| **Jan 31** | Demo preparation | |

## **🎯 What to Show Your Boss Each Day:**

### **Daily Update Template:**
```
Date: Jan 21, 2026
Progress:
- ✅ Server received from logistics
- ✅ ESXi installed on bare metal
- ✅ 3 VMs created (Splunk, Collector, Ansible)
- 🔄 Network configuration in progress

Blockers:
- Need: Splunk Enterprise ISO file
- Need: Ubuntu 22.04 ISO

Next:
1. Install Splunk on VM2
2. Test basic connectivity

Demo Ready: Basic log collection working
```

## **🤔 Answering Your Questions:**

### **Q: On-Prem vs Cloud?**
**A: Go ON-PREM** because:
- No internet in office
- Clients want on-prem
- More control
- No licensing costs for lab

### **Q: Is On-Prem hard?**
**A: Actually EASIER for your case:**
1. Download once (at home with internet)
2. Install locally
3. No internet dependency
4. Faster troubleshooting

### **Q: What about Go & Ansible?**
**Priority order:**
1. **Go collector** (essential for log ingestion)
2. **Splunk** (dashboard/visualization)
3. **Ansible** (later for automation)
4. **Catalyst Center** (much later)

### **Q: Internet requirement?**
**Solution:**
- Download everything at HOME
- Bring USB to OFFICE
- Install offline:
  - Splunk Enterprise
  - Ubuntu ISO
  - Go binaries
  - Splunk apps

## **📁 Offline Installation Pack Preparation:**

### **What to Download at Home:**
```bash
# Create USB with these files:
/Innolab-Setup/
├── Operating-Systems/
│   ├── ubuntu-22.04.3-live-server-amd64.iso
│   └── VMware-ESXi-8.0.iso
├── Splunk/
│   ├── splunk-9.1.1-Linux-x86_64.tgz
│   ├── cisco-addon.tgz
│   ├── paloalto-addon.tgz
│   └── f5-addon.tgz
├── Go/
│   └── go1.21.5.linux-amd64.tar.gz
└── Scripts/
    ├── setup-splunk.sh
    ├── setup-collector.sh
    └── configure-devices.sh
```

## **🚀 Quick Start TODAY:**

### **Immediate Action (First 2 Hours):**
```bash
# 1. Get physical server from logistics
# 2. Install ESXi (ask IT for help if needed)
# 3. Create first VM (Ubuntu for collector)
# 4. Install Ubuntu, set static IP
# 5. Write basic Go syslog receiver
# 6. Test with: echo "test" | nc -u vm-ip 514

# That's Day 1 DONE!
```

## **💡 Pro Tips for Success:**

1. **Start SIMPLE** - Basic syslog collection first
2. **One device at a time** - Cisco switch first
3. **Manual then automate** - Get it working, then Ansible
4. **Document as you go** - Screenshots, configs
5. **Daily demo to boss** - Show progress

## **📞 When You Get Stuck:**

### **Common Issues & Fixes:**
```bash
# Can't download packages offline?
# Solution: Use apt-offline
apt-offline set packages.txt --install-packages golang
# Generate at home, install in office

# Splunk license expired?
# Solution: Free 60-day trial, extend via email

# Go compilation errors?
# Solution: Use pre-compiled binary
wget https://go.dev/dl/go1.21.5.linux-amd64.tar.gz

# No logs appearing?
# Solution: Check firewall
sudo ufw allow 514/udp
sudo tcpdump -i any port 514
```

## **🎯 Final Deliverable (Jan 31):**

```
✅ 1. Physical server with ESXi
✅ 2. 3 VMs running (Splunk, Collector, Ansible)
✅ 3. Go collector ingesting logs from ALL devices
✅ 4. Splunk dashboard showing real-time logs
✅ 5. Basic alerts configured
✅ 6. Runbook for operations
✅ 7. Demo script for boss
```

**Start with getting the server TODAY.** Everything else follows. Your boss wants to see **progress**, not perfection. Show working pieces each day!

**Need specific help with any step?** Tell me what you're stuck on and I'll give you exact commands!
