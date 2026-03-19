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



