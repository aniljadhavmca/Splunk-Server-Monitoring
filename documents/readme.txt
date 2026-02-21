# 🚀 Splunk Server Monitoring Dashboard (CPU & Memory Monitoring)

---

# 📌 Project Purpose

This project demonstrates how to build a complete **Linux Server Monitoring Solution using Splunk Enterprise**.

The solution:

- Collects Linux system metrics using the `top` command
- Logs CPU and Memory usage into a file
- Ingests the log file into Splunk
- Extracts CPU & Memory values using regex
- Visualizes usage in dashboards
- Implements threshold-based alerts
- Supports dynamic time range selection

This project simulates a real-world infrastructure monitoring setup.

---

# 🏗 Architecture Overview

Linux EC2 Instance  
→ System metrics logged using `top`  
→ Log stored at `/var/log/simple_metrics.log`  
→ Splunk monitors the log  
→ Dashboard visualizes metrics  
→ Alert triggers when CPU > 80%

---

# 🖥 Environment Details

| Component | Value |
|------------|--------|
| OS | Amazon Linux |
| Monitoring Tool | Splunk Enterprise |
| Log File | /var/log/simple_metrics.log |
| Metric Source | Linux `top` command |
| Alert Type | Scheduled Alert |

---

# 🔧 STEP 1 – Install Splunk Enterprise

Download Splunk:

```bash
wget -O splunk-9.3.1.rpm https://download.splunk.com/products/splunk/releases/9.3.1/linux/splunk-9.3.1-0b8d769cb912.x86_64.rpm
```

Install:

```bash
sudo rpm -ivh splunk-9.3.1.rpm
```

Start Splunk:

```bash
sudo /opt/splunk/bin/splunk start --accept-license
```

Enable auto-start:

```bash
sudo /opt/splunk/bin/splunk enable boot-start
```

Access Web UI:

```
http://<EC2-Public-IP>:8000
```

---

# 📊 STEP 2 – Generate System Metrics Log

Create continuous log generation using `top`:

```bash
nohup bash -c 'while true; do top -bn1 | head -5 >> /var/log/simple_metrics.log; sleep 10; done' >/dev/null 2>&1 &
```

Verify logs are updating:

```bash
tail -f /var/log/simple_metrics.log
```

Purpose:
- Captures CPU usage
- Captures Memory usage
- Appends to log file every 10 seconds

---

# 📥 STEP 3 – Add Log Monitoring in Splunk

Add monitor:

```bash
/opt/splunk/bin/splunk add monitor /var/log/simple_metrics.log
```

Restart Splunk:

```bash
/opt/splunk/bin/splunk restart
```

Verify ingestion:

```spl
source="/var/log/simple_metrics.log" | head 10
```

---

# 🔍 STEP 4 – Extract CPU Usage

Linux `top` output example:

```
%Cpu(s): 3.1 us, 0.0 sy, 0.0 ni, 96.9 id
```

CPU calculation:

```
CPU Usage = 100 - idle
```

Splunk Query:

```spl
source="/var/log/simple_metrics.log"
| rex "%Cpu\(s\):.*?(?<idle>[0-9.]+)\s+id"
| eval cpu_usage=round(100-idle,2)
| stats avg(cpu_usage) as CPU_Usage
```

---

# 📈 CPU Timechart Query

```spl
source="/var/log/simple_metrics.log"
| rex "%Cpu\(s\):.*?(?<idle>[0-9.]+)\s+id"
| eval cpu_usage=round(100-idle,2)
| timechart avg(cpu_usage) as CPU_Usage
```

Purpose:
- Shows CPU trend over selected time range
- Helps detect performance spikes

---

# 🔍 STEP 5 – Extract Memory Usage

Linux Memory Example:

```
MiB Mem : 3916.1 total, 170.4 free, 824.7 used
```

Splunk Query:

```spl
source="/var/log/simple_metrics.log"
| rex "MiB Mem\s*:\s*(?<total>[0-9.]+)\s+total.*?(?<used>[0-9.]+)\s+used"
| eval mem_percent=round((used/total)*100,2)
| stats avg(mem_percent) as Memory_Usage
```

---

# 📈 Memory Timechart Query

```spl
source="/var/log/simple_metrics.log"
| rex "MiB Mem\s*:\s*(?<total>[0-9.]+)\s+total.*?(?<used>[0-9.]+)\s+used"
| eval mem_percent=round((used/total)*100,2)
| timechart avg(mem_percent) as Memory_Usage
```

Purpose:
- Tracks memory utilization trend
- Detects memory pressure over time

---

# 🎨 STEP 6 – Dashboard Features

Dashboard Includes:

- 🟢 Server Health Status Panel
- 📊 CPU Usage Single Value
- 📊 Memory Usage Single Value
- 📈 CPU Trend Chart
- 📈 Memory Trend Chart
- 🕒 Time Range Picker (Last 60m / 24h / Custom)
- 🔄 Auto Refresh (30 seconds)
- 🎨 Dark Theme

Purpose of Dashboard:

- Real-time infrastructure visibility
- Threshold-based monitoring
- Performance trend analysis
- Operational alert validation

---

# 🚨 STEP 7 – Create CPU Alert (>80%)

Alert Query:

```spl
source="/var/log/simple_metrics.log"
| rex "%Cpu\(s\):.*?(?<idle>[0-9.]+)\s+id"
| eval cpu_usage=100-idle
| where cpu_usage > 80
```

Alert Configuration:

| Setting | Value |
|----------|--------|
| Alert Type | Scheduled |
| Run Every | 5 minutes |
| Time Range | Last 5 minutes |
| Trigger | Results > 0 |
| Throttle | 10 minutes |

Purpose:
- Detect abnormal CPU spikes
- Prevent system overload
- Provide early warning

---

# 🔥 STEP 8 – Simulate High CPU for Testing

Install stress tool:

```bash
sudo yum install stress -y
```

Run CPU stress:

```bash
stress --cpu 4 --timeout 180
```

Alternative:

```bash
yes > /dev/null &
yes > /dev/null &
yes > /dev/null &
yes > /dev/null &
```

---

# 🧠 Splunk Concepts Used

- Field Extraction using `rex`
- Data transformation using `eval`
- Aggregation using `stats`
- Trend analysis using `timechart`
- Dashboard tokens
- Scheduled Alerts
- Throttling
- Log Monitoring

---

# 💼 Interview Explanation

> Designed and implemented a Linux server monitoring solution using Splunk Enterprise. Parsed system metrics from top command logs using regex, visualized CPU and memory utilization trends, implemented dynamic dashboards with tokenized time filters, and configured threshold-based alerting for proactive infrastructure monitoring.

---

# 📌 Future Enhancements

- Multi-host monitoring
- Host dropdown filter
- Email integration
- SNS / Slack alerts
- Load Average monitoring
- Process-level monitoring
- Infrastructure health scoring

---

# 👨‍💻 Author

Anil  
DevOps / AWS / Splunk Practice Project
