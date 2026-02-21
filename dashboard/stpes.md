# 📌 Dependencies for CPU & Memory Dashboard

Before creating the Splunk dashboard or configuring alerts, the following components must be properly set up. The dashboard depends on successful data ingestion and field extraction.

---

## 1️⃣ Splunk Enterprise Installed and Running

Splunk must be installed and active.

Check status:

```bash
/opt/splunk/bin/splunk status
```

Access Web UI:

```
http://<server-ip>:8000
```

If Splunk is not running, dashboards and searches will not function.

---

## 2️⃣ Receiving Port Enabled (If Using Universal Forwarder)

If logs are coming from another server, Splunk must listen on port 9997.

Enable receiving:

```bash
/opt/splunk/bin/splunk enable listen 9997
```

Verify:

```bash
/opt/splunk/bin/splunk list listen
```

Without this, forwarder cannot send data.

---

## 3️⃣ Splunk Universal Forwarder Installed (If Remote Server)

Install Forwarder:

```bash
rpm -ivh splunkforwarder-9.3.1.rpm
```

Start Forwarder:

```bash
/opt/splunkforwarder/bin/splunk start --accept-license
```

Connect to Splunk Enterprise:

```bash
/opt/splunkforwarder/bin/splunk add forward-server <Indexer-IP>:9997
```

Verify connection:

```bash
/opt/splunkforwarder/bin/splunk list forward-server
```

---

## 4️⃣ CPU & Memory Log Generation

The dashboard depends on this log file:

```
/var/log/simple_metrics.log
```

Generate system metrics continuously:

```bash
nohup bash -c 'while true; do top -bn1 | head -5 >> /var/log/simple_metrics.log; sleep 10; done' >/dev/null 2>&1 &
```

Verify log is updating:

```bash
tail -f /var/log/simple_metrics.log
```

If this file is not updating, CPU and Memory panels will show no data.

---

## 5️⃣ Log Monitoring Configuration

Splunk must monitor the metrics file.

Enterprise:

```bash
/opt/splunk/bin/splunk add monitor /var/log/simple_metrics.log
```

Forwarder:

```bash
/opt/splunkforwarder/bin/splunk add monitor /var/log/simple_metrics.log
```

Restart after adding monitor:

```bash
splunk restart
```

---

## 6️⃣ Data Verification Before Dashboard Creation

Confirm data ingestion:

```spl
source="/var/log/simple_metrics.log" | head 5
```

Confirm CPU extraction works:

```spl
source="/var/log/simple_metrics.log"
| rex "%Cpu\(s\):.*?(?<idle>[0-9.]+)\s+id"
| eval cpu_usage=100-idle
| table cpu_usage
```

If this query fails, the dashboard will not display results.

---

# 🔎 Summary of Required Dependencies

| Dependency | Required | Purpose |
|------------|----------|----------|
| Splunk Enterprise Running | ✅ | Dashboard & search functionality |
| Receiving Port 9997 | ✅ (if forwarder used) | Accept logs from forwarder |
| Universal Forwarder | Optional | Remote server log collection |
| Log Generation Script | ✅ | Produces CPU & Memory logs |
| Log Monitoring | ✅ | Sends data into Splunk index |
| Working SPL Query | ✅ | Extracts CPU & Memory values |

---

# 🎯 Key Concept

The dashboard is only a visualization layer.

If:

- Logs are not generated ❌  
- Logs are not indexed ❌  
- Field extraction fails ❌  

Then the dashboard will display:

```
No results found
```

Proper ingestion and parsing must be completed before dashboard creation.
