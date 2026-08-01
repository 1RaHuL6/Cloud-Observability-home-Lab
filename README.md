# Cloud-Observability-home-Lab
A hands-on project building a hybrid cloud monitoring system using Prometheus, Grafana, and Node Exporter, designed to demonstrate production-grade observability skills.


# 🌍 Multi-Cloud Monitoring Lab: AWS + Local VM Observability 

[![Status: In Progress](https://img.shields.io/badge/Status-In%20Progress-yellow)](#)
[![AWS](https://img.shields.io/badge/Cloud-AWS-orange)](https://aws.amazon.com)
[![Prometheus](https://img.shields.io/badge/Monitoring-Prometheus-blue)](https://prometheus.io)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)


---

## 🎯 Project Goal

Build a **safe, reproducible monitoring system** that:
- ✅ Monitors a production-like application on AWS EC2
- ✅ Collects system metrics via Node Exporter
- ✅ Aggregates metrics in Prometheus on a local VirtualBox VM
- ✅ Demonstrates secure cross-cloud connectivity (SSH tunnels)
- ✅ Prepares for multi-cloud expansion (Azure next!)

**Why this matters**: Production support, SRE, and cloud roles require hands-on experience with observability tools. This project proves I can design, deploy, and troubleshoot real monitoring infrastructure.

---

## 🏗️ Architecture Overview

☁️ AWS EC2 (us-east-1)

├── Public IP: [Elastic IP] (static)

├── Web App (Port 5000)

│ ├── GET / → Health check endpoint

│ └── Running as systemd service (auto-restart)

├── Node Exporter (Port 9100)

│ ├── Exposes CPU, memory, disk, network metrics

│ └── Waits for Prometheus to "pull" metrics (PULL model)

└── Security Group

├── Port 22: SSH (my IP only)

├── Port 5000: Flask app (my IP only)

└── Port 9100: Node Exporter (my IP only)


🏠 Windows Host + VirtualBox

├── Monitoring VM (Ubuntu 22.04)

│ ├── Prometheus (Port 9090)

│ │ ├── Scrapes AWS metrics via SSH tunnel

│ │ ├── Configuration: prometheus.yml

│ │ └── Web UI: http://localhost:9090 (port-forwarded)

│ ├── SSH Tunnel (autossh)

│ │ ├── Persistent tunnel: localhost:9101 → AWS:9100

│ │ └── Auto-reconnect on failure

│ └── Port Forwarding (VirtualBox)

│ ├── 2222 (host) → 22 (guest): SSH management

│ └── 9090 (host) → 9090 (guest): Prometheus UI access

└── Future: Grafana (Port 3000) for dashboards



🔗 Data Flow:

AWS Node Exporter (9100)

 SSH Tunnel (9101)
 
 Prometheus scrape (every 15s)
 
 Grafana query (future)
 
 Human observation via browser
 
