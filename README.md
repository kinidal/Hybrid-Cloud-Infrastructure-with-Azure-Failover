# 🌐 Hybrid Cloud Infrastructure with Azure Failover

This project demonstrates how I extended an on-premise IT infrastructure to Microsoft Azure and implemented a failover mechanism using Azure VMs for business-critical servers hosted on VMware ESXi.

---

## 📌 Table of Contents
- [Overview](#overview)
- [Architecture](#architecture)
- [Steps & Configuration](#steps--configuration)
- [Challenges Faced](#challenges-faced)
- [Results](#results)
- [Next Steps / Future Improvements](#next-steps--future-improvements)

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

