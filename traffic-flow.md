# Traffic Flow

How traffic moves through the network in each configuration — VPN tunnels and VLAN-segmented infrastructure.

---

## OpenVPN Traffic Flow

### Authentication Flow

```
Client
  │
  ├─► TLS handshake with OpenVPN server
  │       Server presents its certificate
  │       Client verifies against trusted CA
  │
  ├─► Client presents its certificate
  │       Server verifies against same CA
  │
  ├─► Username + password submitted
  │       pfSense User Manager validates credentials
  │
  └─► Tunnel established
          Client assigned IP from VPN pool (e.g., 10.8.0.x)
```

### Data Flow (Post-Authentication)

```
[Remote Client]  ──UDP 1194──►  [pfSense WAN]
                                      │
                               OpenVPN decrypts
                               and unwraps packet
                                      │
                                      ▼
                               [pfSense LAN]  ──►  [Internal Host]

Return path:
[Internal Host]  ──►  [pfSense LAN]
                              │
                        OpenVPN encrypts
                        and wraps packet
                              │
                              ▼
                       [pfSense WAN]  ──UDP 1194──►  [Remote Client]
```

**What the remote client sees:** A new virtual network interface (tun0) with an IP in the VPN tunnel range (10.8.0.x). All traffic destined for the LAN subnet is routed through this interface and encrypted before leaving the machine.

---

## IPsec Site-to-Site Flow

```
Site A LAN                                    Site B LAN
192.168.1.0/24                               192.168.2.0/24

[Host A]  ──►  [pfSense A WAN] ══ IPsec tunnel ══ [pfSense B WAN]  ──►  [Host B]
               Encrypts packet                      Decrypts packet
               Encapsulates in ESP                  Removes ESP header
               Sends to Site B WAN IP               Forwards to Host B

Tunnel negotiation:
  Phase 1 (IKE): establishes a secure channel, authenticates both firewalls
  Phase 2 (IPsec SA): negotiates encryption for actual data traffic

Hosts on each side have no knowledge of the tunnel — they send packets
to the remote subnet as if it were directly reachable.
```

---

## VLAN Traffic Flow

### Port-Based VLAN Architecture (as configured in this lab)

```
                    ┌──────────────────────────────────────┐
                    │           pfSense / OPNsense          │
                    │                                       │
  [Kali - VLAN 10]  │  ┌──────────┐      ┌──────────────┐  │
  192.168.10.x ─────┼─►│  OPT1   │      │     OPT2     │◄─┼─── 192.168.20.x
                    │  │10.1 /24 │      │  .20.1 /24   │  │  [Ubuntu - VLAN 20]
                    │  └────┬─────┘      └──────┬───────┘  │
                    │       │    Firewall        │          │
                    │       │  (default deny     │          │
                    │       │  between VLANs)    │          │
                    │       └────────┬───────────┘          │
                    │                │                       │
                    │           ┌────┴────┐                 │
                    │           │   WAN   │                 │
                    └───────────┴────┬────┴─────────────────┘
                                     │
                                 [Internet]
```

### Inter-VLAN Traffic (Blocked by Default)

```
Kali (192.168.10.15) ──ping──► Ubuntu (192.168.20.25)
        │
        ▼
Packet hits pfSense OPT1 interface
        │
        ▼
Firewall checks rules for OPT1
        │
        ▼
No rule allows traffic to 192.168.20.0/24
        │
        ▼
Default deny → packet dropped
        │
        ▼
Kali receives: Request timeout / Destination unreachable
```

### Internet Access (Allowed by Rule)

```
Kali (192.168.10.15) ──► 8.8.8.8
        │
        ▼
Packet hits pfSense OPT1
        │
        ▼
Firewall rule: Pass, Source=OPT1 subnet, Destination=Any
        │
        ▼
NAT applied (pfSense WAN IP masquerades for client)
        │
        ▼
Packet exits WAN ──► Internet ──► response returns ──► Kali
```

---

## VPN + VLAN Combined Architecture

```
Remote Worker
(OpenVPN client)
      │
      │ encrypted tunnel
      ▼
[pfSense WAN] ──► OpenVPN decrypts ──► assigned 10.8.0.x
                                              │
                              ┌───────────────┼───────────────┐
                              ▼               ▼               ▼
                         [LAN hosts]    [VLAN 10 hosts]  [VLAN 20 hosts]
                         192.168.1.x    192.168.10.x     192.168.20.x
                              │               │               │
                              └───────────────┴───────────────┘
                                    Access controlled by
                                    per-interface firewall rules
```

VPN clients land in the 10.8.0.0/24 tunnel network. Which LAN/VLAN subnets they can reach depends on what is configured in **IPv4 Local Network** in the OpenVPN server settings, and whether firewall rules on each VLAN interface permit traffic from the VPN subnet.

---

## Certificate Chain (OpenVPN)

```
Certificate Authority (CA)
        │
        ├── signs ──► Server Certificate  (installed on pfSense OpenVPN server)
        │
        └── signs ──► User Certificate    (exported in .ovpn file for client)

During TLS handshake:
  Server presents its cert → client verifies signature against CA ✓
  Client presents its cert → server verifies signature against CA ✓
  Both parties confirmed → tunnel proceeds to user auth stage
```
