
🧩 **PHASE 1 — FIX DATA INGESTION**
---
🔴 STARTING POINT
```bash
index=main source="/var/log/simple_metrics.log"
| rex "%Cpu\(s\):.*?(?<idle>\d+\.\d+)\s+id"
| table idle
```

And got:

❌ 0 events

**✅ STEP 1 — Check If File Exists**
On EC2:
```bash
ls -l /var/log/simple_metrics.log
```

If file NOT found:
```bash
sudo touch /var/log/simple_metrics.log

sudo chmod 644 /var/log/simple_metrics.log

```

**✅ STEP 2 — Configure Splunk To Monitor File**
```bash
sudo mkdir -p /opt/splunk/etc/system/local
```

```bash
sudo vi /opt/splunk/etc/system/local/inputs.conf
```
Add:
```bash
[monitor:///var/log/simple_metrics.log]
disabled = false
index = main
sourcetype = simple_metrics
```
Save

**✅ STEP 3 — Restart Splunk**
```bash
sudo /opt/splunk/bin/splunk restart
```
**✅ STEP 4 — Test Basic Ingestion**
In Splunk Search:
```bash
index=main sourcetype=simple_metrics
| stats count
```
***If still 0 → go to Phase 2.***


**🧩 PHASE 2 — FIX TIMESTAMP PROBLEM**
---
⚠️ Without timestamp, Splunk assigns wrong time (2038 issue).

**✅ STEP 5 — Clear Old Log**
```bash
sudo truncate -s 0 /var/log/simple_metrics.log
```

**✅ STEP 6 — Add props.conf**
```bash
sudo vi /opt/splunk/etc/system/local/props.conf
```
Add

```bash
[simple_metrics]
TIME_PREFIX = ^
TIME_FORMAT = %Y-%m-%d %H:%M:%S
MAX_TIMESTAMP_LOOKAHEAD = 19
```

Restart Splunk again:
```bash
sudo /opt/splunk/bin/splunk restart
```


**🧩 PHASE 3 — START CORRECT LOGGING**
---

**✅ STEP 7 — Start Continuous Logging With Timestamp**

RUN on EC2
```bash
nohup sh -c 'while true; do
  echo "$(date "+%Y-%m-%d %H:%M:%S") $(top -b -n1 | grep "Cpu(s)")" >> /var/log/simple_metrics.log
  echo "$(date "+%Y-%m-%d %H:%M:%S") $(top -b -n1 | grep "MiB Mem")" >> /var/log/simple_metrics.log
  sleep 5
done' &
```

Leave running.

**✅ STEP 8 — Verify File Updating**

Open new terminal:
```bash
tail -f /var/log/simple_metrics.log
```

You must see:

2026-02-25 21:45:01 %Cpu(s): ...
2026-02-25 21:45:01 MiB Mem : ...


**🧩 PHASE 4 — VERIFY SPLUNK TIME**
---

**✅ STEP 9 — Check Splunk _time**

Run:
```bash
index=main sourcetype=simple_metrics
| table _time _raw
| head 5
```

✔ _time must match current time
❌ Not 2038
❌ Not old time


**🧩 PHASE 5 — TEST CPU EXTRACTION**
---

**✅ STEP 10 — Convert To CPU Usage**
```bash
index=main sourcetype=simple_metrics
| rex "%Cpu\(s\):.*?(?<idle>\d+\.\d+)\s+id"
| eval cpu_usage=round(100-idle,2)
| table _time cpu_usage
```


**🧩 PHASE 6 — STRESS TEST**
---

**✅ STEP 11 — Install Stress**
```bash
sudo yum install stress -y
```
**✅ STEP 12 — Run CPU Stress**
```bash
stress --cpu $(nproc) --timeout 60
```
**✅ STEP 13 — Verify Spike**
In Splunk:
```bash
index=main sourcetype=simple_metrics
| rex "%Cpu\(s\):.*?(?<idle>\d+\.\d+)\s+id"
| eval cpu_usage=round(100-idle,2)
| timechart span=10s avg(cpu_usage)
```
You MUST see spike above 80–95%.

🧩 PHASE 7 — CREATE DASHBOARD (CLASSIC)
---
```bash

