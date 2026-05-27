# ☁️ Azure Infrastructure Monitoring & Alerting Lab

> **Designed and deployed a production-style Azure cloud infrastructure from scratch** — covering VM provisioning, Infrastructure as Code, monitoring, alerting, log analytics, backup, and access control.

---

## 📋 Project Overview

This project demonstrates end-to-end Azure cloud operations skills by building a fully monitored, secured, and backed-up multi-VM environment — similar in scope to enterprise production environments.

| Detail | Value |
|--------|-------|
| **Cloud Platform** | Microsoft Azure (Free Trial) |
| **Region** | Central India |
| **VMs Deployed** | 2 (Ubuntu 22.04 LTS + Windows Server 2022) |
| **Completion Date** | May 2026 |
| **Resource Group** | rg-azure-monitoring-lab |

---

## 🏗️ Architecture

```
Azure Subscription
└── Resource Group: rg-azure-monitoring-lab (Central India)
    ├── vm-linux-lab          (Ubuntu 22.04 LTS, Standard_B2ats)
    │   └── Connected to Log Analytics via Azure Monitor Agent
    ├── vm-windows-lab        (Windows Server 2022 Datacenter, Standard_B2ats)
    │   └── Connected to Log Analytics via Azure Monitor Agent
    ├── law-monitoring-lab    (Log Analytics Workspace — Central India)
    │   ├── 1 Linux computer connected ✅
    │   └── 1 Windows computer connected ✅
    ├── rsv-lab-backup        (Recovery Services Vault)
    │   └── bp-daily-7day-retention
    │       ├── vm-linux-lab  (Daily @ 2AM UTC, 7-day retention)
    │       └── vm-windows-lab (Daily @ 2AM UTC, 7-day retention)
    ├── Azure Monitor
    │   ├── alert-linux-high-cpu        (CPU > 80%, Severity 2 - Warning)
    │   ├── alert-vm-stopped            (VM power off, Severity 4 - Verbose, both VMs)
    │   └── alert-windows-low-memory   (Available Memory < 500MB, Severity 2 - Warning)
    ├── Action Group: ag-lab-notifications (Email alerts)
    └── RBAC — rg-azure-monitoring-lab IAM
        ├── Prajwal Sharma → Owner (Subscription Inherited)
        └── Test Viewer    → Reader (This resource)
```

---

## ⚙️ Components Built

| Component | Azure Service | What It Demonstrates |
|-----------|--------------|----------------------|
| 2 Virtual Machines | Azure VMs (Standard_B2ats) | VM provisioning, OS administration |
| Infrastructure as Code | ARM Templates (with tags) | IaC deployment, version control |
| VM Health Monitoring | Azure Monitor — VM Monitor tab | Proactive monitoring, metrics |
| Alert Rules (3) | Azure Monitor Alerts + Action Groups | Automated alerting, incident response |
| Centralised Logging | Log Analytics Workspace + KQL | Observability, log analysis |
| Agent-based Monitoring | Azure Monitor Agent + Data Collection Rules | Modern AMA-based monitoring |
| Automated Backup | Azure Backup + Recovery Services Vault | Backup policy, disaster recovery |
| Access Control | Azure RBAC (Owner + Reader) | Security, least-privilege access |
| Ops Dashboard | Azure Dashboard — Ops View | Real-time visibility, reporting |
| Activity Logs | VM Activity Log | Audit trail, operational history |

---

## 🖥️ Virtual Machines

### vm-linux-lab
- **OS:** Ubuntu Server 22.04.5 LTS (GNU/Linux 6.8.0-1052-azure x86_64)
- **Size:** Standard_B2ats_v2
- **Region:** Central India
- **Public IP:** 20.244.24.225
- **Access:** SSH via Windows Command Prompt (`ssh azureadmin@20.244.24.225`)
- **Verified commands:** `df -h` (6.5% disk used, 28.9GB available), `free -m` (899MB total, 509MB available)

