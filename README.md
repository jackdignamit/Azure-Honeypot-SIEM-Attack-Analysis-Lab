# 🌐 Azure Honeypot & SIEM Attack Analysis Home Lab
This project walks through how I built a cloud-based cybersecurity home lab using **Microsoft Sentinel** with Azure virtual machines to detect and display attack data. 

Inspired by [Josh Madakor's tutorial](https://youtu.be/g5JL2RIbThM?si=Mmk_O8heRLS_-SU1), I followed along and then expanded on it with hands-on exploration and insights.

---

# Project Overview

This project showcases how I established a cloud-based honeypot VM in Azure and sent brute-force attempt logs to a SIEM workspace. 
The workspace was integrated into Microsoft Sentinel to identify failed login attempts, conduct geolocation IP filtering, and visualize source locations on a live attack map. 
I wrote KQL queries for event filtering, analyzing, and alert creation; acquiring hands-on experience in SIEM operations, log analytics, and security monitoring.

## Tech Stack

| Category | Tool / Platform |
|-----------|----------------|
| Cloud Platform | Microsoft Azure |
| SIEM | Microsoft Sentinel |
| Data Ingestion | Log Analytics Workspace |
| OS | Windows Server 2022 |
| Scripting | PowerShell |
| Query Language | KQL (Kusto Query Language) |
| API | IP-API.com (Geolocation) |

---

# Architecture Diagram

## INSERT IMAGE HERE SOON!

  1) Internet Attackers discover and attempt to login to Azure VM Honeypot
  2) Failed login attempts are captured in Windows Event Logs
  3) Windows Event Logs are funneled into Log Analytics Workspace
  4) Microsoft Sentinel with KQL Filters use the workspace to create an attack map visualization

---

# Step-by-Step Setup

### 1️⃣ Create an Azure Environment
1. Begin with a **free Azure subscription** and create a **resource group** for easy cleanup. Put all resources here.
2. Create a virtual network and network security group, naming them logically (e.g. "RG-SOC-Lab". "Vnet-SOC-Lab", "LAW-Sentinel")
  <img width="1833" height="705" alt="image" src="https://github.com/user-attachments/assets/42e2f927-ebd6-4a87-ad2f-364a901bc617" />
  
---

### 2️⃣ Setup and deploy the Honeypot VM
1. Create a **Windows 10 Pro Server VM** (2 vcpus, 8 GiB memory).
2. **SETUP A STRONG PASSWORD.** Use letters (uppercase/lowercase) with numbers, symbols, and minimum 10-15 characters.
3. Configure an **inbound port rule** in the VM's network security group (NSG) that exposes **RDP (port 3389)** and **SSH (port 22)** to the internet.

  <img width="2117" height="701" alt="image" src="https://github.com/user-attachments/assets/fc6ea4bc-10b2-492c-8b10-f71073122189" />

4. Login and disable Windows Defender Firewall **(start -> wf.msc -> properties -> all off)**

  <img width="890" height="497" alt="image" src="https://github.com/user-attachments/assets/9928fefc-cdcd-40d0-89af-0113444d1348" />

  _(This intentionally exposes the VM to attract brute-force attempts. This is safe in isolation but visible to attackers scanning open ports.)_

**Wait 15-30 minutes or more to allow attackers to trigger event logs.**

---

### 3️⃣ Observe Local Logs
Open **Event Viewer -> Windows Logs -> Security** to view incoming failed login attempts **(Event ID 4625)**. Here you can view attempted usernames, IPs, and timestamps.

<img width="2027" height="1045" alt="image" src="https://github.com/user-attachments/assets/d45f99df-15b1-431f-8bfb-2ae1d7db8226" />
<img width="352" height="95" alt="image" src="https://github.com/user-attachments/assets/cc2ab993-e3b5-4039-9012-eb06fd363875" />

---

### 4️⃣ Create a Log Analytics Workspace and link to Microsoft Sentinel
1. Create both a new **Log Analytics Workspace (LAW)** and **Microsoft Sentinel** workspace under your resource group.

   _The Log Analytics Workspace will act as a centralized storage and analysis hub for incoming logs from your VM. Microsoft Sentinel adds threat detection, alerting, and visualization capabilities. Feel free to explore its built-in workbooks and analytic rules._
   
