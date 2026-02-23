# 🚀 Splunk Server Monitoring with Universal Forwarder

---

# 📌 Project Overview

This project demonstrates a complete enterprise-style monitoring setup using:

- Splunk Enterprise (Indexer + Search Head)
- Splunk Universal Forwarder (Data Collector)
- Linux system metrics from `top`
- CPU & Memory dashboard
- Threshold-based alerts

---


## 📸 Dashboard Screenshot

![Splunk Dashboard Last 15 Minutes](https://github.com/aniljadhavmca/Splunk-Server-Monitoring/raw/main/images/dashboard-last-15m.png)

![Splunk timechart Last 15 Minutes](https://github.com/aniljadhavmca/Splunk-Server-Monitoring/blob/main/images/Dashboard-before-stress.png)


*Dashboard showing CPU and Memory usage for the last 15 minutes*

# 🏗 Architecture

- Linux Server (Forwarder Installed)
- Splunk Universal Forwarder
- Splunk Enterprise (Indexer + Search Head)
- Dashboard + Alerts

---

# 🖥 Environment Details

| Component | Description |
|------------|-------------|
| Splunk Enterprise | Installed on EC2 (Port 8000 / 9997) |
| Splunk Forwarder | Installed on Linux Server |
| Log File | /var/log/simple_metrics.log |
| Metrics Source | Linux `top` command |

---

# 🔹 STEP 1 – Install Splunk Enterprise (Indexer)

### Download

```bash
wget -O splunk-9.3.1.rpm https://download.splunk.com/products/splunk/releases/9.3.1/linux/splunk-9.3.1-0b8d769cb912.x86_64.rpm
```

### Install

```bash
sudo rpm -ivh splunk-9.3.1.rpm
```

### Start Splunk

```bash
sudo /opt/splunk/bin/splunk start --accept-license
```

### Enable Boot Start

```bash
sudo /opt/splunk/bin/splunk enable boot-start
```

### Enable Receiving Port (Very Important)

```bash
sudo /opt/splunk/bin/splunk enable listen 9997
```

Verify:

```bash
sudo /opt/splunk/bin/splunk list listen
```

---

Access Web UI:

```
http://<Public-IP>:8000
```

---

# 🔹 STEP 2 – Install Splunk Universal Forwarder

### Download

```bash
wget -O splunkforwarder-9.3.1.rpm https://download.splunk.com/products/universalforwarder/releases/9.3.1/linux/splunkforwarder-9.3.1-0b8d769cb912.x86_64.rpm
```

### Install

```bash
sudo rpm -ivh splunkforwarder-9.3.1.rpm
```

### Start Forwarder

```bash
sudo /opt/splunkforwarder/bin/splunk start --accept-license
```

### Configure Forwarder to Send Logs to Splunk Enterprise

```bash
sudo /opt/splunkforwarder/bin/splunk add forward-server <Indexer-IP>:9997
```

Verify:

```bash
sudo /opt/splunkforwarder/bin/splunk list forward-server
```

---

# 🔹 STEP 3 – Generate Linux Metrics

Create logging loop:

```bash
nohup bash -c 'while true; do top -bn1 | head -5 >> /var/log/simple_metrics.log; sleep 10; done' >/dev/null 2>&1 &
```

Verify:

```bash
tail -f /var/log/simple_metrics.log
```

---

# 🔹 STEP 4 – Configure Forwarder to Monitor Log

```bash
sudo /opt/splunkforwarder/bin/splunk add monitor /var/log/simple_metrics.log
```

Restart Forwarder:

```bash
sudo /opt/splunkforwarder/bin/splunk restart
```

---

# 🔹 STEP 5 – Verify Data in Splunk

In Splunk Search:

```spl
source="/var/log/simple_metrics.log"
```

Check event count:

```spl
index=* source="/var/log/simple_metrics.log" | stats count
```

---

# 🔹 CPU Extraction Query

```spl
source="/var/log/simple_metrics.log"
| rex "%Cpu\(s\):.*?(?<idle>[0-9.]+)\s+id"
| eval cpu_usage=round(100-idle,2)
| stats avg(cpu_usage) as CPU_Usage
```

---

# 🔹 CPU Timechart

```spl
source="/var/log/simple_metrics.log"
| rex "%Cpu\(s\):.*?(?<idle>[0-9.]+)\s+id"
| eval cpu_usage=round(100-idle,2)
| timechart avg(cpu_usage) as CPU_Usage
```

---

# 🔹 Memory Extraction Query

```spl
source="/var/log/simple_metrics.log"
| rex "MiB Mem\s*:\s*(?<total>[0-9.]+)\s+total.*?(?<used>[0-9.]+)\s+used"
| eval mem_percent=round((used/total)*100,2)
| stats avg(mem_percent) as Memory_Usage
```

---

# 🔹 Memory Timechart

```spl
source="/var/log/simple_metrics.log"
| rex "MiB Mem\s*:\s*(?<total>[0-9.]+)\s+total.*?(?<used>[0-9.]+)\s+used"
| eval mem_percent=round((used/total)*100,2)
| timechart avg(mem_percent) as Memory_Usage
```

---

# 🔹 Create Alert (CPU > 80%)

```spl
source="/var/log/simple_metrics.log"
| rex "%Cpu\(s\):.*?(?<idle>[0-9.]+)\s+id"
| eval cpu_usage=100-idle
| where cpu_usage > 80
```

Alert Settings:

- Scheduled
- Run every 5 minutes
- Time range: Last 5 minutes
- Trigger when results > 0
- Throttle 10 minutes

---

# 🔹 Simulate High CPU

```bash
sudo yum install stress -y
stress --cpu 4 --timeout 180
```

---

# 🧠 Concepts Demonstrated

- Splunk Enterprise Installation
- Splunk Universal Forwarder Configuration
- Forwarder to Indexer Communication
- Log Monitoring
- Field Extraction using rex
- Data Transformation using eval
- Dashboard Creation
- Threshold-based Alerting
- Infrastructure Monitoring

---

# 💼 Interview Explanation

> Implemented an enterprise-style monitoring architecture using Splunk Enterprise and Universal Forwarder. Configured forwarder-to-indexer communication, parsed Linux system metrics using regex, built dynamic dashboards with time-based filtering, and implemented scheduled threshold alerts for proactive monitoring.

---

# 👨‍💻 Author

Anil  
AWS | DevOps | Splunk Practice Project