### vm-windows-lab
- **OS:** Windows Server 2022 Datacenter
- **Size:** Standard_B2ats_v2
- **Region:** Central India
- **Public IP:** 20.244.27.109
- **Access:** RDP (`mstsc` → public IP)
- **Verified:** Windows Server 2022 desktop confirmed via RDP

Both VMs deployed to **same Virtual Network** for internal communication.

---

## 📄 Infrastructure as Code — ARM Template

Full ARM template at [`arm-templates/template.json`](./arm-templates/template.json).

The template defines all resources: both VMs, VNet, subnets, NSGs, public IPs, and storage — exported directly from the Azure Resource Group.

**Resource tags added for governance:**
```json
"tags": {
    "environment": "lab",
    "owner": "prajwal-sharma",
    "project": "azure-monitoring-lab",
    "costCenter": "personal-learning"
}
```

**Parameters defined in template:**
- `virtualMachines_vm_linux_lab_name` → vm-linux-lab
- `virtualMachines_vm_windows_lab_name` → vm-windows-lab
- `publicIPAddresses_pip_linux_lab_name` → pip-linux-lab

---

## 📊 Azure Monitor & Alerting

### Alert Rules

| Alert Name | Condition | Severity | Target | Signal Type | Status |
|-----------|-----------|----------|--------|-------------|--------|
| alert-linux-high-cpu | Percentage CPU > 80 | 2 - Warning | vm-linux-lab | Metrics | ✅ Enabled |
| alert-vm-stopped | Category=Administrative, Operation=Power Off | 4 - Verbose | vm-linux-lab, vm-windows-lab | Activity log | ✅ Enabled |
| alert-windows-low-memory | Available Memory Bytes < 500000000 | 2 - Warning | vm-windows-lab | Metrics | ✅ Enabled |

### Action Group
- **Name:** ag-lab-notifications
- **Notification:** Email alert on trigger
- **Covers:** All 3 alert rules

---

## 📈 Log Analytics Workspace & KQL Queries

**Workspace:** law-monitoring-lab (Central India)
**Connection method:** Azure Monitor Agent (AMA) — modern DCR-based approach
**Connected:** ✅ 1 Linux computer + ✅ 1 Windows computer

### KQL Queries

**1. VM Heartbeat — confirmed both VMs alive and connected**
```kql
Heartbeat
| where TimeGenerated > ago(1h)
| summarize LastHeartbeat = max(TimeGenerated) by Computer
| order by LastHeartbeat desc
```
> **Result:** vm-windows-lab: 27/5/2026 8:18:56 PM | vm-linux-lab: 27/5/2026 8:18:37 PM ✅

**2. CPU Usage Over Time**
```kql
Perf
| where ObjectName == "Processor" and CounterName == "% Processor Time"
| where TimeGenerated > ago(1h)
| summarize AvgCPU = avg(CounterValue) by bin(TimeGenerated, 5m), Computer
| render timechart
```

**3. Failed Login Attempts (Security)**
```kql
SecurityEvent
| where EventID == 4625
| where TimeGenerated > ago(24h)
| summarize FailedLogins = count() by Computer, Account
| order by FailedLogins desc
```

**4. Available Memory**
```kql
Perf
| where ObjectName == "Memory" and CounterName == "Available MBytes"
| where TimeGenerated > ago(1h)
| summarize AvgMemoryMB = avg(CounterValue) by Computer
```

**5. Top Events by Volume**
```kql
Event
| where TimeGenerated > ago(24h)
| summarize EventCount = count() by EventLevelName, Computer
| top 10 by EventCount desc
```

---

## 💾 Azure Backup Configuration

**Recovery Services Vault:** rsv-lab-backup (Central India)
**Policy Name:** bp-daily-7day-retention

| Setting | Value |
|---------|-------|
| Frequency | Daily |
| Time | 2:00 AM UTC |
| Instant restore snapshots | 2 days |
| Daily retention | 7 days |
| Weekly retention | 4 weeks (every Sunday @ 2AM) |
| Monthly/Yearly | Not configured |
| VMs protected | vm-linux-lab + vm-windows-lab |

