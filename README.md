# Private Networks Lab

Hands-on lab configuring two core network segmentation and secure access mechanisms — OpenVPN remote access and VLAN-based network segmentation — on a pfSense/OPNsense firewall in an isolated virtual lab environment.

> All configuration was performed on a self-hosted firewall VM within a private lab network. No production systems were involved.

---

## Topics Covered

- OpenVPN server setup with certificate-based authentication
- Certificate Authority (CA) creation and certificate signing
- VPN client export and connection testing
- IPsec site-to-site tunnel configuration
- VLAN creation and interface assignment on pfSense
- Firewall rules for inter-VLAN traffic control
- DHCP server configuration per VLAN
- Port-based VLAN isolation vs 802.1Q tagged VLANs

---

## Tools & Technologies

| Tool | Purpose |
|------|---------|
| pfSense / OPNsense | Firewall, VPN server, VLAN gateway |
| OpenVPN | Remote access VPN |
| IPsec | Site-to-site network tunnel |
| openvpn-client-export | pfSense package for .ovpn file generation |
| Kali Linux VM | VLAN 10 test client |
| Ubuntu VM | VLAN 20 test client |

---

## Environment

- **Firewall:** pfSense/OPNsense (VM)
- **Client machines:** Kali Linux, Ubuntu, Windows VM
- **Network:** Isolated internal LAN + VLAN segments
- **Virtualization:** VMware

---

## Lab Walkthrough

### Activity 1 – VPN Configuration

#### Part A — OpenVPN (Remote Access)

**Goal:** Set up an OpenVPN server on pfSense that allows remote clients to authenticate and access the internal LAN securely.

---

##### Step 1 — Install the Client Export Package

1. Go to **System > Package Manager > Available Packages**.
2. Search for `openvpn-client-export` and click **Install**.
3. Wait for the success confirmation.

This package adds a **Client Export** tab to the OpenVPN menu, allowing you to generate ready-to-use `.ovpn` configuration files for clients.

---

##### Step 2 — Create a Certificate Authority (CA)

A CA is the trust anchor for the VPN — it signs all certificates used by the server and clients, allowing each side to verify the other's identity.

1. Go to **System > Cert. Manager > CAs > Add**.
2. Select **Create an internal Certificate Authority**.
3. Fill in the descriptive name and leave defaults for key length and algorithm.
4. Click **Save**.

---

##### Step 3 — Create Server and Client Certificates

1. Go to **System > Cert. Manager > Certificates > Add/Sign**.
2. First certificate:
   - Method: `Create an internal Certificate`
   - Type: `Server Certificate`
   - Certificate Authority: select the CA created above
   - Click **Save**
3. Second certificate (repeat):
   - Method: `Create an internal Certificate`
   - Type: `User Certificate`
   - Click **Save**

---

##### Step 4 — Configure the OpenVPN Server

1. Go to **VPN > OpenVPN > Servers > Add**.
2. Configure:

   | Setting | Value |
   |---------|-------|
   | Server Mode | Remote Access (SSL/TLS + User Auth) |
   | Protocol | UDP on IPv4 only |
   | Interface | WAN |
   | Local Port | 1194 |
   | TLS Authentication | Enabled |
   | Peer Certificate Authority | Your CA |
   | IPv4 Tunnel Network | `10.8.0.0/24` (private range for VPN clients) |
   | IPv4 Local Network | Your LAN subnet (e.g., `192.168.1.0/24`) |

3. Click **Save**.

---

##### Step 5 — Assign Certificate to User

1. Go to **System > User Manager**.
2. Select or create the user who will connect via VPN.
3. Scroll to **User Certificates > Add**.
4. Method: `Choose existing certificate` → select the user certificate created in Step 3.
5. Save.

---

##### Step 6 — Firewall Rule for VPN Traffic

1. Go to **Firewall > Rules > OpenVPN tab**.
2. Click **Add**:
   - Action: `Pass`
   - Interface: `OpenVPN`
   - Protocol: `Any`
   - Source: `any`
   - Destination: `any`
3. Save and **Apply Changes**.

---

##### Step 7 — Export and Test

1. Go to **VPN > OpenVPN > Client Export**.
2. Find your server and user, click **Inline Configurations: Most Clients** to download the `.ovpn` file.
3. On the client machine, install the OpenVPN client and import the `.ovpn` file.
4. Connect — enter username and password when prompted.
5. Verify connection: the OpenVPN icon turns green.
6. In pfSense, go to **VPN > OpenVPN > Status** to confirm the client appears as connected.

**Key takeaway:** OpenVPN uses a combination of TLS certificates (for server/client identity verification) and username/password (for user authentication). Both factors must succeed for the connection to be established — this is why it's called SSL/TLS + User Auth mode.

---

#### Part B — IPsec (Site-to-Site)

**Goal:** Connect two separate networks through an encrypted tunnel so hosts on each side can reach each other as if on the same LAN.

##### Step 1 — Create Phase 1 (IKE)

