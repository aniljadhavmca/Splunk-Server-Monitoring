
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

(you can run stress after creating dashboards)

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

Creates clasic dashboard, using this XML and refresh dashboard immidetly.

```bash
<dashboard version="1.1" refresh="30" theme="dark">
  <label>Server Monitoring Pro</label>

  <!-- TIME PICKER -->
  <fieldset submitButton="false">
    <input type="time" token="time_range">
      <label>Select Time Range</label>
      <default>
        <earliest>-15m</earliest>
        <latest>now</latest>
      </default>
    </input>
  </fieldset>

  <!-- SERVER HEALTH -->
  <row>
    <panel>
      <title>Server Health Status</title>
      <single>
        <search>
          <query><![CDATA[
index=main sourcetype=simple_metrics
| rex "%Cpu\(s\):.*?(?<idle>\d+\.\d+)\s+id"
| rex "MiB Mem\s*:\s*(?<total>\d+\.\d+)\s+total.*?(?<used>\d+\.\d+)\s+used"
| eval cpu_usage=round(100-tonumber(idle),2)
| eval mem_percent=round((tonumber(used)/tonumber(total))*100,2)
| eval health=if(cpu_usage>80 OR mem_percent>80,1,0)
| stats max(health) as health
          ]]></query>
          <earliest>$time_range.earliest$</earliest>
          <latest>$time_range.latest$</latest>
        </search>
        <option name="colorMode">block</option>
        <option name="rangeValues">1</option>
        <option name="rangeColors">["green","red"]</option>
      </single>
    </panel>
  </row>

  <!-- CPU + MEMORY SINGLE -->
  <row>
    <panel>
      <title>CPU Usage (%)</title>
      <single>
        <search>
          <query><![CDATA[
index=main sourcetype=simple_metrics
| rex "%Cpu\(s\):.*?(?<idle>\d+\.\d+)\s+id"
| eval cpu_usage=round(100-tonumber(idle),2)
| stats avg(cpu_usage) as CPU_Usage
          ]]></query>
          <earliest>$time_range.earliest$</earliest>
          <latest>$time_range.latest$</latest>
        </search>
        <option name="unit">%</option>
        <option name="colorMode">block</option>
        <option name="rangeValues">70,85</option>
        <option name="rangeColors">["green","orange","red"]</option>
      </single>
    </panel>

    <panel>
      <title>Memory Usage (%)</title>
      <single>
        <search>
          <query><![CDATA[
index=main sourcetype=simple_metrics
| rex "MiB Mem\s*:\s*(?<total>\d+\.\d+)\s+total.*?(?<used>\d+\.\d+)\s+used"
| eval mem_percent=round((tonumber(used)/tonumber(total))*100,2)
| stats avg(mem_percent) as Memory_Usage
          ]]></query>
          <earliest>$time_range.earliest$</earliest>
          <latest>$time_range.latest$</latest>
        </search>
        <option name="unit">%</option>
        <option name="colorMode">block</option>
        <option name="rangeValues">70,85</option>
        <option name="rangeColors">["green","orange","red"]</option>
      </single>
    </panel>
  </row>

  <!-- TRENDS -->
  <row>
    <panel>
      <title>CPU Usage Trend</title>
      <chart>
        <search>
          <query><![CDATA[
index=main sourcetype=simple_metrics
| rex "%Cpu\(s\):.*?(?<idle>\d+\.\d+)\s+id"
| eval cpu_usage=round(100-tonumber(idle),2)
| timechart span=10s avg(cpu_usage) as CPU_Usage
          ]]></query>
          <earliest>$time_range.earliest$</earliest>
          <latest>$time_range.latest$</latest>
        </search>
        <option name="charting.chart">line</option>
        <option name="charting.axisY.minimumNumber">0</option>
        <option name="charting.axisY.maximumNumber">100</option>
        <option name="charting.legend.placement">right</option>
      </chart>
    </panel>

    <panel>
      <title>Memory Usage Trend</title>
      <chart>
        <search>
          <query><![CDATA[
index=main sourcetype=simple_metrics
| rex "MiB Mem\s*:\s*(?<total>\d+\.\d+)\s+total.*?(?<used>\d+\.\d+)\s+used"
| eval mem_percent=round((tonumber(used)/tonumber(total))*100,2)
| timechart span=10s avg(mem_percent) as Memory_Usage
          ]]></query>
          <earliest>$time_range.earliest$</earliest>
          <latest>$time_range.latest$</latest>
        </search>
        <option name="charting.chart">area</option>
        <option name="charting.axisY.minimumNumber">0</option>
        <option name="charting.axisY.maximumNumber">100</option>
        <option name="charting.legend.placement">right</option>
      </chart>
    </panel>
  </row>

</dashboard>

```
