## Network Deployment & Troubleshooting Lab

## Project Overview

This project simulates the deployment of a segmented small-enterprise network in Cisco Packet Tracer. The network was designed with separate Admin, Operations, IT, and Management VLANs, with inter-VLAN routing, DHCP, internal DNS, secure SSH management, and an ASA firewall providing PAT and connectivity to a simulated ISP and external network.

I created this project to better understand what I studied for the Network + certification.  This really helped cement what I learned and provided great practice with device configuration on the Cisco ISO.  

## Network Topology

![MCorp Network Topology](img/network-topology.png)

## Project Requirements & Design

I wanted this project to require departmental segmentation, centralized network services, secure device management, and controlled Internet connectivity.  I used the following features:

- Separate Admin, Operations, and IT systems using VLAN segmentation.
- A dedicated management VLAN for network infrastructure.
- Enabled communication between VLANs through router-on-a-stick inter-VLAN routing.
- Dynamically assigned client addressing through centralized DHCP services.
- Internal DNS resolution for network resources.
- Secure remote management of network switches using SSHv2.
- An ASA firewall between the internal network and the simulated ISP.
- Outbound connectivity for internal clients using PAT.

- ### VLAN Addressing Plan

| VLAN | Name | Network | Default Gateway |
|------|------|---------|-----------------|
| 10 | Admin | 192.168.10.0/24 | 192.168.10.1 |
| 20 | Operations | 192.168.20.0/24 | 192.168.20.1 |
| 30 | IT | 192.168.30.0/24 | 192.168.30.1 |
| 99 | Management | 192.168.99.0/24 | 192.168.99.1 |

### Infrastructure Addressing

| Device / Interface | IP Address | Purpose |
|--------------------|------------|---------|
| R1 → FW1 | 10.0.0.1/30 | Internal transit link |
| FW1 Inside | 10.0.0.2/30 | Internal firewall interface |
| FW1 Outside | 203.0.113.1/30 | ISP-facing firewall interface |
| ISP-R1 | 203.0.113.2/30 | Simulated ISP gateway |
| ISP-R1 External LAN | 198.51.100.1/24 | External network gateway |
| INTERNET-SRV | 198.51.100.10/24 | Simulated Internet host |
| SW1 Management | 192.168.99.2/24 | SSH management |
| SW2 Management | 192.168.99.3/24 | SSH management |
| DNS-SRV | 192.168.30.5/24 | Internal DNS |

## Layer 2 Switching & VLAN Segmentation

I segmented the internal network into four VLANs to separate departmental traffic and provide a dedicated network for infrastructure management.

I created VLANs 10, 20, 30, and 99 on both switches. I configured end-device switchports as access ports and assigned them to their appropriate VLANs.

I configured 802.1Q trunk links between R1 and SW1 and between SW1 and SW2, allowing traffic from all four VLANs to traverse the shared links.

| VLAN | Department / Purpose |
|------|----------------------|
| 10 | Admin |
| 20 | Operations |
| 30 | IT |
| 99 | Network Management |

### Trunk Links

| Link | Purpose | Allowed VLANs |
|------|---------|---------------|
| R1 ↔ SW1 | Router-on-a-stick / inter-VLAN routing | 10, 20, 30, 99 |
| SW1 ↔ SW2 | VLAN transport between switches | 10, 20, 30, 99 |

### Verification

VLAN membership was verified using `show vlan brief`, confirming that access ports were assigned to their intended VLANs.

![VLAN Verification](img/vlan-verification.png)

802.1Q trunk operation and allowed VLANs were verified using `show interfaces trunk`.

![Trunk Verification](img/trunk-verification.png)

### Troubleshooting: VLAN and Trunk Connectivity

During initial deployment, VLAN traffic was not passing as expected across the switch infrastructure. Verification of the trunk and access-port configurations revealed configuration issues affecting VLAN connectivity.

I used `show vlan`, `show interfaces trunk`, and `show mac address-table` to verify VLAN membership, trunk status, and learned MAC addresses. Access ports were explicitly configured in access mode and the trunk configuration was corrected to carry the required VLANs.

After correcting the configurations, VLAN membership and trunk operation were revalidated before continuing with Layer 3 configuration.

## Inter-VLAN Routing & DHCP

R1 provides inter-VLAN routing using a router-on-a-stick configuration. I created subinterfaces for each VLAN and used 802.1Q tagging so the VLANs could share the physical connection between R1 and SW1.

The router also acts as the DHCP server for the Admin, Operations, and IT VLANs. Each DHCP pool provides clients with an IP address, subnet mask, and the correct default gateway for its VLAN.

### Router Subinterfaces

| Subinterface | VLAN | Gateway |
|---|---:|---|
| Gi0/0.10 | 10 - Admin | 192.168.10.1/24 |
| Gi0/0.20 | 20 - Operations | 192.168.20.1/24 |
| Gi0/0.30 | 30 - IT | 192.168.30.1/24 |
| Gi0/0.99 | 99 - Management | 192.168.99.1/24 |

### DHCP

I configured DHCP pools on R1 for the three user VLANs and excluded the first several addresses in each subnet so they could be reserved for gateways, servers, or other infrastructure.

I verified DHCP operation with `show ip dhcp binding` and checked the addressing received by the client PCs.
