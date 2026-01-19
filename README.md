# Workstations Hardware Resources Monitoring User Guide

**Author:** Kshitij Kumar Pandey

## Table of Contents

1. [Introduction](#1-introduction)
2. [How to Access](#2-how-to-access)
3. [Dashboard Layout](#3-dashboard-layout)

## 1. Introduction

We have deployed a real-time monitoring system to track the health and availability of our shared compute nodes (**Telemaco** and **Penelope**). Before launching heavy simulations or training jobs, please check this dashboard to ensure the hardware is free and healthy.

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

### 4. How to Deploy on Your Workstation

If you want to deploy this monitoring service on your workstation (so it is accessible via `http://<your-hostname>.maths.sissa.it:3000`), follow these steps.

**Prerequisites:**

- Linux Workstation (Ubuntu/CentOS).
- NVIDIA Drivers installed.

**Section Overview:**

1. Download the official binaries for **Prometheus** (Database), **Grafana** (Dashboard), and **Exporters** (Hardware Probes) into a `~/Monitoring` folder.
2. Configure Prometheus to listen to your local CPU and GPU.
3. Configure Grafana to allow **public read-only access** (so you don't need to log in).
4. Start all services in the background.
5. Import the dashboard.

### **Download & Extract**

Copy and paste this block to create the directory and download the required software.

```bash
mkdir -p ~/Monitoring && cd ~/Monitoring

# Download
wget -q -nc [https://github.com/prometheus/prometheus/releases/download/v2.45.0/prometheus-2.45.0.linux-amd64.tar.gz](https://github.com/prometheus/prometheus/releases/download/v2.45.0/prometheus-2.45.0.linux-amd64.tar.gz)
wget -q -nc [https://dl.grafana.com/oss/release/grafana-10.2.0.linux-amd64.tar.gz](https://dl.grafana.com/oss/release/grafana-10.2.0.linux-amd64.tar.gz)
wget -q -nc [https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz](https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz)
wget -q -nc [https://github.com/utkuozdemir/nvidia_gpu_exporter/releases/download/v0.5.0/nvidia_gpu_exporter_0.5.0_linux_x86_64.tar.gz](https://github.com/utkuozdemir/nvidia_gpu_exporter/releases/download/v0.5.0/nvidia_gpu_exporter_0.5.0_linux_x86_64.tar.gz)

# Extract
tar -xzf prometheus-2.45.0.linux-amd64.tar.gz
tar -xzf grafana-10.2.0.linux-amd64.tar.gz
tar -xzf node_exporter-1.7.0.linux-amd64.tar.gz
tar -xzf nvidia_gpu_exporter_0.5.0_linux_x86_64.tar.gz
```

### **Configuration**

Copy and paste this block to configure Prometheus to listen to your hardware and Grafana to allow access without a password.

```bash
cd ~/Monitoring

# Configure Prometheus (Database)
cat > prometheus-2.45.0.linux-amd64/prometheus.yml <<EOF
global:
  scrape_interval: 5s
scrape_configs:
  - job_name: 'local_workstation'
    static_configs:
      - targets: ['localhost:9100', 'localhost:9835']
EOF

# Configure Grafana (Dashboard) to allow Anonymous Viewers
CONF="./grafana-10.2.0/conf/defaults.ini"
sed -i 's/^;http_addr =.*/http_addr = 0.0.0.0/' $CONF
sed -i '/^\[auth.anonymous\]/,/^\[/ s/^;enabled = false/enabled = true/' $CONF
sed -i '/^\[auth.anonymous\]/,/^\[/ s/^enabled = false/enabled = true/' $CONF
sed -i '/^\[auth.anonymous\]/,/^\[/ s/^;org_role = .*/org_role = Viewer/' $CONF
```

### **Launch Services**

Run this block to start the background processes.

```bash
cd ~/Monitoring

# Start Exporters and Services
nohup ./node_exporter-1.7.0.linux-amd64/node_exporter --web.listen-address=":9100" > node.log 2>&1 &
nohup ./nvidia_gpu_exporter_0.5.0_linux_x86_64/nvidia_gpu_exporter --web.listen-address=":9835" > gpu.log 2>&1 &
cd prometheus-2.45.0.linux-amd64 && nohup ./prometheus > ../prometheus.log 2>&1 &
cd ../grafana-10.2.0 && nohup ./bin/grafana-server > ../grafana.log 2>&1 &

echo "✅ Monitoring Online! Access at: http://$(hostname).maths.sissa.it:3000"
```

### **Import the Dashboard**

1. Open the link generated by the script.
2. Login once with `admin` / `admin`.
3. Go to **Dashboards** $\rightarrow$ **New** $\rightarrow$ **Import**.
4. Upload the `Hardware Resources.json` file (attached below).
  
