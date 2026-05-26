# Automating Multi-AS BGP Routing & L2 Switch Deployment with Ansible & EVE-NG

[![Ansible](https://img.shields.io/badge/Ansible-black.svg?style=for-the-badge&logo=ansible&logoColor=white)](https://www.ansible.com/)
[![Cisco](https://img.shields.io/badge/Cisco-IOS-blue.svg?style=for-the-badge&logo=cisco&logoColor=white)](https://www.cisco.com/)
[![Network Automation](https://img.shields.io/badge/Network-Automation-orange.svg?style=for-the-badge)](https://github.com/topics/network-automation)
[![BGP](https://img.shields.io/badge/Routing-BGP-red.svg?style=for-the-badge)](https://en.wikipedia.org/wiki/Border_Gateway_Protocol)

An enterprise-ready Ansible automation project designed to deploy a **Multi-Autonomous System (AS) eBGP** network and associated Layer 2 switched infrastructure. The configuration targets Cisco IOS routers (`R1-R5`) and switches (`SW1-SW5`) simulated inside **EVE-NG** (Emulated Virtual Environment - Next Generation).

This project highlights modern Infrastructure as Code (IaC) principles by separating configuration data (variables) from execution logic (playbooks and roles).

---

## 📌 Project Overview & Topology

The topology forms a ring-like multi-AS network connecting 5 routers through eBGP peers. Associated switches handle access and trunk VLAN distribution at the local sites.

### 🗺️ BGP AS and Interface Layout

*   **R1 (AS 65001):**
    *   Router ID: `1.1.1.1` (Loopback0)
    *   eBGP Peers: `R2` (AS 65002) via `192.168.12.2`, `R5` (AS 65005) via `192.168.15.5`
*   **R2 (AS 65002):**
    *   Router ID: `2.2.2.2` (Loopback0)
    *   eBGP Peers: `R1` (AS 65001) via `192.168.12.1`, `R3` (AS 65003) via `192.168.23.3`
*   **R3 (AS 65003):**
    *   Router ID: `3.3.3.3` (Loopback0)
    *   eBGP Peers: `R2` (AS 65002) via `192.168.23.2`, `R4` (AS 65004) via `192.168.34.4`
*   **R4 (AS 65004):**
    *   Router ID: `4.4.4.4` (Loopback0)
    *   eBGP Peers: `R3` (AS 65003) via `192.168.34.3`, `R5` (AS 65005) via `192.168.45.5`
*   **R5 (AS 65005):**
    *   Router ID: `5.5.5.5` (Loopback0)
    *   eBGP Peers: `R4` (AS 65004) via `192.168.45.4`, `R1` (AS 65001) via `192.168.15.1`

### ⚙️ Automated Features

*   **System Identity:** Configures standardized hostnames (`R1-R5` and `SW1-SW5`) dynamically.
*   **Layer 3 Routed Infrastructure:**
    *   IPv4 address assignment for physical interfaces.
    *   Router-ID loopback interfaces creation and addressing.
*   **External BGP (eBGP) Routing:**
    *   Activation of BGP routing process per router with unique Autonomous System Numbers (ASN).
    *   Establishing BGP neighbors with correct remote ASNs and descriptions.
    *   Advertising local loopback network blocks into the BGP table.
*   **Layer 2 Switch Infrastructure:**
    *   VLAN creation (`10: IT`, `20: HR`, `30: FINANCE`, `40: HUMAN_RESOURCE`, `50: WIFI`, `60: SERVICE`).
    *   Trunk port configuration with 802.1Q encapsulation.
    *   Access ports mapped to designated VLANs.
    *   Spanning-Tree optimization configured in **Rapid-PVST** mode.
*   **Configuration Persistence:** Ensures running-config is saved to startup-config (`write memory` / `save_when: always`).

---

## 📂 Project Structure

```directory
bgp/
├── host.ini               # Inventory file containing host details and connection vars
├── site.yml               # Main master playbook mapping roles to hosts
├── requirements.yml       # Ansible Galaxy collection dependencies
├── screenshot/            # Directory containing network topology diagram
└── roles/
    ├── router_config/     # Role for Layer 3 interface & BGP configuration
    │   ├── tasks/
    │   │   └── main.yml   # Main task execution flow for routers
    │   └── vars/
    │       └── main.yml   # Interface IPs, ASNs, loopbacks, and peer settings
    └── switch_config/     # Role for Layer 2 VLAN, trunks, and Spanning-Tree
        ├── tasks/
        │   └── main.yml   # Main task execution flow for switches
        └── vars/
            └── main.yml   # VLAN databases and switchport allocation profiles
```

---

## 🛠️ Prerequisites & Setup

### 1. Control Node Requirements
*   Python 3.9+
*   Ansible Core 2.12+
*   Ansible Cisco.IOS Collection (v4.0.0 or higher)

### 2. Install Required Collections
Verify and install dependencies using `requirements.yml`:
```bash
ansible-galaxy collection install -r requirements.yml
```

### 3. Connection Inventory (`host.ini`)
Devices use the `ansible.netcommon.network_cli` connection plugin:
```ini
[routers]
R1 ansible_host=192.168.110.21
R2 ansible_host=192.168.110.22
R3 ansible_host=192.168.110.23
R4 ansible_host=192.168.110.24
R5 ansible_host=192.168.110.25

[switches]
SW1 ansible_host=192.168.20.21
SW2 ansible_host=192.168.20.22
SW3 ansible_host=192.168.20.23
SW4 ansible_host=192.168.20.24
SW5 ansible_host=192.168.20.25

[cisco:vars]
ansible_connection=ansible.netcommon.network_cli
ansible_network_os=cisco.ios.ios
ansible_user=admin
ansible_password=cisco123
ansible_become=yes
ansible_become_method=enable
ansible_become_password=cisco123
```

---

## 🚀 Deployment

Run the master playbook to deploy variables to all devices:

```bash
ansible-playbook -i host.ini site.yml
```

### Target Execution Using Limits
Limit configuration changes to only routers or only switches:
```bash
# Deploy router interfaces & BGP configurations only
ansible-playbook -i host.ini site.yml --limit routers

# Deploy switch L2 interfaces & VLAN configurations only
ansible-playbook -i host.ini site.yml --limit switches
```

---

## 🔍 Verification & Troubleshooting

After execution, log into your Cisco IOS command-line interfaces to verify operational status.

### 1. Verify BGP Routing Table and Peers
*   **Check BGP Neighbors & Summary:**
    ```ios
    show ip bgp summary
    ```
    *Ensure peer states transition to `Established` with prefixes received.*

*   **View Advertised/Received BGP Routes:**
    ```ios
    show ip bgp
    ```

*   **Check IPv4 Routing Table for BGP Routes:**
    ```ios
    show ip route bgp
    ```
    *Verify that remote loopbacks (e.g., `5.5.5.5` on `R1`) are accessible via BGP.*

### 2. Verify Layer 2 Switch Configurations
*   **VLAN Database Check:**
    ```ios
    show vlan brief
    ```
*   **Trunk Ports Check:**
    ```ios
    show interfaces trunk
    ```
*   **Spanning Tree Check:**
    ```ios
    show spanning-tree summary
    ```

---

## 🤝 Contributing
Feel free to open an issue or submit a pull request if you want to add further features (such as iBGP setups, route reflectors, prefix-lists, or route maps).
