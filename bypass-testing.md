# Bypass Testing

Testing the security boundaries of the VPN and VLAN configurations — verifying isolation holds and identifying where it can be circumvented.

> All tests performed on lab VMs within an isolated network.

---

## VLAN Bypass Tests

### Test 1 — Direct Layer 3 Routing Between VLANs

**Hypothesis:** VLANs are isolated by firewall rules. If no rule allows OPT1 to reach OPT2, traffic between them should be dropped.

**Test:**
```bash
# From Kali (VLAN 10 - 192.168.10.x):
ping 192.168.20.x    # Ubuntu on VLAN 20
```

**Expected result:** No response — firewall drops the packet at the OPT1 interface because no rule permits traffic to the 192.168.20.0/24 subnet.

**Confirmed:** VLAN segmentation holds at Layer 3.

---

### Test 2 — VLAN Hopping (802.1Q Double Tagging)

**Hypothesis:** Classic VLAN hopping attacks rely on double-tagging 802.1Q frames to jump VLANs on a managed switch. This requires a trunk port and a native/untagged VLAN misconfiguration.

**Why it doesn't apply here:** This lab uses port-based VLANs via dedicated virtual interfaces — not a single trunk with 802.1Q tags. Each VLAN has its own physical (or virtual) interface on the firewall. There is no trunk port to exploit, no native VLAN to abuse, and no 802.1Q tag processing happening. Double-tagging attacks are architecturally impossible in this setup.

**Takeaway:** The port-based approach trades switch flexibility for a stronger isolation model. If you need many VLANs, 802.1Q is more scalable — but requires careful trunk port hardening (disable DTP, set explicit native VLAN, use VLAN pruning).

---

### Test 3 — Add a Static Route on the Client

**Hypothesis:** If a Kali machine manually adds a static route pointing to the Ubuntu subnet via the pfSense gateway, could it bypass VLAN isolation?

**Test:**
```bash
# On Kali:
ip route add 192.168.20.0/24 via 192.168.10.1
ping 192.168.20.x
```

**Expected result:** Still fails — the route sends the packet to pfSense (192.168.10.1), but pfSense applies its firewall rules before forwarding. Since no rule on OPT1 permits traffic to 192.168.20.0/24, the packet is dropped at the firewall. Static routes on the client don't bypass firewall policy.

**Takeaway:** The firewall is the enforcement point, not the client's routing table. Clients can add routes, but the firewall decides whether to forward.

---

### Test 4 — Shared Service Reachability

**Hypothesis:** Both VLANs have internet access. A service running on a public IP could act as an indirect bridge between VLAN 10 and VLAN 20 hosts.

**Test:**
```
Kali uploads data to external server
Ubuntu downloads same data from external server
```

**Expected result:** This works — but it's not a VLAN bypass. The VLANs remain isolated at Layer 3. The external server is a separate system acting as a relay. This is a data exfiltration path, not a network bypass.

**Takeaway:** VLAN isolation controls direct Layer 3 communication. It does not prevent hosts from communicating through shared external services (cloud storage, messaging, internet relays). DLP (Data Loss Prevention) and application-layer controls are needed to address this.

---

## VPN Bypass / Attack Tests

### Test 5 — Connect Without Valid Certificate

**Hypothesis:** OpenVPN is configured in SSL/TLS + User Auth mode. Attempting to connect without a valid client certificate should fail during the TLS handshake, before the username/password stage is even reached.

**Test:**
```
Attempt OpenVPN connection with:
  - No client certificate
  - Or a self-signed cert not signed by the lab CA
```

**Expected result:** Connection fails at TLS handshake — the server rejects the client because the certificate is not signed by the trusted CA.

**Takeaway:** Certificate pinning to a specific internal CA means even valid TLS certificates from public CAs won't work. An attacker would need to compromise the CA private key to forge a valid client certificate.

---

### Test 6 — Brute Force User Credentials

**Hypothesis:** After TLS authentication succeeds (valid cert), the server prompts for username and password. Repeated incorrect attempts may or may not be rate-limited depending on pfSense configuration.

**Gap:** By default, pfSense OpenVPN does not have built-in brute-force protection. An attacker with a valid client certificate could attempt many username/password combinations.

**Mitigation:**
- Enable **pfBlockerNG** or **fail2ban** (via package) to block IPs after repeated failures
- Use certificate-only authentication (remove the user auth requirement) — if the cert is the only factor, there's no password to brute force
- Enforce strong passwords + account lockout via RADIUS if using external authentication

---

### Test 7 — Intercept .ovpn File

**Hypothesis:** The `.ovpn` file exported from pfSense contains the CA certificate, client certificate, client private key, and server address — everything needed to connect. If this file is intercepted or stolen, an attacker gains full VPN access as that user.

**Gap:** The `.ovpn` file is a complete credential bundle. Losing it is equivalent to losing a password + hardware token combined.

**Mitigation:**
- Deliver `.ovpn` files over an encrypted channel only (never email in plaintext)
- Add a password to the client private key inside the `.ovpn` file — even with the file, the attacker needs the key passphrase
- Revoke compromised certificates immediately via **System > Cert. Manager > CAs > CRL**
- Use short certificate validity periods and rotate regularly

---

### Test 8 — Access Unreachable VLANs from VPN

**Hypothesis:** The OpenVPN server config specifies which local networks VPN clients can reach via **IPv4 Local Network**. If only the LAN (192.168.1.0/24) is listed, VPN clients should not be able to reach VLAN 10 or VLAN 20.

**Test:**
```
Connected via VPN (assigned 10.8.0.x):
ping 192.168.10.x    # VLAN 10
ping 192.168.20.x    # VLAN 20
```

**Expected result:** Fails — pfSense only pushes routes for the networks listed in the OpenVPN server's **IPv4 Local Network** field. VLAN subnets not listed there are not reachable from VPN clients.

**Takeaway:** VPN access scope is explicitly defined by the admin. The principle of least privilege applies — VPN clients should only reach the subnets they actually need.

---

## Summary

| Test | What's Tested | Result | Key Point |
|------|--------------|--------|-----------|
| Direct ping between VLANs | VLAN isolation | Blocked ✓ | Firewall default deny enforces segmentation |
| 802.1Q double-tagging | VLAN hopping | N/A — not applicable | Port-based VLANs are immune to this attack |
| Static route on client | VLAN isolation | Blocked ✓ | Client routing table doesn't override firewall policy |
| Shared external service | Data path | Possible | Not a Layer 3 bypass — needs DLP controls |
| Connect without cert | OpenVPN auth | Rejected ✓ | TLS mutual auth fails before password stage |
| Brute force credentials | OpenVPN auth | Possible gap | No built-in rate limiting — needs fail2ban/pfBlockerNG |
| Stolen .ovpn file | Credential security | Full access | Protect delivery; password-protect private key; use CRL |
| VPN → unreachable VLAN | VPN scope | Blocked ✓ | Route push only covers declared local networks |
