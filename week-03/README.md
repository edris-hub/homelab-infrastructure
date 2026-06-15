# Week 3 — GLPI ITSM & Mailcow Integration

Part of the ProtechCorp 90-day enterprise lab build.

## Summary

This week covered deploying GLPI (ITSM/CMDB) with AD/LDAP authentication and
SMTP integration, plus a full self-hosted email stack: Mailcow in the DMZ
behind OPNsense, with an OVH VPS acting as the public-facing mail gateway
over a WireGuard tunnel.

## What's in this folder

- **TROUBLESHOOTING.md** — detailed log of 10 issues encountered and
  resolved while wiring up the VPS relay, WireGuard routing, NAT (inbound +
  outbound), and the GLPI ↔ Mailcow mail-to-ticket integration.
- **screenshots/** — evidence for each Week 3 deliverable (see below).

## Architecture

```
Internet → VPS (146.59.232.85, nginx SNI + iptables DNAT)
         → WireGuard tunnel (10.0.0.0/24)
         → OPNsense (Policy-Based Routing, DNAT)
         → Mailcow (192.168.30.11, DMZ)
         ← GLPI (192.168.10.11, MGMT) via IMAPS (993)
```

## Deliverables → Screenshots

| Deliverable | File |
|---|---|
| GLPI accessible from clients | `glpi_accessible_from_linux-client-01.png`, `glpi_accessible_from_win-client-01.png` |
| CMDB asset inventory | `glpi_inventory.png`, `glpi_inventory-1.png` |
| Ticket lifecycle | `glpi_ticket_01.png` |
| AD/LDAP user import | `LDAP_user_import.png` |
| Mailcow web UI accessible | `mailcow_SOGo_desktop.png`, `mailcow_from_phone.png` |
| Mailcow admin restricted (WireGuard-only) | `mailcow_admin_forbidden.png`, `mailcow_admin_from_phone.png` |
| Inbound mail flow proof | `smtpd_inbound.png`, `email_reception_proof.png` |
| Outbound SMTP proof | `smtp_outbound.png` |
| GLPI mail-to-ticket automation | `email_automatic_ticket_creation_GLPI.png` |
| DKIM/DMARC/SPF validation | `mail_tester_score.png`, `mail_tester_dkim.png` |
| OPNsense WireGuard / firewall config | `opnsense_wg_peers.png`, `opnsense_wg_gw.png`, `mgmt_dmz.png` |
| GLPI config reference | `glpi_conf_file.png` |

## Key skills demonstrated

GLPI ITSM, CMDB, SMTP/IMAPS integration, Mailcow (Docker Compose), WireGuard
site-to-site tunneling, multi-hop NAT (DNAT/SNAT), OPNsense firewall &
Policy-Based Routing, DKIM/DMARC/SPF, nginx SNI reverse proxying.
