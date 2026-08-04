📊 Data Flow: How Metrics Travel Through the System 

This document explains how metrics flow from AWS EC2 to Grafana dashboard — step by step.

🔄 The PULL Model (Key Concept)

**Critical Understanding**: Prometheus uses a **PULL model**, not PUSH.

### Why This Matters:
- Node Exporter **does not initiate connections**
- Node Exporter **just waits** on port 9100 for HTTP requests
- Prometheus **actively scrapes** every 15 seconds
- This is more secure (no outbound connections from cloud needed)

---

## 🛣️ Complete Data Flow Path

### Step 1: Metrics Generation (AWS EC2)
Location: AWS EC2 Instance
Process: Node Exporter (port 9100)
What happens:
Node Exporter reads /proc filesystem (CPU, memory, disk stats)
Formats data as Prometheus metrics (plain text)
Waits for HTTP GET requests on /metrics endpoint
Responds with metrics when asked
Example metric output:
HELP node_cpu_seconds_total Seconds the CPUs spent in each mode.
TYPE node_cpu_seconds_total counter
node_cpu_seconds_total{cpu="0",mode="idle"} 12345.67 

### Step 2: SSH Tunnel (Secure Transport)
Location: Monitoring VM → AWS EC2
Process: autossh (persistent SSH tunnel)
What happens:
Monitoring VM establishes SSH connection to AWS EC2
Creates port forward: localhost:9101 → AWS:localhost:9100
Tunnel encrypts all traffic between VMs
Auto-reconnects if connection drops (autossh feature)
Command:
autossh -M 0 -f -N -L 9101:localhost:9100 ubuntu@<aws-elastic-ip>
Why tunnel instead of direct access?
✅ No need to expose Node Exporter publicly
✅ Encrypted traffic (SSH encryption)
✅ Works with restrictive Security Groups
✅ Authentication via SSH keys

### Step 3: Prometheus Scraping (Collection)
Location: Monitoring VM
Process: Prometheus (port 9090)
What happens:
Every 15 seconds, Prometheus reads prometheus.yml
For each target, makes HTTP GET request to /metrics
For aws-ec2 job: requests http://localhost:9101/metrics
Request travels through SSH tunnel to AWS Node Exporter
Prometheus receives metrics, parses them
Stores in time-series database with timestamp
Configuration (prometheus.yml):
job_name: 'aws-ec2'
static_configs:
targets: ['localhost:9101'] # Via SSH tunnel

### Step 4: Grafana Query (Visualization)
Location: Monitoring VM
Process: Grafana (port 3000)
What happens:
User opens Grafana dashboard in browser
Dashboard panels contain PromQL queries
Grafana sends queries to Prometheus API
Prometheus returns time-series data
Grafana renders as charts/graphs
Example PromQL query:
rate(node_cpu_seconds_total{mode="idle"}[5m])
Result: CPU usage percentage over time

### Step 5: Browser Display (Human Observation)
Location: Windows Host Browser
Process: Chrome
What happens:
User navigates to http://localhost:3000
VirtualBox port forward: 3000 (host) → 3000 (guest)
Grafana web UI loads in browser
User sees real-time metrics from AWS EC2
Can set time range, refresh interval, etc.
Port Forwarding Chain:
Windows Browser:3000
→ VirtualBox Port Forward
→ Grafana VM:3000
→ Prometheus Query
→ SSH Tunnel
→ AWS Node Exporter:9100


---

## 🔍 Key Ports in the Flow

| Port    | Service       | Direction     | Purpose                       |
|------   |---------      |-----------    |---------                      |
| 9100    | Node Exporter | AWS → Tunnel  | Exposes system metrics        |
| 9101    | SSH Tunnel    | Monitoring VM | Local endpoint for Prometheus |
| 9090    | Prometheus    | Monitoring VM | Metrics collection + API      |
| 3000    | Grafana       | Monitoring VM | Dashboard visualization       |
| 2222→22 | SSH           | Windows → VM  | Management access             |

---

## 🧪 Verification Commands

### Verify Each Step:

```bash
# Step 1: Node Exporter responding on AWS
ssh -i aws-key.pem ubuntu@<aws-ip>
curl http://localhost:9100/metrics | head -n 10

# Step 2: SSH tunnel active
sudo systemctl status tunnel-aws
# Expected: Active: active (running)

# Step 3: Prometheus scraping
curl http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | {job, health}'
# Expected: "health": "up"

# Step 4: Grafana querying Prometheus
curl http://localhost:3000/api/datasources/proxy/1/api/v1/query?query=up
# Expected: {"status":"success","data":{"result":[...]}}

# Step 5: Browser access
# Open: http://localhost:3000
# Expected: Grafana login page
