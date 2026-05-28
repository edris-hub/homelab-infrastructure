# Homelab — Zero Trust Enterprise Lab

Personal infrastructure lab built on a single Debian 13 laptop using KVM/QEMU.  
Simulates a segmented enterprise network with OPNsense, Active Directory, SSO, monitoring, and automation.  
**Status: Active — Week 2/13**

---

## Architecture Overview

```
                        [ Internet ]
                             │
                        [ OPNsense ]
                        (Firewall/Router)
                    ┌────────┼────────┐
                    │        │        │
               VLAN 10    VLAN 20   VLAN 30
                MGMT        CORP      DMZ
           192.168.10.0  192.168.20.0  192.168.30.0
                /24          /24          /24
```

---

## Physical Host

| | |
|---|---|
| **Machine** | Laptop — Intel i7-12700H, 32GB RAM, RTX 3070 Ti |
| **Hypervisor** | KVM/QEMU + virt-manager on Debian 13 (Trixie) |
| **Virtual switch** | `vmbr0` — Linux bridge with 802.1Q VLAN filtering |

---

## 🖥️ VM Inventory

| Hostname | VLAN | IP | Role |
|---|---|---|---|
| opnsense | WAN + trunk | 192.168.20.1 / .10.1 / .30.1 | Firewall / Router / DHCP |
| win-srv-01 | CORP 20 | 192.168.20.10 | Active Directory / DNS / DHCP / GPO |
| win-srv-02 | CORP 20 | 192.168.20.11 | File Server / Samba |
| win-client-01 | CORP 20 | 192.168.20.12 | Windows 11 Workstation |
| linux-client-01 | CORP 20 | 192.168.20.13 | Debian Workstation (Admin) |
| linux-srv-01 | DMZ 30 | 192.168.30.10 | Nginx / Nextcloud / Keycloak SSO |
| linux-srv-02 | MGMT 10 | 192.168.10.10 | Zabbix + Grafana / Wazuh |
| linux-srv-03 | MGMT 10 | 192.168.10.11 | Ansible / GLPI / Bastion / rsyslog |

---

## 🔒 Firewall Policy — Default Deny

Inter-VLAN traffic follows a Zero Trust default-deny model. No implicit trust between segments.

### CORP (VLAN 20) Rules

| Source | Destination | Port | Action | Description |
|---|---|---|---|---|
| CORP | OPNsense | 53 | ✅ Allow | DNS |
| Admin WS | OPNsense WebGUI | 443 | ✅ Allow | Admin access only |
| CORP | Internet | 80/443 | ✅ Allow | Web browsing |
| CORP | win-srv-01 | AD ports | ✅ Allow | AD/DNS |
| CORP | win-srv-02 | SMB | ✅ Allow | File server |
| CORP | DMZ web | 80/443 | ✅ Allow | Nextcloud/Keycloak |
| Admin WS | Bastion | 22 | ✅ Allow | SSH to bastion only |
| CORP | RFC1918 | * | ❌ Deny | Inter-VLAN block (logged) |

### DMZ (VLAN 30) Rules

| Source | Destination | Port | Action | Description |
|---|---|---|---|---|
| DMZ | win-srv-01 | 389/636 | ✅ Allow | Keycloak → AD LDAP/LDAPS |
| DMZ | Internet | 80/443 | ✅ Allow | Updates/packages |
| DMZ | RFC1918 | * | ❌ Deny | No lateral movement (logged) |

### MGMT (VLAN 10) Rules

| Source | Destination | Port | Action | Description |
|---|---|---|---|---|
| Bastion | All VLANs | 22 | ✅ Allow | SSH management plane |
| MGMT | All VLANs | ICMP | ✅ Allow | Monitoring reachability |
| Monitoring | All VLANs | SNMP | ✅ Allow | Zabbix polling |
| MGMT | Internet | 80/443 | ✅ Allow | Updates/packages |
| MGMT | * | * | ❌ Deny | Default deny |

---

## 🧱 Architecture Decisions

### Why Linux Bridges over isolated KVM networks?

- Native 802.1Q trunking support
- Matches Proxmox mechanics — transferable to production environments
- Cleaner VM configuration — only two vNICs needed on OPNsense
- Single `vmbr0` bridge acts as a managed virtual switch

### Why Default-Deny inter-VLAN policy?

- Follows Zero Trust model — no implicit trust between segments
- Matches enterprise DSI standards
- Minimises lateral movement risk if a segment is compromised
- Forces explicit documentation of every allowed traffic flow

---

## 📁 Repository Structure

```
homelab-infrastructure/
├── README.md
├── docs/
│   ├── architecture-choices.md
│   └── ip-addressing-table.md
└── screenshots/
    ├── CORP-firewall-rules.png
    ├── DMZ-firewall-rules.png
    └── MGMT-firewall-rules.png
```

---

## 🛠️ Stack

**Networking:** OPNsense · VLANs 802.1Q · Default-Deny · DMZ · Bastion Host  
**Identity:** Active Directory · Keycloak SSO · LDAPS · GPO  
**Monitoring:** Zabbix · Grafana · Wazuh · SNMP · rsyslog  
**Automation:** Ansible · PowerShell · Bash  
**Services:** Nextcloud · GLPI · Nginx · Samba  

---

## 👤 Author

**Edris Ahmad Dost** — Technicien Supérieur Systèmes et Réseaux  
🌐 [protechcorp.net](https://protechcorp.net) · 💼 [LinkedIn](https://linkedin.com/in/edrisahmaddost)
