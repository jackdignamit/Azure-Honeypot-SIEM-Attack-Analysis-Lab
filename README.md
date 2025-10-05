# 🌐 Azure Honeypot & SIEM Attack Analysis Home Lab 🌐
This project walks through how I built a cloud-based cybersecurity home lab using **Microsoft Sentinel** with Azure virtual machines to detect and display attack data. 

Inspired by [Josh Madakor's tutorial](https://youtu.be/g5JL2RIbThM?si=Mmk_O8heRLS_-SU1), I followed along and then expanded on it with hands-on exploration and insights.

---

# Project Overview

This project showcases how I established a cloud-based honeypot VM in Azure and sent brute-force attempt logs to a SIEM workspace. 
The workspace was integrated into Microsoft Sentinel to identify failed login attempts, conduct geolocation IP filtering, and visualize source locations on a live attack map. 
I then wrote KQL queries for event filtering, analyzing, and alert creation; acquiring hands-on experience in SIEM operations, log analytics, and security monitoring.

### **Platforms and Technology Used:** 
Azure VMs, SIEM (Microsoft Sentinel), KQL, Log Analytics, Data visualization, Networking & firewall configuration, Geolocation IP filtering

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

### 4️⃣ Create LAW and link to Microsoft Sentinel
1. Create both a new **Log Analytics Workspace (LAW)** and **Microsoft Sentinel** workspace under your resource group.

   _The Log Analytics Workspace will act as a centralized storage and analysis hub for incoming logs from your VM. Microsoft Sentinel adds threat detection, alerting, and visualization capabilities. Feel free to export its built-in workbooks and analytic rules._
   
3. On Microsoft Sentinel, **add the previously created Log Analytics Workspace.** This links the LAW to its respective SIEM.
4. Under **Content Management > Content Hub** search for and install Windows Security Events.
<img width="1398" height="741" alt="image" src="https://github.com/user-attachments/assets/46f6791d-e4dd-4eea-96fb-58df9d3410f9" />

5. Under Windows Security Events, click on "manage" and install **"Windows Security Events via AMA"**.

_Azure Monitoring Agent (AMA) allows all security events from the Windows machine to stream directly into the Microsoft Sentinel workspace._

6. Once installed, open connector page and **create a new data collection rule**. Select your VM under the resources tab and collect **All Security Events**. It may take a moment for the extension to be applied to the VM.

---

### 5️⃣ Query for logs within the LAW
1. Observe some of the ingested logs using KQL filters.
