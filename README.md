# 🔒 Enterprise Network & Systems Lab: 12-Week Zero Trust Validation

This repository tracks the 12-week deployment of a complete, production-grade enterprise IT environment. Built from scratch on a single physical host (i7-12700H, 32GB RAM), this lab replicates the infrastructure and operational workflows expected in a modern ESN or DSI.

The architecture enforces a strict **Zero Trust / Default-Deny** security posture, featuring micro-segmented VLANs, federated identity management, centralized observability, and Infrastructure as Code (IaC) automation.

---

## 🗺️ High-Level Architecture

The network is physically bridged but logically separated into three strict security zones behind an OPNsense firewall.

```text
                                     [ Public Internet ]
                                              │
                                     [ OPNsense Firewall ]
                                        (Default-Deny)
                                              │
                      ┌───────────────────────┼───────────────────────┐
                      │                       │                       │
                   VLAN 10                 VLAN 20                 VLAN 30
                 [ MGMT ]                 [ CORP ]                 [ DMZ ]
              Management Plane        Internal Enterprise        Public-Facing
              192.168.10.0/24         192.168.20.0/24          192.168.30.0/24

🛠️ Core Technology Stack

    Routing & Security: OPNsense, WireGuard (Split-Tunneling), nftables, Fail2ban

    Identity & Access: Active Directory (Windows Server 2022), Keycloak (SSO / SAML / LDAP)

    ITSM & Ticketing: GLPI (Automated IMAPS ticket ingestion)

    Messaging: Sovereign Mailcow (Dockerized) with DKIM/DMARC/SPF via public VPS relay

    Automation (IaC): Ansible, Bash, PowerShell

    Observability: Zabbix (Telemetry & Webhooks), Grafana, Centralized rsyslog

📁 Repository Structure

Detailed documentation, architectural diagrams, configurations, and scripts are categorized by their respective deployment week.

    /week-01/ — Network Architecture, IP Addressing, OPNsense Default-Deny Policies & VLANs.

    /week-02/ — Hybrid Identity: Active Directory deployment, OU structures, CSV automation, and Keycloak SSO federation.

    /week-03/ — ITSM & Sovereign Mail: GLPI deployment, Mailcow staging via WireGuard, and automated IMAPS mail-to-ticket routing.

    /week-04/ — Zero Trust Access: Bastion Host implementation and aggressive SSH hardening. (Upcoming)

(Folders for Weeks 05 through 12 covering Ansible automation, Zabbix monitoring, HAProxy reverse proxies, and Disaster Recovery will be published as the deployment progresses.)
🚀 Project Roadmap

Phase 1: Enterprise Foundation & Security Boundary (Weeks 1–4)
Establishing the hypervisor, routing, default-deny perimeter, directory services, and ITIL ticketing workflows.

Phase 2: Operations, Automation & Visibility (Weeks 5–8)
Transitioning from manual configuration to Infrastructure as Code (Ansible), deploying enterprise telemetry (Zabbix/Grafana), and configuring reverse proxies (HAProxy) with automated PKI.

Phase 3: Disaster Recovery & Production Handover (Weeks 9–12)
Simulating ransomware attacks with timed backups/restores, high-availability failovers, vulnerability scanning (Lynis/Wazuh), and final TSSR jury dossier preparation.
👤 Contact

    Lead Infrastructure Engineer: Edris Ahmad Dost

    Portfolio & Full Documentation: protechcorp.net

    Professional Profile: LinkedIn