1. Go to **VPN > IPsec > Add P1**.
2. Select **IKEv2** (preferred) or IKEv1 for older device compatibility.
3. Configure authentication — pre-shared key (PSK) for simplicity, or certificates for higher security.
4. Set the remote gateway IP (the other firewall's WAN IP).

##### Step 2 — Configure Phase 2 (Traffic Selectors)

1. Add a P2 entry under the P1 tunnel.
2. Set **local network** (your LAN subnet) and **remote network** (the other site's LAN subnet).
3. Enable **Network-to-Network** mode.
4. Save and apply.

##### Step 3 — Firewall Rules

1. Go to **Firewall > Rules > IPsec tab**.
2. Add a Pass rule allowing traffic between the two subnets.

##### Step 4 — Test

From either network, ping a host on the other side. Successful replies confirm the tunnel is up and routing correctly.

**Key takeaway:** IPsec operates at Layer 3 and is transparent to end hosts — they have no idea their traffic is being tunneled. Unlike OpenVPN (which requires client software), IPsec is built into most OSes and network devices, making it the standard for site-to-site connections.

---

### Activity 2 – VLAN Implementation

**Goal:** Segment the internal network into isolated VLANs, each with its own subnet and DHCP range, with firewall rules controlling inter-VLAN traffic.

**Approach used:** Port-based VLAN via dedicated virtual interfaces (one VMware adapter per VLAN), rather than a single trunk port with 802.1Q tags. This provides equivalent logical isolation while eliminating VLAN hopping attack vectors that exist in 802.1Q trunk configurations.

---

#### Step 1 — Add Virtual Adapters

In VMware, add two additional network adapters to the pfSense VM and set both to **Host-Only** mode. These will become the VLAN interfaces.

---

#### Step 2 — Assign Interfaces in pfSense

1. Go to **Interfaces > Assignments**.
2. From the **Available network ports** dropdown, select the new adapter (e.g., `em2`) and click **Add**.
3. Repeat for the second adapter.
4. Go to each new interface (OPT1, OPT2) and configure:
   - Enable the interface
   - IPv4 Configuration Type: `Static IPv4`
   - IPv4 Address: gateway IP for that VLAN (e.g., `192.168.10.1/24` for VLAN 10, `192.168.20.1/24` for VLAN 20)
5. Save and **Apply Changes**.

---

#### Step 3 — Configure DHCP for Each VLAN

1. Go to **Services > DHCP Server**.
2. Click the **OPT1** tab:
   - Enable DHCP server on OPT1
   - Address Pool: `192.168.10.10` to `192.168.10.100`
   - (Addresses `.1` to `.9` reserved for static/server use)
3. Save.
4. Repeat for **OPT2** tab using range `192.168.20.10` to `192.168.20.100`.

---

#### Step 4 — Firewall Rules per VLAN

1. Go to **Firewall > Rules** and click the **OPT1** tab.
2. Add rules as needed:
   - Allow internet access: Action=Pass, Source=OPT1 subnet, Destination=Any
   - Block inter-VLAN: do NOT add a rule allowing OPT1 to reach OPT2 subnet (default deny handles this)
3. Repeat for **OPT2**.
4. Save and **Apply Changes**.

---

#### Step 5 — Connect Test Clients

1. In VMware, change the Kali Linux VM's network adapter to the same **Host-Only** network as OPT1.
2. Change the Ubuntu VM's adapter to the OPT2 Host-Only network.
3. On each VM, run `ifconfig` (or `ip a`) to confirm they received DHCP addresses in their respective subnets.

---

#### Step 6 — Verify Segmentation

```bash
# From Kali (VLAN 10):
ping <Ubuntu-IP>      # Should FAIL — different VLAN, no inter-VLAN rule

# From Ubuntu (VLAN 20):
ping <Kali-IP>        # Should FAIL — same reason

# Both should reach internet (if internet access rule added)
ping 8.8.8.8          # Should SUCCEED
```

Successful ping failure between VLANs confirms segmentation is working. Hosts are isolated at Layer 3 by the firewall's default deny policy.

**Key takeaway:** VLANs create logical network boundaries. Without an explicit firewall rule allowing inter-VLAN traffic, hosts on different VLANs cannot communicate — even if they are physically on the same switch or hypervisor. All routing between VLANs goes through the firewall, giving you full visibility and control over cross-segment traffic.

---

## Summary

| Activity | Mechanism | Purpose |
|----------|-----------|---------|
| OpenVPN | SSL/TLS + user auth tunnel | Secure remote access to LAN |
| IPsec | Encrypted Layer 3 tunnel | Site-to-site network bridging |
| VLAN (port-based) | Dedicated virtual interfaces | Network segmentation and isolation |
| Firewall rules | Per-interface ACLs | Control inter-VLAN and VPN traffic |

---

## Disclaimer

All configuration documented here was performed in a controlled, isolated lab environment for educational purposes only. Do not apply these techniques to networks you do not own or have explicit written permission to administer.
