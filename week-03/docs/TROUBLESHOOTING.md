# Mailcow + VPS Relay + OPNsense WireGuard — Troubleshooting Log

**Project:** ProtechCorp homelab — Week 3 (GLPI ITSM + Mailcow integration)
**Architecture:** Self-hosted Mailcow in DMZ (192.168.30.11) behind OPNsense,
public OVH VPS (146.59.232.85) as internet-facing mail gateway, connected via
WireGuard tunnel. Goal: keep home IP hidden while exposing mail services.

---

## 1. Mailcow web UI unreachable on standard ports

**Symptom:** `http://mail.protechcorp.net` returned a Squid "Zero Sized Reply".

**Cause:** Mailcow's nginx listens on 8080/8443 (`HTTP_PORT`/`HTTPS_PORT` in
`mailcow.conf`), nothing on 80/443.

**Fix:** Kept mailcow on 8080/8443 internally; reverse-proxied from the VPS's
nginx to `192.168.30.11:8443`.

---

## 2. Port 80/443 conflict with existing VPS website

**Symptom:** VPS already serves `protechcorp.net` on 80/443; needed
`mail.protechcorp.net` on the same IP/ports too.

**Cause:** Two hostnames, same IP/ports → requires SNI-based virtual hosting.

**Fix:** Added a dedicated `server_name mail.protechcorp.net` block
proxying to mailcow, alongside the existing site block. Kept the existing
catch-all `return 444` for unmatched hostnames.

---

## 3. Mailcow proxy block pointed to wrong target

