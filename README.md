# 🛡️ Building My First SIEM Lab: A Journey into Security Monitoring


---

This project demonstates how I went from reading about security monitoring to actually running a working SIEM that collects real security events from a Windows endpoint. It took a lot of troubleshooting, and more coffee than I'd like to admit. But by the end, I had infrastructure that does exactly what enterprise SOC teams use to detect threats.

**The Goal:** Build a working Wazuh SIEM that monitors a Windows 11 endpoint and collects security events in real-time.

---

## 🏗️ Part 1: Planning the Lab Architecture

Before touching any VMs, I needed a plan. After researching how real SOC environments work, I designed this architecture:

```
┌─────────────────────────────────────────────────────────┐
│                    VMware Workstation                    │
│                                                          │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Wazuh SIEM Server (Ubuntu Server 24.04)         │  │
│  │  IP: 192.168.229.128                             │  │
│  │  RAM: 8GB  │  CPU: 4 cores  │  Disk: 50GB        │  │
│  │                                                   │  │
│  │  Running:                                        │  │
│  │  ├─ Wazuh Manager (detects threats)              │  │
│  │  ├─ Wazuh Indexer (stores logs)                  │  │
│  │  └─ Wazuh Dashboard (web interface)              │  │
│  └──────────────────────────────────────────────────┘  │
│                          ▲                               │
│                          │                               │
│                    Port 1514 (Encrypted)                 │
│                          │                               │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Windows 11 Pro (Monitored Endpoint)             │  │
│  │  IP: 192.168.229.129                             │  │
│  │  RAM: 4GB  │  CPU: 2 cores                       │  │
│  │                                                   │  │
│  │  ├─ Wazuh Agent (sends logs to server)           │  │
│  │  └─ Sysmon (detailed system monitoring)          │  │
│  └──────────────────────────────────────────────────┘  │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

**The idea:** The Wazuh server would act as my centralized monitoring platform, while the Windows 11 VM would be the endpoint I'm monitoring. Every login, every process that starts, every network connection, the SIEM would see it all.

---

## 🖥️ Part 2: Setting Up the Wazuh SIEM Server

### **Creating the Ubuntu Server VM**

First, I needed a solid foundation for the SIEM. Wazuh recommends at least 4GB RAM, but since I wanted good performance and room to expand, I went with 8GB.

![VM Hardware Configuration](https://github.com/lionelmsango/wazuh-siem-lab/blob/9dbc42e0e62860636f4a1132d9bcceda9adec34c/screenshots/09-vm-hardware.jpg)
*Screenshot 1: VM hardware settings*

---

### **Installing Wazuh**

Wazuh offers an "all-in-one" installation script that sets up the Manager, Indexer, and Dashboard in one go. For a home lab, this is perfect.

I downloaded the installation script:

```bash
curl -sO https://packages.wazuh.com/4.9/wazuh-install.sh
chmod +x wazuh-install.sh
```

Then ran it:

```bash
sudo bash wazuh-install.sh -a
```

![Wazuh Installation in Progress](https://github.com/lionelmsango/wazuh-siem-lab/blob/7a9e08703d30aa1fcfd2a558172e75e00f0235ed/screenshots/10-wazuh-installation.jpg)
*Screenshot 2: Wazuh installation assistant running - installing dependencies*

The script started installing packages:

1. **Wazuh Indexer** (based on OpenSearch/Elasticsearch) - for storing logs
2. **Wazuh Manager** - the brain that analyzes security events
3. **Wazuh Dashboard** - the web interface for viewing alerts

The installation finished successfully. The script printed out:
- **Dashboard URL:** `https://192.168.229.128`
- **Username:** `admin`
- **Password:** [randomly generated which i copied and saved immediately]

---

### **Verifying Wazuh Manager is Running**

Before trying the web interface, I wanted to make sure the Wazuh Manager service was actually running.

```bash
sudo systemctl status wazuh-manager
```

