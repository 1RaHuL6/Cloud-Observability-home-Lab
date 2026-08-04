**AWS remote server
---

```markdown
# ️ AWS EC2 Setup: Flask App + Node Exporter

This guide walks through setting up the production-like application on AWS EC2.
---
## 🎯 Objectives

- ✅ Launch t3.micro Ubuntu instance
- ✅ Deploy Flask web application
- ✅ Install Node Exporter for system metrics
- ✅ Configure Security Group for least-privilege access
- ✅ Assign Elastic IP for static addressing

---

## 📋 Prerequisites

- AWS Free Tier account
- AWS CLI installed (optional, for advanced users)
- SSH key pair (.pem file)
- Basic familiarity with AWS Console

---

## 🚀 Step-by-Step Setup

### Step 1: Launch EC2 Instance

AWS Console → EC2 Dashboard → "Launch instance"
Configure:
• Name: aws-production-server
• AMI: Ubuntu Server 22.04 LTS
• Instance type: t2.micro (Free tier eligible)
• Key pair: Select your existing .pem key
• Security group: Create new (configure in Step 4)
• Storage: 8 GB gp2
Click: "Launch instance"
Wait 2-5 minutes for status: "Running"

### Step 2: Allocate Elastic IP

EC2 Dashboard → "Elastic IPs" (left sidebar)
Click: "Allocate Elastic IP address"
Configure:
• Amazon's pool of IPv4 addresses
• Click: Allocate
Select the new Elastic IP → "Actions" → "Associate"
Configure:
• Resource type: Instance
• Instance: aws-production-server
• Click: Associate
Note the Elastic IP (e.g., 54.123.45.67)
→ This IP never changes (unless you release it)

### Step 3: SSH Into Instance

```powershell
# From Windows PowerShell:
cd C:\Users\YourName\.ssh
ssh -i your-key.pem ubuntu@<elastic-ip>

# First connection warning:
# Type: yes


### Step 4: Configure Security Group
1. EC2 Dashboard → "Security Groups"

2. Select your instance's security group

3. Edit inbound rules:

| Type | Port | Source | Description |
|------|------|--------|-------------|
| SSH | 22 | My IP | SSH access |
| Custom TCP | 5000 | My IP | Flask app |
| Custom TCP | 9100 | My IP | Node Exporter |

4. Click: "Save rules"

### Step 5: Update System Packages

# Inside EC2 instance:
sudo apt update
sudo apt upgrade -y
sudo apt install -y python3-pip python3-venv git curl

### Step 6: Deploy Flask Application

# Create app directory
mkdir -p ~/myapp
cd ~/myapp

# Create Flask app
cat > app.py << 'EOF'
from flask import Flask, jsonify
import time
import os

app = Flask(__name__)

@app.route('/')
def hello():
    return jsonify({
        "message": "Hello from AWS Production Server!",
        "cloud": "AWS",
        "instance": os.uname().nodename,
        "timestamp": time.time()
    })

@app.route('/health')
def health():
    return jsonify({
        "status": "healthy",
        "cloud": "AWS"
    }), 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF

# Create virtual environment
python3 -m venv venv
source venv/bin/activate
pip install flask

# Test app (temporary)
python3 app.py &
curl http://localhost:5000/health
# Expected: {"status": "healthy", "cloud": "AWS"}
kill %1

### Step 7: Create Syetemd Service

# Create service file
sudo nano /etc/systemd/system/myapp.service
[Unit]
Description=My Flask Web Application
After=network.target

[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu/myapp
ExecStart=/home/ubuntu/myapp/venv/bin/python3 /home/ubuntu/myapp/app.py
Restart=always
Environment="PATH=/home/ubuntu/myapp/venv/bin"

[Install]
WantedBy=multi-user.target
Save and start:
sudo systemctl daemon-reload
sudo systemctl start myapp
sudo systemctl enable myapp
sudo systemctl status myapp
# Expected: Active: active (running)

### Step 8: Install Node Exporter

# Download
cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.0/node_exporter-1.6.0.linux-amd64.tar.gz

# Extract
tar xvfz node_exporter-1.6.0.linux-amd64.tar.gz

# Install
sudo mv node_exporter-1.6.0.linux-amd64/node_exporter /usr/local/bin/

# Create service
sudo nano /etc/systemd/system/node_exporter.service

Content:
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=ubuntu
ExecStart=/usr/local/bin/node_exporter
Restart=always

[Install]
WantedBy=multi-user.target

Save and start:
sudo systemctl daemon-reload
sudo systemctl start node_exporter
sudo systemctl enable node_exporter
sudo systemctl status node_exporter
# Expected: Active: active (running)

### Step 9: Verify Everything Works
# Test Flask app
curl http://localhost:5000/health
# Expected: {"status": "healthy", "cloud": "AWS"}

# Test Node Exporter
curl http://localhost:9100/metrics | head -n 10
# Expected: Prometheus metrics format

# From Windows (PowerShell):
curl http://<elastic-ip>:5000/health
curl http://<elastic-ip>:9100/metrics | Select-Object -First 10




