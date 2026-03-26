# 🐧 Linux Cloud Operations Cheatsheet

**Real-world commands used daily in production Cloud Operations — supporting enterprise SaaS platforms on Azure & AWS.**

> Built from 3+ years of 24×7 production operations, incident response, and troubleshooting across PLM and IIoT platforms.

---

## 📋 Table of Contents

- [System Health & Resource Monitoring](#-system-health--resource-monitoring)
- [Disk Space Management](#-disk-space-management)
- [Log Management & Analysis](#-log-management--analysis)
- [Process Management](#-process-management)
- [Service Management](#-service-management)
- [Cache & Memory Management](#-cache--memory-management)
- [Java Application Troubleshooting](#-java-application-troubleshooting)
- [Network & Connectivity](#-network--connectivity)
- [File Operations](#-file-operations)
- [Zabbix Agent Operations](#-zabbix-agent-operations)
- [Incident Response Workflow](#-incident-response-workflow)

---

## 🖥️ System Health & Resource Monitoring

```bash
# Check memory usage (human readable)
free -mh

# Real-time process monitoring sorted by memory usage
top -c

# Check last 10/20 login sessions (useful for audit trail)
last -10
last -20

# List running Java processes
jps

# Check system hostname
hostname
```

---

## 💾 Disk Space Management

```bash
# Check disk space across all volumes
df -kh
df -h

# Find top 10 largest directories/files in current path
du -sh * | sort -rh | head -10

# View root directory disk usage — top 20 largest items
sudo du -ahx / | sort -rh | head -n 20

# Check which directories are using GB+ space
du -sh * | grep G

# Clean yum cache (when root volume is full)
yum clean all

# Clear temp directories safely
cd /tmp
rm -rf *
```

### ⚠️ Disk Space Alert Response Workflow

```
1. SSH into server → sudo su
2. Run: df -kh (identify which volume is full)
3. Navigate to the full volume
4. Run: du -sh * | sort -rh | head -10 (find what's consuming space)
5. Compress old logs with gzip (preferred over deletion)
6. If root volume < 15% free → try yum clean all
7. If still not resolved → escalate to Infrastructure team
```

---

## 📝 Log Management & Analysis

### Viewing Logs

```bash
# List logs sorted by time (newest last)
ls -ltr

# Count log files for a specific date
ll * | grep 'Feb 25' | wc -l

# View a log file
vi <logfile>

# Search pattern inside vi editor
/lock=java        # search forward for java locks
:q                # quit vi without saving
```

### Compressing Old Logs (Preferred over deletion)

```bash
# Gzip specific log files by date pattern
gzip *250114*

# Compress logs within a date range
find . -type f -name "*.log" -newermt "2026-01-24" ! -newermt "2026-01-30" -exec gzip {} \;

# Compress first N log files (e.g., 300 out of 1000)
ls MethodServer-*.log.2025-12-17_* | head -n 300 | xargs -I{} gzip {}

# Compress with progress feedback
ls MethodServer-*.log.2025-05-26_* | head -n 200 | while read -r file; do
  echo "Compressing: $file"
  gzip "$file"
done
echo "Done compressing files."

# Compress logs for specific dates (safe — skips already gzipped)
for i in *2026-03-25*; do
  if [ -f "$i" ] && [ ! -f "$i.gz" ]; then
    gzip "$i"
  fi
done
```

### Moving Logs to Backup

```bash
# Move old logs to backup directory
mv *log4j.log.2025-05-30* *log4j.log.2025-05-31* /backup/
```

### Log Spamming Fix

```bash
# Reset all loggers to default level (stops DEBUG log flooding)
windchill wt.util.jmx.SetLogLevel -resetAll

# Check for DEBUG level logs on a specific date
grep "2026-03-02" * | grep "DEBUG"

# Verify log count after reset (should show 0 new logs)
ll MethodServer-* | grep 'Feb 25' | wc -l
```

---

## ⚙️ Process Management

```bash
# Find specific processes
ps aux | grep tomcat
ps -ef | grep -i Tomcat
ps -ef | grep solr
ps -ef | grep index
ps -ef | grep zabbix
ps aux | grep firefox

# Kill a process by PID
kill -9 <PID>

# Kill all Firefox processes (should not run on prod)
pkill -15 firefox

# Verify process is killed
ps aux | grep firefox
```

---

## 🔄 Service Management

```bash
# Restart Zabbix agent
systemctl restart zabbix-agent

# Restart CylanceSVC (security agent)
systemctl restart cylancesvc

# Start Apache Tomcat
cd /path/to/tomcat/bin
./startup.sh start

# Check hosts file (find server mappings)
cat /etc/hosts

# Switch to application admin users
sudo su wcadmin    # Windchill admin
sudo su twadmin    # ThingWorx admin
```

---

## 🧹 Cache & Memory Management

```bash
# Simple cache clear
free -mh
sync
echo 3 > /proc/sys/vm/drop_caches
free -mh

# Comprehensive cache clear with before/after comparison
sudo bash -c '
  echo "Memory usage before clearing caches:"
  free -mh
  echo -e "\nTop 5 memory-consuming processes:"
  top -b -o %MEM | head -n 12
  sync
  echo 3 > /proc/sys/vm/drop_caches
  echo -e "\nMemory usage after clearing caches:"
  free -mh
  echo -e "\nCache cleared successfully!"
'
```

---

## ☕ Java Application Troubleshooting

### Deadlock Detection

```bash
# Search for deadlocks in all log files
grep -iE 'deadlock|java lock' *.log

# Search for deadlocks in BackgroundMethodServer logs
grep -iE 'deadlock|BackgroundMethodServer' *.log

# Find which log files contain deadlocks (specific date)
grep -E -l 'deadlock|java lock' *2025-06-09*

# Show filename with matches
grep -H -E 'deadlock|java lock' *2025-05-29*

# Search for java locks in specific log file
cat MethodServer-*-log4j.log | grep -i deadlock

# Search for lock patterns
grep -l lock=java *2502092351*
```

### Java Heap / Memory Issues

```
Workflow:
1. Run: top -c (check which Java process is consuming memory)
2. List recent logs: ls -ltr
3. Open the latest log file: vi <latest-log>
4. Search for locks: /lock=java
5. If pattern found → reassign to Incident team
   Note: "Java locks consuming memory, please investigate"
6. If pattern not found → exit with :q
```

### Clearing Application Cache (ThingWorx)

```bash
# Clear ThingWorx catalina.out (truncate without deleting)
> catalina.out
```

---

## 🌐 Network & Connectivity

```bash
# Check hosts file for server name resolution
cat /etc/hosts

# Edit hosts file
vi /etc/hosts

# SSH tunnel for RDP access (Windows servers)
# Set up SSH tunnel: hostname:3389 → local port
# Then connect via RDP to localhost:<local-port>
```

### Identifying Server Environment

```
AWS servers  → 4-word hostname pattern
Azure servers → 2-word hostname pattern

Key environments:
- QA and PRD → critical, wait for auto-resolve before escalating
- PRD (Production) → highest priority
```

---

## 📁 File Operations

```bash
# Find files or folders by name
find . -name 'logs'

# Change file permissions
chmod 777 <filename>

# List files with details
ll
ls -ltr

# Remove files (use with caution in production!)
rm -rf <directory>/*

# Always prefer gzip over rm for logs
```

---

## 🔔 Zabbix Agent Operations

```bash
# Check if Zabbix agent is running
ps -ef | grep zabbix

# Restart Zabbix agent (fixes most Zabbix agent alerts)
systemctl restart zabbix-agent
```

---

## 🚨 Incident Response Workflow

### When an Alert Fires (PagerDuty/Zabbix)

```
Step 1: Determine if True or False alert
  ├── Check work emails (any planned changes?)
  ├── Check team channels (patches/maintenance?)
  ├── Check Slack
  └── Check recent logins: last -10

Step 2: Initial Investigation
  ├── SSH into the server
  ├── sudo su (root access)
  ├── Check system health: top -c, free -mh, df -kh
  └── Check application: jps, ps aux | grep <service>

Step 3: Based on Alert Type
  ├── Memory alert → Cache clear, check Java processes
  ├── Disk alert → Find large files, gzip old logs
  ├── Service down → Check process, restart if needed
  ├── Certificate expiry → Create ticket for App team
  └── Java deadlock → Check logs, escalate to Incident team

Step 4: Resolution
  ├── If resolved → Document in ServiceNow, close PagerDuty
  ├── If not resolved → Escalate to appropriate team
  └── Create Salesforce case if customer-impacting
```

### Certificate Expiry Handling

```
1-7 days to expiry   → URGENT: Create Incident ticket (Cloud Incident PLM)
10-30 days to expiry  → Create Service Request for Application team
30-360 days to expiry → Create Service Request for Application team (planned)
```

### Escalation Matrix

```
Issue Type              → Escalate To
─────────────────────────────────────────
Java locks/deadlocks    → Incident Team
Certificate renewals    → Application Team (SSL, ThingWorx, PingFederate)
Disk < 15% on root      → Infrastructure Team
Black screen on server  → Infrastructure Team
Database issues         → Database Team
Application errors      → Application Team
```

---

## 🏗️ Environment Reference

### Key Services to Monitor

| Service | Check Command | Start Command |
|---------|--------------|---------------|
| Tomcat | `ps -ef \| grep -i Tomcat` | `./startup.sh start` |
| Solr | `ps -ef \| grep solr` | Navigate to SolrServer/bin |
| Zabbix Agent | `ps -ef \| grep zabbix` | `systemctl restart zabbix-agent` |
| CylanceSVC | - | `systemctl restart cylancesvc` |
| Java/JVM | `jps` | Varies by application |

### Useful Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `Ctrl + C` | Cancel/exit current command |
| `Ctrl + D` | Exit from current user/shell |
| `:q` | Exit vi without saving |
| `:wq` | Save and exit vi |
| `/pattern` | Search forward in vi |

---

## ☸️ Kubernetes (AKS) Operations

### Connecting to AKS Cluster

```bash
# Login to Azure
az login --use-device-code

# Set subscription
az account set --subscription <subscription-name>

# Get cluster credentials
az aks get-credentials --name <cluster-name> --resource-group <resource-group> --overwrite-existing --admin
```

### Pod & Namespace Operations

```bash
# List all pods across all namespaces
kubectl get pods --all-namespaces

# List pods in a specific namespace
kubectl get pods -n <namespace>

# Check pod logs
kubectl logs <pod-name> -n <namespace>

# Describe a pod (events, status, restarts)
kubectl describe pod <pod-name> -n <namespace>

# Get events sorted by time
kubectl get events --sort-by=.lastTimestamp -n <namespace>

# Check node status
kubectl get nodes

# Check pod restarts (CrashLoopBackOff, OOMKilled)
kubectl get pods -n <namespace> | grep -E 'Error|CrashLoop|OOM'
```

---

## 🔧 Monitoring Tools Installation & Validation

### Zabbix Agent

```bash
# Install Zabbix agent
salt-call state.sls zabbix.zabbix-agent-new

# Check Zabbix agent status
systemctl status zabbix-agent

# Restart Zabbix agent (fixes most agent alerts)
systemctl restart zabbix-agent

# Remove Zabbix agent
yum remove zabbix-agent -y
```

### Sumo Logic Collector

```bash
# Install Sumo Logic
salt-call state.apply sumologic.init

# Check Sumo Logic collector status
./collector status
```

### Post Go-Live Monitoring Checklist

```
Tool          Validation Step
────────────────────────────────────────────────
Sumo Logic    Verify installation and collector status
PagerDuty     Confirm service entry in Service Directory
Zabbix        Ensure host appears and is reporting metrics
Catchpoint    Obtain URL from ticket owner, validate setup
```

### Environment Monitoring Matrix

```
Environment     Zabbix    PagerDuty    Sumo Logic    Catchpoint
──────────────────────────────────────────────────────────────
Production       ✅         ✅            ✅             ✅
QA               ✅         ✅            ⚠️ Optional    ❌
Dev/Test         ✅         ✅            ⚠️ Optional    ❌
Staging          ✅         ✅            ⚠️ Optional    ❌
Integration      ✅         ✅            ⚠️ Optional    ❌
```

---

## 🔑 File Ownership & Permissions

```bash
# Change file ownership
chown <user>:<group> <filename>

# Change ownership recursively
chown -R <user>:<group> <directory>

# Change file permissions
chmod 777 <filename>    # Full access (use cautiously!)
chmod 755 <filename>    # Owner full, others read+execute
chmod 644 <filename>    # Owner read+write, others read only
```

---

## 🌐 LDAP & Network Checks

```bash
# Check if LDAP ports are listening
netstat -tunlp | grep -Ei '(:1389|:636)'

# Alternative using ss
ss -tunlp | grep -Ei '(:1389|:636)'
```

---

## ⏰ Crontab Operations

```bash
# List all cron jobs
crontab -l

# Search for specific cron job
crontab -l | grep <job-id>

# Edit crontab
crontab -e

# Inside vi editor:
# /search-term     → find the job
# i                → enter insert mode
# Make changes (comment with # or uncomment by removing #)
# :wq!             → save and exit

# Cron schedule format:
# ┌───── minute (0-59)
# │ ┌───── hour (0-23)
# │ │ ┌───── day of month (1-31)
# │ │ │ ┌───── month (1-12)
# │ │ │ │ ┌───── day of week (0-7, 0=Sun)
# * * * * * command
```

### Interface Monitoring via Crontab

```
Comment (#) = Disable monitoring
Uncomment (remove #) = Enable monitoring

Use case: Temporarily disable monitoring during maintenance
→ Comment out the cron job
→ Perform maintenance
→ Uncomment to re-enable
```

---

## 🖥️ Windows Server Operations

### Connecting to Windows Servers

```
1. Set up SSH tunnel for RDP access
2. Configure tunnel: <hostname>:3389
3. Set local source port (any 4-digit number)
4. Connect via Remote Desktop to localhost:<port>
```

### Common Windows Checks

```
- Check C: and D: drive disk space
- Verify application services are running
- Check Worker Agent Administration status
- Restart workers if needed via Worker Agent Admin UI
```

### Worker Troubleshooting

```
1. Check Worker Agent Administration portal
2. Click green flag to start stopped workers
3. If worker won't start:
   a. Navigate to application directory on D: drive
   b. Clear .wf and .wvs folders
   c. Delete .crc files
   d. Run purge.bat
   e. Open command prompt in x86_64/obj folder
   f. Run: WorkerDaemon.exe 601
   g. Verify worker started in Worker Agent Admin
```

---

## 📖 Salesforce Case Creation Workflow

```
1. Go to Salesforce → Create Case
2. Enter SCN Number (from customer name in Salesforce)
3. Add details:
   - Alert Received: Yes
   - Monitoring Tool: Zabbix (or appropriate tool)
4. Add entitlement
5. Change owner:
   - For application issues → Cloud Service Request (Application)
   - For customer-reported incidents → Cloud Incident PLM
6. Tick "Send notification email"
7. Monitor and manually resolve when done
```

---

## 💡 Pro Tips from Production

> **Always gzip, never delete** — Compressed logs can be reviewed later for RCA. Deleted logs are gone forever.

> **Check for planned changes first** — Before investigating an alert, check emails, Slack, and team channels. 50% of alerts during maintenance windows are expected.

> **QA and PRD alerts** — Wait for auto-resolve first, then escalate to Incident team if persistent.

> **Document everything** — Every action taken during an incident should be noted in ServiceNow. Future you (or your teammate) will thank you.

> **When in doubt, escalate** — It's better to escalate early and be wrong than to sit on a Sev0 for 30 minutes trying to fix it alone.

---

## 📦 File Backup & Archival Operations

### Moving Files to Backup (Date Range Pattern)

```bash
# Create backup directory
mkdir -p /backup/tmp/PAA

# Preview files before moving (always do this first!)
ls *_250921*

# Move files for specific dates
mv *_250921* /backup/tmp/PAA/

# Move files for date range using brace expansion
mv *_{0922..0925}* /backup/tmp/PAA/

# Verify files moved successfully
ls /backup/tmp/PAA/
```

### Archiving Old HTTP Server Logs

```bash
# Navigate to logs directory
cd /path/to/HTTPServer/logs

# Preview files before archival
ls mod_jk.log_2023* mod_jk.log_2024* error.log_2023* error.log_2024*

# Create backup dir and zip logs in one command
mkdir -p /backup/HTTPServer && zip /backup/HTTPServer/logs_2023_to_2024.zip \
  mod_jk.log_2023* mod_jk.log_2024* \
  error.log_2023* error.log_2024* \
  ssl_request.log_2023* ssl_request.log_2024* \
  access.log_2023*.gz access.log_2024*.gz

# Verify zip contents
unzip -l /backup/HTTPServer/logs_2023_to_2024.zip

# Alternative: move instead of zip
mkdir -p /backup/HTTPServer && mv mod_jk.log_2023* mod_jk.log_2024* /backup/HTTPServer/
```

---

## 🔐 SSL Certificate Operations

### Check Certificate Details

```bash
# View certificate details (expiry date, issuer, subject)
openssl x509 -in /path/to/certificate.crt -text -noout

# Find all certificate files on the system
find /appl -type f -name "*.crt"

# Check Java keystore certificates (jssecacerts)
keytool -list -v -keystore /path/to/jre/lib/security/jssecacerts -storepass <password>

# Check Java truststore certificates (cacerts)
keytool -list -v -keystore /path/to/jre/lib/security/cacerts -storepass <password>
```

### Certificate Expiry Response

```
Expiry Timeline          Action Required
──────────────────────────────────────────────
1-7 days                 URGENT → Incident ticket immediately
7-10 days                HIGH → Incident Team (Cloud Incident PLM)
10-30 days               MEDIUM → Application Team (Service Request)
30-360 days              LOW → Application Team (planned renewal)
```

---

## 🏭 Application Configuration & Properties

### Check Application Memory Settings

```bash
# View heap and memory configuration from properties file
grep -E '^(wt\.manager\.maxHeap|wt\.method\.maxHeap|wt\.method\.bg\.maxHeap)' \
  /path/to/codebase/wt.properties

# Check monitored services configuration
grep -E '^(wt\.manager\.monitor\.services|wt\.manager\.monitor\.start)' \
  /path/to/codebase/wt.properties
```

### Application Shell & Log Management

```bash
# Enter application shell
cd /path/to/application/bin
./windchill shell

# Reset all log levels (fix log spamming)
windchill wt.util.jmx.SetLogLevel -resetAll

# Count logs for a specific date
ll MethodServer-* | grep 'Mar  2' | wc -l

# Check for DEBUG level logging (should be minimal in production)
grep DEBUG *
grep "2026-03-05" * | grep "DEBUG"
```

---

## 🤝 About

Built by [Dinesh Ravichandiran](https://linkedin.com/in/dineshravichandiran) — Cloud Operations Engineer supporting Fortune 500 PLM & IIoT platforms on Azure & AWS.

- 📧 mailtodinesh0808@gmail.com
- 💼 [LinkedIn](https://linkedin.com/in/dineshravichandiran)
- 🌐 [Portfolio](https://dinesh-cloudops.netlify.app)

---

*"Preparation beats panic, every time." — 3+ years of on-call rotations* 🚀