![Wazuh Manager Service Status](https://github.com/lionelmsango/wazuh-siem-lab/blob/624f7d88fb60b4e2cad4add519e14b320ecf2eb6/screenshots/12-wazuh-manager-status.jpg)
*Screenshot 3: Wazuh Manager service showing as active (running)*

**Status:** `active (running)` ✅

Perfect. The SIEM server was up and operational. Now to access the dashboard.

---

### **Accessing the Wazuh Dashboard**

I opened Firefox on the Ubuntu machine and navigated to the DHCP-assigned IP address for the ubuntu server:

```
https://192.168.229.128
```

The browser warned about the self-signed certificate (expected for a home lab). I clicked "Accept Risk and Continue."

![Wazuh Login Page](https://github.com/lionelmsango/wazuh-siem-lab/blob/ec51e3048eb82518e15c063e6f4299f032076586/screenshots/01-wazuh-login.jpg)
*Screenshot 4: Wazuh dashboard login page*

The login was succesful and i now had access to the wazuh dashboard.

![Wazuh Dashboard Homepage](https://github.com/lionelmsango/wazuh-siem-lab/blob/ec51e3048eb82518e15c063e6f4299f032076586/screenshots/02-dashboard-overview.jpg)
*Screenshot 5: Wazuh dashboard showing 88 alerts (14 medium severity, 74 low severity)*

The dashboard loaded, showing:
- **Agents Summary:** No agents yet (expected - I haven't connected Windows yet)
- **Last 24 Hours Alerts:** Already showing some alerts from the server itself
- Various security modules: Configuration Assessment, Malware Detection, Threat Hunting

I had a working SIEM dashboard. Now I needed something to monitor.

---

## 💻 Part 3: Setting Up the Windows 11 Endpoint
I created a second VM on VMware with windows OS. After installing Windows 11 and going through the setup, I had a fresh Windows desktop ready to be monitored.

---

### **Installing the Wazuh Agent**

The Wazuh agent is what sends logs from Windows to the SIEM server. I downloaded it from the Wazuh packages repository:

**PowerShell (as Administrator):**

```powershell
Invoke-WebRequest -Uri "https://packages.wazuh.com/4.x/windows/wazuh-agent-4.9.0-1.msi" -OutFile "$env:USERPROFILE\Downloads\wazuh-agent.msi"
```

Then installed it with the server IP:

```powershell
cd $env:USERPROFILE\Downloads
msiexec.exe /i wazuh-agent.msi /q WAZUH_MANAGER="192.168.229.128" WAZUH_AGENT_NAME="WIN11-CLIENT"
```

Started the service:

```powershell
NET START WazuhSvc
```

**Output:**
```
The Wazuh service is starting.
The Wazuh service was started successfully.
```

![Wazuh Agent Started Successfully](https://github.com/lionelmsango/wazuh-siem-lab/blob/a17b39adf4637354e0a7277629a8616356fefb8f/screenshots/12a-wazuh-manager-status.jpg)
*Screenshot 6: PowerShell showing Wazuh agent intallation*

The agent was running. But would it connect to the server? 
I Waited about 30 seconds, then refreshed the wazuh dashboard and clicked on **"Agents"** in the left sidebar.
**Status:** **Active** ✅ (green dot)

![Windows 11 Agent Active in Dashboard](https://github.com/lionelmsango/wazuh-siem-lab/blob/a17b39adf4637354e0a7277629a8616356fefb8f/screenshots/13-agent-active.jpg)
*Screenshot 7: Wazuh dashboard showing WIN11-CLIENT as "active" with green status indicator*

---

## 🔍 Part 4: Adding Enhanced Monitoring with Sysmon

At this point, I had basic Windows event logging working. But I wanted more detailed visibility - specifically process creation, network connections, and file modifications.

The solution was sysmon, a free Windows tool from Microsoft Sysinternals that provides way more detail than standard Windows Event Logs.

---

### **Downloading Sysmon and Configuration**

I needed two things:
1. **Sysmon itself** (from Microsoft)
2. **A good configuration file** (I used SwiftOnSecurity's config - industry standard)

**PowerShell commands:**

```powershell
# Download Sysmon
Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" -OutFile "$env:USERPROFILE\Downloads\Sysmon.zip"

# Extract it
Expand-Archive -Path "$env:USERPROFILE\Downloads\Sysmon.zip" -DestinationPath "$env:USERPROFILE\Downloads\Sysmon" -Force

# Download SwiftOnSecurity configuration
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" -OutFile "$env:USERPROFILE\Downloads\Sysmon\sysmonconfig.xml"
```

![Downloading Sysmon and Configuration](https://github.com/lionelmsango/wazuh-siem-lab/blob/eb6371dc3287cd32335854941292454a9d1b7d0e/screenshots/03-sysmon-download.jpg)
*Screenshot 8: PowerShell showing Sysmon and configuration file downloads*

---

### **Installing Sysmon**

With Sysmon downloaded and the config ready, I installed it:

```powershell
cd $env:USERPROFILE\Downloads\Sysmon
.\Sysmon64.exe -accepteula -i sysmonconfig.xml
```

**Output:**
![Sysmon Installation Successful](https://github.com/lionelmsango/wazuh-siem-lab/blob/eb6371dc3287cd32335854941292454a9d1b7d0e/screenshots/04-sysmon-installation.jpg)
*Screenshot 9: Sysmon installation completing successfully*

**Perfect!** Sysmon was installed and running.

---

### **Verifying Sysmon Service**

To make sure Sysmon was actually running as a service:

```powershell
Get-Service Sysmon64
```

![Sysmon Service Status](https://github.com/lionelmsango/wazuh-siem-lab/blob/eb6371dc3287cd32335854941292454a9d1b7d0e/screenshots/05-sysmon-service.jpg)
*Screenshot 10: PowerShell showing Sysmon64 service running*

**Status:** `Running` ✅

---

### **Checking Sysmon Logs in Event Viewer**

I opened Event Viewer to verify Sysmon was actually logging events:

**Path:** `Applications and Services Logs → Microsoft → Windows → Sysmon → Operational`

![Event Viewer Showing Sysmon Logs](https://github.com/lionelmsango/wazuh-siem-lab/blob/eb6371dc3287cd32335854941292454a9d1b7d0e/screenshots/06-event-viewer-sysmon.jpg)
*Screenshot 11: Windows Event Viewer showing 19 Sysmon operational events*


I clicked on one of the Event ID 1 entries to see the details:

![Sysmon Event ID 1 Details](https://github.com/lionelmsango/wazuh-siem-lab/blob/eb6371dc3287cd32335854941292454a9d1b7d0e/screenshots/07-sysmon-event-id-1.jpg)
*Screenshot 12: Detailed view of Sysmon Event ID 1 (Process Creation) showing full process information*

**This is the kind of detail** that makes threat detection possible. A normal Windows Security log would just say "a process started." Sysmon tells you exactly what process, who started it, when, and with what parameters.

---

### **Configuring Wazuh to Collect Sysmon Logs**

Sysmon was logging to Windows Event Viewer, but Wazuh didn't know to collect these logs yet. I needed to modify the Wazuh agent configuration file.

**File to edit:** `C:\Program Files (x86)\ossec-agent\ossec.conf`

I opened it in Notepad (as Administrator) and added this section:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

![Wazuh ossec.conf with Sysmon Configuration](https://github.com/lionelmsango/wazuh-siem-lab/blob/eb6371dc3287cd32335854941292454a9d1b7d0e/screenshots/08-ossec-conf.jpg)
*Screenshot 13: ossec.conf file showing Sysmon event channel configuration added*

**What this does:** Tells the Wazuh agent to read from the Sysmon event channel and send those logs to the SIEM server.

After saving the file, I restarted the Wazuh agent so it would pick up the new configuration:

```powershell
Restart-Service WazuhSvc
```

---

## 📊 Part 5: Seeing It All Come Together

### **The Moment of Truth**

I went back to the Wazuh Dashboard and clicked on **"Discover"** (the log search interface). I filtered for:

```
agent.name: "WIN11-CLIENT"
```

And watched as logs started flooding in.

---

## 🎓 What I Learned (The Real Value)

### **Technical Skills**

**Linux Administration:**
- Installing packages on Ubuntu Server
- Managing services with systemd
- Reading log files for troubleshooting

- **SIEM Operations:**
- Installing and configuring Wazuh
- Deploying and managing security agents
- Understanding log collection architecture
- Reading and interpreting security alerts
- Knowing the difference between events (raw data) and alerts (detected patterns)

**Windows Security:**
- Understanding Windows Event IDs
- Installing and configuring Sysmon
- Modifying system-level configuration files
- PowerShell for system administration

---

## 🚀 What's Next

This lab is **Phase 1** - the foundation. I now have:
- ✅ Working SIEM infrastructure
- ✅ Monitored Windows endpoint
- ✅ Enhanced logging with Sysmon
- ✅ Real security events being collected

**Future enhancements I'm planning:**

**Short-term:**
- Run attack simulations (brute-force, privilege escalation, malicious PowerShell)
- Create custom alert rules for specific threat patterns
- Practice incident investigation workflows
- Document findings for each attack scenario

**Medium-term:**
- Add a Linux endpoint (Ubuntu workstation) to monitor
- Implement File Integrity Monitoring (FIM) for critical system files
- Set up automated responses to specific threats
- Create custom dashboards for different attack types

**Long-term:**
- Integrate threat intelligence feeds (AlienVault OTX, AbuseIPDB)
- Add network traffic analysis with Suricata IDS
- Build a complete incident response playbook
- Simulate multi-stage attacks (reconnaissance → exploitation → persistence)













