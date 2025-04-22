# 🌐 Hybrid Cloud Infrastructure with Azure Failover

This project demonstrates how I extended an on-premise IT infrastructure to Microsoft Azure and implemented a failover mechanism using Azure VMs for business-critical servers hosted on VMware ESXi.

---

## 📌 Table of Contents
- [Overview](#-overview)
- [Architecture](#-architecture)
- [Steps & Configuration – Hybrid Network Setup](#-steps--configuration--hybrid-network-setup)
- [Steps & Configuration – Azure Migrate and Failover](#-steps--configuration--azure-migrate-and-failover)
- [Conclusion](#-conclusion)
---

## 🧾 Overview

This project combines two key goals:

1. **Hybrid Networking:** Extend the internal network to Azure to allow communication between Azure VMs and on-prem servers (such as AD and internal applications).
2. **Failover to Azure:** Set up a failover mechanism from on-premise VMware ESXi to Azure for internal servers like a Web server and Git server, ensuring high availability.

---

## 🗺️ Architecture

The hybrid infrastructure consists of an on-premises network connected to Azure via a Site-to-Site VPN. Here's how the setup is structured:

### On-Premise Infrastructure
- **Firewall**: FortiGate with public IP.
- **LAN/Production Network**: Hosts internal systems like:
  - Active Directory Domain Controller
  - Web Server (running in VMware ESXi)
  - Git Server and user desktops

### Azure Infrastructure
- **Virtual Network (VNet)**: Custom address space aligned with on-premise subnet ranges.
  - **Gateway Subnet**: Contains Azure VPN Gateway for S2S VPN.
  - **Production Subnet**: Hosts Azure VMs which need access to on-premise services.
 
    ![ChatGPT Image Apr 21, 2025, 10_16_24 PM](https://github.com/user-attachments/assets/8c120446-9917-4e21-a8c8-f777e36ca467)

## 🛠️ Steps & Configuration – Hybrid Network Setup

This section covers the detailed configuration to establish a hybrid network connection between on-premise infrastructure and Azure using a Site-to-Site VPN.

---

### ☁️ Azure Configuration

1. **Create Virtual Network (VNet)**
   - Define a CIDR block **not overlapping** with the on-prem network.
   - This VNet will contain the VPN gateway subnet and the production subnet for Azure VMs.

2. **Rename Default Subnet**
   - Rename the default subnet to `Productionnetwork` (or a suitable name).
   - Assign a CIDR block (e.g., `10.1.0.0/24`) — this is where Azure VMs will be placed to communicate with on-prem servers.

3. **Create Gateway Subnet**
   - Create a dedicated `GatewaySubnet` (e.g., `10.1.255.0/27`) required for VPN Gateway.

4. **Create Public IP**
   - Create a public IP to be associated with the VPN Gateway.

5. **Create Virtual Network Gateway**
   - Region must match with VNet.
   - Associate the newly created Gateway Subnet and Public IP.

6. **Create Local Network Gateway**
   - This represents the on-prem network in Azure.
   - Provide:
     - **Public IP of FortiGate firewall**
     - **Address space of on-prem LAN** (e.g., `192.168.10.0/24`)

7. **Create Site-to-Site VPN Connection**
   - Go to the Virtual Network Gateway → Connections → Create New
   - **Connection type**: Site-to-site (IPsec)
   - Select the created Local Network Gateway
   - Enter a **Shared Key** (Pre-shared key) — this will be used in FortiGate configuration too.

---

### 🏠 On-Premises FortiGate Configuration

1. **Login to FortiGate Firewall**

2. **Create IPSec VPN Tunnel**
   - Go to **VPN → IPsec Wizard → Select Custom**
   - Configuration:
     1. **Remote Gateway IP**: Azure VPN Gateway public IP
     2. **Interface**: Public (WAN-facing)
     3. **Pre-shared Key**: Use the same key as defined in Azure
     4. **NAT**: Disabled
     5. **Dead Peer Detection**: On Idle
     6. **Version**: IKEv2
     7. **Phase 1/2 Encryption Algorithms**: Choose secure algorithms like AES256, SHA256

![image](https://github.com/user-attachments/assets/638b7ff5-3460-4c51-b8a9-23bd1c1e9e8e)

![image](https://github.com/user-attachments/assets/6c852285-dfe5-424b-b4e7-3032eefd15d1)


3. **Create Address Object for Azure Network**
   - Go to **Policy & Objects → Addresses → Create New**
   - Name: `Azure-VNet`
   - Subnet: CIDR of Azure Productionnetwork (e.g., `10.1.0.0/24`)
   - Interface: IPsec Tunnel Interface

4. **Create Firewall Policies**

   **Policy 1: On-Prem → Azure**
   - Incoming Interface: On-Prem LAN
   - Outgoing Interface: IPsec Tunnel
   - Source: All
   - Destination: `Azure-VNet`
   - Service: All (or restrict as needed)
   - NAT: Disabled

   **Policy 2: Azure → On-Prem**
   - Incoming Interface: IPsec Tunnel
   - Outgoing Interface: On-Prem LAN
   - Source: `Azure-VNet`
   - Destination: All
   - Service: All (or restrict as needed)
   - NAT: Disabled

5. **Add Static Route for Azure Traffic**
   - Go to **Network → Static Routes → Create New**
   - Destination: Azure VNet CIDR (e.g., `10.1.0.0/24`)
   - Interface: IPsec Tunnel Interface
   - Administrative Distance: Lower than default route (e.g., `5`)

---

### ✅ Validation Steps

1. **In Azure Portal**
   - Go to **Virtual Network Gateway → Connections**
   - Check if **Status = Connected**

2. **In FortiGate Firewall**
   - Go to **Monitor → IPsec Monitor**
   - Check if tunnel is **Up**

3. **Connectivity Test**
   - Deploy a VM in Azure under `Productionnetwork` subnet
   - Try pinging:
     - On-prem servers (e.g., AD server)
     - Access internal web apps or file shares from Azure VM

---

✅ Once this setup is complete, Azure VMs will be able to communicate with internal on-prem servers, and vice versa. This enables a true hybrid cloud environment.

## 🛠️ Steps & Configuration – Azure Migrate and Failover

![ChatGPT Image Apr 22, 2025, 07_58_16 PM](https://github.com/user-attachments/assets/3362d669-f0e2-410c-bcb1-e4b38ea8617b)

### ✅ Pre-requisites
- Hybrid Network infrastructure with **VPN Gateway** already established (see previous section).
- On-premise **VMware ESXi** environment.
- Azure subscription with **VNet and subnet** configured (non-conflicting CIDR with on-prem).

---

### 🚀 Azure Migrate Setup

1. **Create a Migration Project**
   - Navigate to **Azure Migrate**.
   - Go to **"Servers, Databases and Webapps"** → Create a project.
   - Use **Public Endpoint**.

2. **Setup Appliance Server (Connector)**
   - Prepare a **Windows Server VM** in your on-prem network.
   - In Azure Migrate → click **Discover**, choose:
     - **"Via Appliance"**
     - Platform: **VMware vSphere**
   - Generate project key.
   - Download the **PowerShell template (instead of OVA)** and run:
     ```powershell
     .\AzureMigrateInstaller.exe
     ```
     Follow prompts:
     - Select **VMware**
     - Azure **Public**
     - Primary appliance
     - Provide previously generated **Project Key**

3. **Configure the Appliance**
   - Open the Azure Migrate Appliance Manager (browser shortcut).
   - Complete **pre-requisites** checks.
   - **Authenticate to Azure** and validate the project key.
   - Enter VMware ESXi details:
     - IP, port, credentials
     - Skip SQL, software inventory, ASP.NET discovery (optional)
   - Start **Discovery**

---

### 📊 Assess VM Compatibility

1. In Azure Migrate → Go to **Machines → Assess**
   - Select **Azure VM** as target.
   - Fill assessment settings (region, subscription, VM sizing).
   - Create a **Group** and assign VMs.
   - Review **compliance report**, VM sizing, and **estimated monthly costs**.
  
     ![image](https://github.com/user-attachments/assets/d20875e7-e326-4d71-ab57-2d24c34f6964)

![image](https://github.com/user-attachments/assets/a2d019fc-e314-4ccb-8f09-1e874b73456f)


---

### 🔄 Replicate VM to Azure

1. In Azure Migrate → **Replicate**:
   - Choose source: **VMware**
   - Use existing **Appliance** and **Group**
   - Select VMs to replicate
   - Set:
     - **Target Region**
     - **VNet** (Hybrid setup)
     - **Production Subnet**
     - **No infrastructure redundancy** (or change as required)
   - Validate and begin **Replication**

2. After replication:
   - Go to **Replicated Machines**
   - Ensure **"Migration Ready"** status

---

### 🔁 Migrate (Failover)

1. In Azure Migrate → **Migrate** the replicated VM
   - Optionally **Test Migration**
   - Shutdown original on-prem VM (to avoid data mismatch)
   - Proceed with full migration

2. Monitor progress under **Migration Jobs**

---

### 🧪 Post-Migration Validation

- Stop replication to avoid unwanted changes.
- Assign a **Public IP** to Azure VM.
- **RDP into the migrated VM**
- Confirm:
  - Services and applications are intact
  - Connectivity to on-premise via VPN is working

---

## 📌 Conclusion

This project demonstrates a comprehensive real-world implementation of a **hybrid IT infrastructure** by securely extending an on-premises network to **Microsoft Azure** using a Site-to-Site VPN configuration. Key accomplishments include:

- 🔐 **Secure Hybrid Connectivity**: Designed and implemented a VPN tunnel between on-premise FortiGate Firewall and Azure VPN Gateway, ensuring seamless communication between on-prem systems (Active Directory, internal applications) and Azure-hosted VMs.

- ☁️ **Azure Migration & Failover**: Leveraged Azure Migrate to assess, replicate, and migrate critical on-prem virtual machines (Web Server and Git Server) from VMware ESXi to Azure for business continuity and high availability.

- 🛠️ **Custom Appliance Setup**: Deployed and configured the Azure Migrate appliance on-premises to integrate with VMware vSphere, handling discovery, assessment, and replication of workloads.

- 🧪 **Testing & Validation**: Verified end-to-end connectivity, application accessibility, and service availability post-migration through RDP access and hybrid network tests.

This solution ensures **high availability**, **disaster recovery readiness**, and **infrastructure scalability** by leveraging both on-premise and cloud environments in a unified and secure architecture.

