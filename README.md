# 🔒 Enterprise Infrastructure Lab — 90-Day Zero Trust Environment

[![Author](https://img.shields.io/badge/Author-Edris%20Ahmad%20Dost-blue)][Portfolio]
[![Platform](https://img.shields.io/badge/Hypervisor-KVM%2FQEMU%20%7C%20Debian%2013-orange)]()
[![Status](https://img.shields.io/badge/Status-Active%20%7C%20Phase%201-green)]()

This repository tracks the 12-week deployment of a complete, production-grade enterprise IT environment. Built from scratch on a single physical host (i7-12700H, 32GB RAM), this lab replicates the infrastructure and operational workflows expected in a modern French ESN (Entreprise de Services Numériques) or DSI.

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