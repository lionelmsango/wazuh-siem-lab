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