2. On Microsoft Sentinel, **add the previously created Log Analytics Workspace.** This links the LAW to its respective SIEM.
3. Under **Content Management > Content Hub** search for and install Windows Security Events.
<img width="1398" height="741" alt="image" src="https://github.com/user-attachments/assets/46f6791d-e4dd-4eea-96fb-58df9d3410f9" />

4. Under Windows Security Events, click on "manage" and install **"Windows Security Events via AMA"**.

_Azure Monitoring Agent (AMA) allows all security events from the Windows machine to stream directly into the Microsoft Sentinel workspace._

5. Once installed, open connector page and **create a new data collection rule**. Select your VM under the resources tab and collect **All Security Events**. It may take a moment for the extension to be applied to the VM.

---

### 5️⃣ Query for logs within the LAW
1. Observe some of the ingested logs using KQL filters. To start, input **"SecurityEvent"** in a new query and press run. It will output all security related logs.
<img width="1760" height="1088" alt="Screenshot 2025-10-03 154830" src="https://github.com/user-attachments/assets/8b94f1cb-2230-4abc-ac2d-430aaff2b521" />

2. To filter for attacker's failed login alerts, set **| where EventID == 4625**. You can further filter using Account, TimeGenerated, Computer, Activity, and IpAddress information using a pipe under "project".
_The pipe passes the result of one command as input to the next command._

**SecurityEvent   
    | where EventId == 4625**  

<img width="1740" height="1074" alt="Screenshot 2025-10-03 160646" src="https://github.com/user-attachments/assets/ee578e70-a62c-47b0-b485-c634de4c44e9" />

---

### 6️⃣ Enrich Logs with Geolocation Data
1. The LAW does not include the table option to filter by geolocation. The table must be implemented into Sentinel first. To do this, download this spreadsheet from Google Drive: [geoip-summarized.csv](https://drive.google.com/file/d/13EfjM_4BohrmaxqXZLB5VUBIz2sv9Siz/view)
  
     _alternative link: https://raw.githubusercontent.com/joshmadakor1/lognpacific-public/refs/heads/main/misc/geoip-summarized.csv_

2. Within Sentinel, create the watchlist titled **"geoip"** as a local file with the search key **"network"** and upload the above .csv spreadsheet.
3. Using KQL, you can now filter utilizing the watchlist table:

**let GeoIPDB_FULL = _GetWatchlist("geoip");**  
**let WindowsEvents = SecurityEvent    
    | where IpAddress == (attacker IP address)  
    | where EventID == 4625  
    | order by TimeGenerated desc  
    | evaluate ipv4_lookup(GeoIPDB_FULL, IpAddress, network);**  
**WindowsEvents**

_(insert a random attacker IP address)_

<img width="1745" height="1066" alt="Screenshot 2025-10-03 161308" src="https://github.com/user-attachments/assets/3675d345-7197-4f44-813e-ce3c20a7c6a9" />

---

### 7️⃣ Build the Attack Map in Sentinel
1. On a new tab, go to Sentinel and create a new workbook that will be our attack map.
2. Add a query and paste [this JSON code](https://drive.google.com/file/d/1ErlVEK5cQjpGyOcu4T02xYy7F31dWuir/view?usp=drive_link) into the advanced editor.
3. You can freely adjust colors, bubble sizes, etc under **map settings** in the edit query tab.
      
<img width="2492" height="1185" alt="Screenshot 2025-10-03 161740" src="https://github.com/user-attachments/assets/aa9130d9-a3d1-400d-9452-4a95026fbb21" />
<img width="267" height="67" alt="image" src="https://github.com/user-attachments/assets/c1c8dcee-7cce-4281-b79f-324f5b1ab027" />

_Attacks from a 20 minute period_

---

## Future Improvements

---

## Reflection

---

## Credits
Tutorial by [Josh Madakor](https://www.youtube.com/@JoshMadakor).  
All implementation, exploration, and documentation performed independently as part of my cybersecurity learning journey.
