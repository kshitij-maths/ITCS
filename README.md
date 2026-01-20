# Workstations Hardware Resources Monitoring User Guide

**Author:** Kshitij Kumar Pandey

## Table of Contents

1. [Introduction](#1-introduction)
2. [How to Access](#2-how-to-access)
3. [Dashboard Layout](#3-dashboard-layout)
4. [How to Deploy on Your Workstation](#4-how-to-deploy-on-your-workstation)

## 1. Introduction

We have deployed a real-time monitoring system to track the health and availability of our shared compute nodes (**Telemaco** and **Penelope**). Before launching heavy simulations or training jobs, please check this dashboard to ensure the hardware is free and healthy.



[Image of monitoring system architecture]


## 2. How to Access

We have centralized the monitoring on **Duchatelet**, allowing you to view both workstations from a single link. No password or SSH tunneling is required.

**Note:** You must be connected to the **SISSA Network** to access this link.

### **Step 1: Open the Dashboard**

Click the link below to open the monitor in your browser:

👉 **[http://duchatelet.maths.sissa.it:3000](http://duchatelet.maths.sissa.it:3000)**

### **Step 2: Select a Workstation**

The dashboard displays one node at a time. To switch views:

1. Locate the **"instance"** dropdown menu at the top-left corner of the screen.
2. Select the workstation you wish to inspect (e.g., `telemaco` or `penelope`).
3. The graphs will update to show the real-time metrics for that specific machine.

*(Note: No login credentials are required. Access is public within the internal network).*

## 3. Dashboard Layout

The dashboard is organized into three distinct rows to help you quickly assess system health:

### **Row 1: Current Status**

- **CPU & RAM Gauges:** Shows the current load percentage. If these are red, the system is fully loaded.
- **Disk Usage:** A pie chart showing the used vs. free space on the main storage (`/` and `/mnt/raid`).
- **Disk I/O & Network:** Real-time throughput graphs to check if a job is writing heavy data or transferring files.

### **Row 2: GPU Status (Current)**

- **Real-time Gauges:** Displays **Utilization**, **Memory**, **Temperature**, and **Power Draw** for every NVIDIA GPU installed on the node.



### **Row 3: History & Trends**

- **Time-Series Graphs:** Scroll down to view the performance history over the last selected time period.
- **Usage:** To identify when a crash occurred, if any (e.g., a sudden drop in GPU memory usage), or to verify if the node was busy recently.

## 4. How to Deploy on Your Workstation

If you want to deploy this monitoring service on your workstation (so it is accessible via `http://<your-hostname>.maths.sissa.it:3000`), follow these steps.

**Prerequisites:**

- Linux Workstation (Ubuntu/CentOS).
- NVIDIA Drivers installed.

**Overview:**

1. Download the official binaries for **Prometheus**, **Grafana**, and **Exporters**.
2. Configure Prometheus to listen to your local CPU and GPU.
3. Configure Grafana to **automatically load** the dashboard and set it as **Home**.
4. Start all services in the background using a startup script.

### **Step 1: Download & Extract**

Copy and paste this block. It uses absolute paths to ensure files are placed correctly.

```bash
# Create directory
mkdir -p $HOME/Monitoring
cd $HOME/Monitoring

# Download Components (Plain text URLs)
wget -q -nc https://github.com/prometheus/prometheus/releases/download/v2.45.0/prometheus-2.45.0.linux-amd64.tar.gz
wget -q -nc https://dl.grafana.com/oss/release/grafana-10.2.0.linux-amd64.tar.gz
wget -q -nc https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
wget -q -nc https://github.com/utkuozdemir/nvidia_gpu_exporter/releases/download/v0.5.0/nvidia_gpu_exporter_0.5.0_linux_x86_64.tar.gz

# Extract
tar -xzf prometheus-2.45.0.linux-amd64.tar.gz
tar -xzf grafana-10.2.0.linux-amd64.tar.gz
tar -xzf node_exporter-1.7.0.linux-amd64.tar.gz
tar -xzf nvidia_gpu_exporter_0.5.0_linux_x86_64.tar.gz
```

### **Step 2: Configuration**

This block configures the ports, creates the necessary folders, downloads the dashboard file, and connects everything automatically.

```bash
cd $HOME/Monitoring

# 1. Configure Prometheus (The Database)
# Note: Using ports 9091/9101 to avoid conflicts
cat > $HOME/Monitoring/prometheus-2.45.0.linux-amd64/prometheus.yml <<EOF
global:
  scrape_interval: 5s
scrape_configs:
  - job_name: 'local_workstation'
    static_configs:
      - targets: ['localhost:9101', 'localhost:9836']
EOF

# 2. Setup Dashboard Provisioning Folders
mkdir -p $HOME/Monitoring/grafana-10.2.0/conf/provisioning/datasources
mkdir -p $HOME/Monitoring/grafana-10.2.0/conf/provisioning/dashboards
mkdir -p $HOME/Monitoring/dashboards

# 3. Download the Dashboard JSON into the correct folder
wget -O $HOME/Monitoring/dashboards/Hardware_Resources.json https://raw.githubusercontent.com/kshitij-maths/ITCS/main/Hardware_Resources.json

# 4. Configure Grafana (Custom Settings)
# - Port 3000
# - Anonymous Access Enabled
# - Auto-Set Home Dashboard
cat > $HOME/Monitoring/grafana-10.2.0/conf/custom.ini <<EOF
[server]
http_port = 3000
http_addr = 0.0.0.0

[auth.anonymous]
enabled = true
org_role = Viewer

[dashboards]
default_home_dashboard_path = $HOME/Monitoring/dashboards/Hardware_Resources.json
EOF

# 5. Auto-Connect Prometheus Data
cat > $HOME/Monitoring/grafana-10.2.0/conf/provisioning/datasources/prometheus.yml <<EOF
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://localhost:9091
    isDefault: true
EOF

# 6. Auto-Load Dashboards from Folder
cat > $HOME/Monitoring/grafana-10.2.0/conf/provisioning/dashboards/main.yml <<EOF
apiVersion: 1
providers:
  - name: 'Default'
    orgId: 1
    folder: ''
    type: file
    disableDeletion: false
    editable: true
    options:
      path: $HOME/Monitoring/dashboards
EOF
```

### **Step 3: Launch Services**

This block creates a helper script start_monitoring.sh so you can easily restart the services later.

```bash
# Create the startup script
cat > $HOME/Monitoring/start_monitoring.sh << 'EOF'
#!/bin/bash

# Define Paths
DIR="$HOME/Monitoring"
cd "$DIR"

# 1. Start Node Exporter (Port 9101)
nohup "$DIR/node_exporter-1.7.0.linux-amd64/node_exporter" --web.listen-address=":9101" > node.log 2>&1 &

# 2. Start GPU Exporter (Port 9836)
if [ -f "$DIR/nvidia_gpu_exporter" ]; then
    chmod +x "$DIR/nvidia_gpu_exporter"
    nohup "$DIR/nvidia_gpu_exporter" --web.listen-address=":9836" > gpu.log 2>&1 &
fi

# 3. Start Prometheus (Port 9091)
cd "$DIR/prometheus-2.45.0.linux-amd64" 
nohup ./prometheus --config.file=prometheus.yml --web.listen-address="0.0.0.0:9091" > ../prometheus.log 2>&1 &

# 4. Start Grafana (Port 3000)
cd "$DIR/grafana-10.2.0" 
nohup ./bin/grafana-server > ../grafana.log 2>&1 &
EOF

# Make the script executable and run it
chmod +x $HOME/Monitoring/start_monitoring.sh
$HOME/Monitoring/start_monitoring.sh

echo "✅ Monitoring Online!"
echo "👉 Access at: http://$(hostname -s).maths.sissa.it:3000"
```