**Symptom:** `proxy_pass https://10.0.0.2:443;` (OPNsense's WireGuard IP)
returned nothing usable.

**Cause:** Mailcow doesn't run on OPNsense's IP — it's at `192.168.30.11:8443`.

**Fix:** Corrected to `proxy_pass https://192.168.30.11:8443;` with
`proxy_ssl_verify off` (mailcow's cert won't match the private IP).

---

## 4. OPNsense WireGuard peer AllowedIPs too narrow

**Symptom:** VPS couldn't route to `192.168.30.0/24` (DMZ) through the tunnel.

**Cause:** VPS peer's `AllowedIPs` on OPNsense was only `10.0.0.0/24` —
WireGuard cryptokey routing drops packets for destinations outside
AllowedIPs.

**Fix:** Updated AllowedIPs to `10.0.0.0/24, 192.168.30.0/24`. Added a
persistent route on the VPS (`PostUp`/`PostDown` in `wg0.conf`):
```
ip route add 192.168.30.0/24 via 10.0.0.2 dev wg0
```

---

## 5. Inbound mail port forwarding (DNAT)

**Goal:** Forward public ports 25/465/587/993 from the VPS through WireGuard
to mailcow.

**Fix:**
- **VPS:** iptables DNAT (25/465/587/993 → 192.168.30.11), FORWARD rules
  for `<public_iface> <-> wg0` with ESTABLISHED/RELATED, MASQUERADE on
  POSTROUTING.
- **OPNsense:** Destination NAT on WG_VPS interface
  (`WG_VPS:MAIL_PORTS → 192.168.30.11:MAIL_PORTS`).

**Result:** Rules were written, but later found to be silently
non-functional — see Problem 6a below.

---

## 6. Outbound SMTP from mailcow timing out (port 25)

**Symptom:** Mailcow's Postfix couldn't deliver to external servers (e.g.
Google) on port 25 — connections timed out. VPS itself could reach Google on
25 directly, ruling out OVH egress blocking.

**Outbound path:** Mailcow → OPNsense (Policy-Based Routing on DMZ forces
traffic to `WG_VPS_GW` = 10.0.0.1) → Outbound NAT (192.168.30.11 → 10.0.0.2)
→ WireGuard tunnel → VPS (forward to `eth0`, MASQUERADE to 146.59.232.85)
→ internet.

**Diagnosis:**
- VPS FORWARD rules and `ip_forward=1` looked correct.
- OPNsense packet capture on WG_VPS showed SYN packets leaving (src
  10.0.0.2 → Google:25), retransmitted with no SYN-ACK — one-way traffic.
- `tcpdump` on VPS `wg0` during a synchronized test to check SYN arrival.
- `wg show` on VPS: both peers (OPNsense, laptop) had healthy handshakes;
  AllowedIPs correct.
- VPS POSTROUTING NAT table had multiple/duplicate, overly broad
  MASQUERADE rules.

**Resolution:** Root cause identified as 6a (wrong interface name in NAT
rules) — see below. Once corrected, outbound SMTP confirmed working via
Postfix logs and successful delivery to Gmail.

---

## 6a. Root cause: DNAT/MASQUERADE rules used wrong interface name (`eth0` vs `ens3`)

**Symptom:** Both inbound (Problem 5) and outbound (Problem 6) mail traffic
failed despite rules appearing syntactically correct. `iptables -t nat -L
... -v -x` showed **0 packets / 0 bytes** on every DNAT and the relevant
MASQUERADE rule, even after repeated test traffic.

**Cause:** All PREROUTING DNAT and the affected POSTROUTING MASQUERADE rules
specified `-i eth0` / `-o eth0`. This VPS's actual public interface is
**`ens3`** (confirmed via `ip a`) — `eth0` does not exist on this host. Every
rule was valid syntax but matched zero traffic.

**Fix:**
```bash
# Inbound DNAT — flush and rebuild with correct interface
sudo iptables -t nat -F PREROUTING
sudo iptables -t nat -A PREROUTING -i ens3 -p tcp --dport 25  -j DNAT --to-destination 192.168.30.11:25
sudo iptables -t nat -A PREROUTING -i ens3 -p tcp --dport 143 -j DNAT --to-destination 192.168.30.11:143
sudo iptables -t nat -A PREROUTING -i ens3 -p tcp --dport 465 -j DNAT --to-destination 192.168.30.11:465
sudo iptables -t nat -A PREROUTING -i ens3 -p tcp --dport 587 -j DNAT --to-destination 192.168.30.11:587
sudo iptables -t nat -A PREROUTING -i ens3 -p tcp --dport 993 -j DNAT --to-destination 192.168.30.11:993

# Outbound MASQUERADE — same fix
sudo iptables -t nat -D POSTROUTING -o eth0 -s 10.0.0.0/24 -j MASQUERADE
sudo iptables -t nat -A POSTROUTING -o ens3 -s 10.0.0.0/24 -j MASQUERADE
sudo iptables -t nat -A POSTROUTING -o ens3 -s 192.168.30.0/24 -j MASQUERADE
```

**Lesson:** Always verify interface names with `ip a` on the target host —
don't assume `eth0`; cloud VPS images (OVH, etc.) commonly use `ens3`,
`enp0s3`, etc. A 0-packet counter on a rule after live traffic is the
fastest tell that the rule isn't matching at all.

---

## 6b. Secondary blocker: postscreen → smtpd handoff expected PROXY protocol

**Symptom:** After fixing 6a, inbound connections reached mailcow's Postfix
but hung with no SMTP banner:
```
postfix/smtpd[1036]: connect from unknown[192.168.30.11]
postfix/smtpd[1036]: timeout after CONNECT from unknown[192.168.30.11] commands=0/0
```

**Cause:** `master.cf` had `postscreen_upstream_proxy_protocol=haproxy` on
the internal `10025` handoff, expecting an haproxy in front sending PROXY
protocol headers — but no haproxy is in this path, so Postfix waited
indefinitely for a header that never arrives.

**Fix:** Removed/commented `-o postscreen_upstream_proxy_protocol=haproxy`
in `data/conf/postfix/master.cf` (host-mounted file, persists across
container restarts — editing inside the running container does not
persist). Recreated the container (`docker compose down/up`, not
`restart`).

---

## 6c. Tertiary blocker: outbound IPv6 routing unreachable

**Symptom:** After 6a/6b, inbound worked but outbound to Gmail still failed
intermittently with `Network is unreachable` to IPv6 addresses
(`2a00:1450:...`).

**Cause:** Postfix tried Gmail's IPv6 MX records first; the
WireGuard/tunnel path is IPv4-only (10.0.0.0/24), so IPv6 attempts failed
before falling back to IPv4 (with delay).

**Fix:**
```bash
sudo docker compose exec postfix-mailcow postconf -e "inet_protocols = ipv4"
sudo docker compose restart postfix-mailcow
sudo docker compose exec postfix-mailcow postqueue -f
```

**Result:** Full inbound + outbound mail flow confirmed end to end via
Postfix logs and mail-tester.com (10/10).

---

**Symptom:** Setting up the GLPI mail fetcher cron failed with an
unrecognized option.

**Cause:** Typo — used `-w` instead of `-u`.

**Fix:**
```
sudo crontab -u www-data -e
```
```
* * * * * /usr/bin/php /var/www/html/glpi/front/cron.php --force mailgate
```

---

## 8. GLPI mailgate connection timeout (port 143)

**Symptom:** `stream_socket_client(): Unable to connect to
tcp://192.168.30.11:143 (Connection timed out)`.

**Cause:** OPNsense MGMT firewall only allowed `SRV_BASTION` → Mailcow; GLPI
(192.168.10.11) hit the default-deny rule.

**Fix:** Added a Pass rule on the OPNsense MGMT interface:
`VLAN10_MGMT → SRV_MAILCOW`, ports 143 & 993, placed **above** the broad
deny rules.

---

## 9. Mailcow rejecting valid GLPI credentials

**Symptom:** Network reachable, but GLPI reported invalid login. Dovecot
logs: `imap-login: Disconnected: Aborted login by logging out (auth failed)`.

**Cause:** GLPI was authenticating over port 143 (plaintext, no STARTTLS).
Mailcow's defaults reject unencrypted auth attempts.

**Fix:** Switched GLPI receiver to port **993 (IMAPS)** with connection
options `/imap /ssl /novalidate-cert` (`novalidate-cert` required because
GLPI connects via private IP, which won't match the public Let's Encrypt
cert).

---

## 10. Emails fetched and deleted, but no ticket created

**Symptom:** Dovecot logs showed successful fetch (`deleted=2 expunged=2`),
but GLPI's ticket queue stayed empty.

**Cause:** GLPI's default policy rejects tickets from unrecognized
(non-AD-synced) senders — emails were consumed to avoid a fetch loop but not
converted to tickets.

**Fix:** GLPI → Setup → General → Assistance → enabled "Allow anonymous
ticket creation" (`helpdesk.receiver`).

**Result:** Full end-to-end automation confirmed — email to
`support@protechcorp.net` → VPS → WireGuard → Mailcow → GLPI (IMAPS fetch)
→ auto-created ticket.

---

## Additional hardening

- Mailcow admin UI (8443) restricted to WireGuard-only — not exposed via
  VPS reverse proxy.
- SSH to VPS restricted to WireGuard tunnel only.
- DKIM, DMARC (`p=none` initially), SPF, PTR/rDNS configured.
- SMTP2GO determined unnecessary — VPS IP is clean with full DKIM control.

---

## Key takeaways

1. Non-default service ports require either reconfiguring the service or
   adjusting the proxy layer to match — don't assume defaults.
2. Multiple services on one public IP → use nginx SNI virtual hosting, not
   separate port forwards.
3. WireGuard `AllowedIPs` is bidirectional: it gates both accepted source
   IPs *and* routable destination subnets through that peer.
4. Multi-hop NAT/tunnel issues debug fastest with **simultaneous** packet
   captures at each hop, captured *during* the test — not before/after.
5. Outbound NAT (MASQUERADE) must exist at **every** hop where the source IP
   changes to stay routable.
6. Keep NAT rule sets clean — duplicate/overly broad rules (e.g.
   `0.0.0.0/0` MASQUERADE) hide the real issue.
7. Modern mail servers reject plaintext auth — use IMAPS (993) with
   `/ssl /novalidate-cert` when connecting via private IPs.
8. Firewalls evaluate top-down — a correct allow rule below a broad deny
   rule is still dropped.
9. Complex multi-hop NAT/routing/integration work should be staged in a
   homelab before any production deployment.
