# Comparison — OpenVPN vs IPsec vs VLAN Segmentation

Three mechanisms configured in this lab. Each solves a different problem and operates differently under the hood.

---

## At a Glance

| | OpenVPN | IPsec | VLAN Segmentation |
|-|---------|-------|------------------|
| **Purpose** | Secure remote access for individual users | Connect two separate networks | Isolate groups of hosts within the same physical network |
| **OSI Layer** | Layer 3 (tunnels IP in UDP/TCP) | Layer 3 (ESP/AH encapsulation) | Layer 2 / Layer 3 boundary |
| **Typical use case** | Remote worker accessing office LAN | Branch office ↔ HQ connectivity | Separating guest, staff, server networks |
| **Requires client software** | Yes (OpenVPN client) | Usually built into OS | No — transparent to hosts |
| **Authentication** | Certificates + username/password | PSK or certificates (IKE) | None — based on port/interface assignment |
| **Encrypts traffic** | Yes (TLS) | Yes (ESP) | No — isolation only, no encryption |
| **Complexity** | Medium | Medium–High | Low (port-based) / Medium (802.1Q) |
| **Bypassable by** | Stolen .ovpn file, weak credentials | Compromised PSK, IKE downgrade | Misconfigured firewall rules, 802.1Q hopping (tagged only) |

---

## OpenVPN

**What it does:** Creates an encrypted tunnel between a remote client and the pfSense firewall. The client is assigned a virtual IP in the tunnel network and can reach internal resources as if it were physically on the LAN.

**How it authenticates:**
- TLS mutual authentication (both server and client present certificates signed by the same CA)
- Username + password (second factor on top of certificate)

Both must succeed. A stolen password without a valid cert goes nowhere. A stolen cert without the password (in SSL/TLS + User Auth mode) also fails.

**Best for:**
- Remote workers, traveling staff, home office access
- Encrypted access to internal infrastructure from untrusted networks
- Scenarios where you need per-user identity and audit trail

**Limitations:**
- Requires client software installation
- UDP 1194 may be blocked on restrictive networks (can be worked around by running OpenVPN over TCP 443)
- Performance overhead from encryption (negligible on modern hardware)
- `.ovpn` file is a sensitive credential bundle — requires careful handling

---

## IPsec

**What it does:** Establishes an encrypted Layer 3 tunnel between two firewalls (or routers). Hosts on each side see the other's network as directly reachable — no client software required, no VPN client to install.

**Two phases:**
- **IKE Phase 1:** Authenticates the two gateways and establishes a secure channel for negotiation
- **IKE Phase 2 / IPsec SA:** Negotiates the actual encryption parameters for data traffic

**Best for:**
- Site-to-site connectivity (office ↔ office, office ↔ datacenter, cloud VPC ↔ on-prem)
- Scenarios where hosts on each side should communicate seamlessly without any client configuration
- Standards-based interoperability (IPsec is supported by virtually all network vendors)

**Limitations:**
- More complex to configure and troubleshoot than OpenVPN
- NAT traversal (NAT-T) adds complexity when firewalls are behind NAT
- Pre-shared keys (PSKs) are weaker than certificate-based auth — shared secret means any device with the PSK can authenticate
- IKEv1 has known weaknesses — always prefer IKEv2

**OpenVPN vs IPsec for remote access:**

| | OpenVPN | IPsec (IKEv2) |
|-|---------|--------------|
| Client software | Required | Built into most OSes |
| Firewall traversal | Easy (TCP 443 option) | Harder (UDP 500/4500, ESP) |
| Per-user auth | Yes (certs + password) | Yes (EAP methods) |
| Speed | Slightly slower | Faster (kernel-level) |
| Debugging | Easier (verbose logs) | Harder |

---

## VLAN Segmentation

**What it does:** Divides a physical network into multiple logical networks. Hosts on different VLANs cannot communicate directly — all cross-VLAN traffic must pass through a Layer 3 device (the firewall), which applies access rules.

**Two implementation approaches:**

### Port-based VLANs (used in this lab)
- Each VLAN gets a dedicated physical or virtual interface on the firewall
- No 802.1Q tagging involved
- Simpler to configure; immune to VLAN hopping attacks
- Less scalable (requires a physical port per VLAN)

### 802.1Q Tagged VLANs (industry standard)
- A single trunk port carries multiple VLANs using 802.1Q tags
- Requires a managed switch that understands VLAN tags
- Scalable to dozens/hundreds of VLANs
- Vulnerable to VLAN hopping if trunk ports are misconfigured (DTP enabled, native VLAN not hardened)

**Best for:**
- Separating guest Wi-Fi from corporate network
- Isolating IoT devices, servers, printers into separate broadcast domains
- Compliance requirements (PCI-DSS requires cardholder data network isolation)
- Reducing blast radius of a compromised host

**Limitations:**
- Does not encrypt traffic — isolation only
- Misconfigured firewall rules can accidentally allow inter-VLAN traffic
- Does not prevent hosts from communicating through external/cloud services
- 802.1Q tagged VLANs require managed switch hardware

---

## How the Three Work Together

```
Remote worker needs access to the corporate network:
  └─► OpenVPN — authenticates and encrypts the connection

Two offices need to share internal resources:
  └─► IPsec — connects the two LANs transparently

Inside the corporate network, different departments should be isolated:
  └─► VLANs — segments staff, guests, servers, IoT into separate networks
              with firewall rules controlling what crosses the boundary
```

These three controls operate at different scopes:
- **OpenVPN** = secure the *edge* (who gets in from outside)
- **IPsec** = connect *sites* (how separate locations talk to each other)
- **VLANs** = segment the *interior* (what hosts can reach each other inside)

A complete network security design typically uses all three.

---

## When to Use Each

| Scenario | Use |
|----------|-----|
| Remote worker needs LAN access | OpenVPN |
| Two office sites need shared resources | IPsec site-to-site |
| Guest Wi-Fi should not reach corporate LAN | VLAN |
| IoT devices should not reach server subnet | VLAN |
| Road warrior needs access from hotel/airport | OpenVPN over TCP 443 |
| Cloud VPC needs to connect to on-prem | IPsec |
| PCI compliance requires network isolation | VLAN (mandatory) |
| Full remote office connection | IPsec or OpenVPN site-to-site |