---

## 🔐 RBAC — Access Control (IAM)

Role assignments on **rg-azure-monitoring-lab**:

| User | Type | Role | Scope |
|------|------|------|-------|
| Prajwal Sharma | User | Owner | Subscription (Inherited) |
| Test Viewer | User | Reader | This resource |

Total role assignments: **2 of 4000** available.

Demonstrates **least-privilege access model** — separating admin (Owner) from read-only (Reader) access, a standard enterprise security practice.

---

## 🗂️ Activity Logs Captured

**vm-linux-lab activity log (last week):**
- Deallocate Virtual Machine — Succeeded
- Start Virtual Machine — Succeeded
- Health Event Resolved
- Health Event Updated
- audit Policy action — Succeeded

**vm-windows-lab activity log (last week):**
- Deallocate Virtual Machine — Succeeded
- audit Policy action — Succeeded
- Health Event Resolved
- Health Event Updated

---

## 📊 Ops Dashboard

**Dashboard:** Azure Monitoring Lab — Ops View (Private dashboard)

Tiles configured:
- CPU Credits Consumed — vm-windows-lab (real-time chart)
- Available Memory Bytes — vm-linux-lab (476.84 MiB avg)
- All Resources tile (shows: rsv-lab-backup, law-monitoring-lab, vm-linux-lab, vm-windows-lab, alert rules)
- Resource Groups tile

---

## 🗂️ Repository Structure

```
azure-monitoring-lab/
├── arm-templates/
│   └── template.json          # Full ARM template for all resources
├── kql-queries/
│   ├── heartbeat.kql
│   ├── cpu-usage.kql
│   ├── failed-logins.kql
│   ├── memory-usage.kql
│   └── top-events.kql
├── screenshots/               # Portal screenshots (18 total)
└── README.md
```

---

## 🔑 Key Learnings

1. **Region availability matters** — B1s showed `NotAvailableForSubscription` in East US for free trial; switching to Central India resolved it immediately
2. **Azure Monitor Agent (AMA) is the modern standard** — the legacy Diagnostics Extension is deprecated (March 2026); DCR-based AMA is the current Microsoft-recommended approach
3. **VM Insights auto-configures everything** — enabling monitoring via the VM Monitor tab automatically creates the workspace, installs agents, and sets up Data Collection Rules in one step
4. **Data ingestion has a delay** — after connecting agents, KQL query results take 10–30 minutes to appear; this is normal production behaviour
5. **ARM templates export as single combined files** — Azure now embeds parameters directly in template.json; this is the modern clean format
6. **Resource tags are governance tools** — adding environment/owner/project tags to ARM templates mirrors real-world enterprise billing and ownership tracking

---

## 🛠️ Technologies Used

![Azure](https://img.shields.io/badge/Microsoft_Azure-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu_22.04-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![Windows Server](https://img.shields.io/badge/Windows_Server_2022-0078D6?style=for-the-badge&logo=windows&logoColor=white)
![GitHub](https://img.shields.io/badge/GitHub-181717?style=for-the-badge&logo=github&logoColor=white)

**Azure Services used:**
Azure Monitor · Azure Alerts · Log Analytics Workspace · KQL · Azure Backup · Azure RBAC · ARM Templates · Data Collection Rules · Azure Monitor Agent · Recovery Services Vault · Azure Dashboard · Microsoft Entra ID · Activity Logs · VM Insights

---

## 👤 Author

**Prajwal Sharma**
Database & Cloud Operations Engineer | Oracle DBA | Azure Administrator (AZ-104)
📧 prajwalsharma88@gmail.com
🔗 [linkedin.com/in/prajwal-sharma88](https://linkedin.com/in/prajwal-sharma88)

---

*Built as a hands-on Azure lab to demonstrate cloud infrastructure and operations skills for Azure / Cloud Operations Engineer roles. All screenshots taken from actual Azure Portal deployments.*
