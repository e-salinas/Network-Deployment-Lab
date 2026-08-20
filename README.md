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

I verified VLAN membership using `show vlan brief`, confirming that access ports were assigned to their intended VLANs.

![VLAN Verification](img/VLANS_created_SW1.png)

![VLAN Verification](img/ADMIN_PC_added_VLAN10.png)

The 802.1Q trunk operation and allowed VLANs I verified using `show interfaces trunk`.

![Trunk Verification](img/Trunk_config_SW1G:02.png)

![Trunk Verification](img/Trunk_interface_SW1.png)


## Inter-VLAN Routing & DHCP

R1 provides inter-VLAN routing using a router-on-a-stick configuration. I created subinterfaces for each VLAN and used 802.1Q tagging so the VLANs could share the physical connection between R1 and SW1.

![Subinterfaces](img/Router_Subinterface_0.10_created&verified.png)
![Subinterfaces](img/Router_Verification_subinterfaces_up.png)

### Router Subinterfaces

| Subinterface | VLAN | Gateway |
|---|---:|---|
| Gi0/0.10 | 10 - Admin | 192.168.10.1/24 |
| Gi0/0.20 | 20 - Operations | 192.168.20.1/24 |
| Gi0/0.30 | 30 - IT | 192.168.30.1/24 |
| Gi0/0.99 | 99 - Management | 192.168.99.1/24 |

### DHCP

The router also acts as the DHCP server for the Admin, Operations, and IT VLANs. Each DHCP pool provides clients with an IP address, subnet mask, and the correct default gateway for its VLAN.

I configured DHCP pools on R1 for the three user VLANs and excluded the first several addresses in each subnet so they could be reserved for other infrastructure.

I verified DHCP operation with `show ip dhcp binding` and checked the addressing received by the client PCs.

![DHCP Config](img/DHCP_config.png)
![DHCP Bindings](img/DHCP_bindings.png)

### Troubleshooting a DHCP Gateway Problem

One of the Admin clients received an IP address through DHCP but could not communicate outside its local network.

I checked the DHCP configuration and found that I had entered the Admin VLAN's default gateway as `102.168.10.1` instead of `192.168.10.1`.

![DHCP Troubleshoot](img/Noticed_Typo_DefaultGateway.png)

After correcting the DHCP pool and renewing the client configuration, the PC was able to reach its gateway and communicate with hosts in the other VLANs.
This was a simple typo, but it was a useful troubleshooting exercise.

## Internal DNS

I added an internal DNS server at `192.168.30.5` in the IT VLAN. The server provides name resolution for internal resources using the `mcorp.local` domain.

I created a DNS record for:

- `dns.mcorp.local` → `192.168.30.5`

The DHCP pools on R1 were configured to provide the DNS server address to clients so they could use DNS without having to configure it manually.
![DHCP Troubleshoot](img/DNS_.png)
![DHCP Troubleshoot](img/DNS_config.png)

## Management VLAN & SSH

I created VLAN 99 as a dedicated management VLAN for the network switches. This keeps switch management separate from the Admin, Operations, and IT user VLANs.

Because SW1 and SW2 are Layer 2 switches, I configured a Switch Virtual Interface (SVI) on each switch to provide an IP address for remote management.

| Device | Management IP | Default Gateway |
|---|---|---|
| SW1 | 192.168.99.2/24 | 192.168.99.1 |
| SW2 | 192.168.99.3/24 | 192.168.99.1 |

R1 provides the Layer 3 gateway for the management VLAN through its `Gi0/0.99` subinterface at `192.168.99.1`. The switch SVIs provide management connectivity but do not perform the inter-VLAN routing.

### Management SVI

I configured the VLAN 99 SVI on each switch and verified that the interfaces were up/up.

![Management SVI](img/SW1_SVI_Vlan99.png)

### SSH Configuration

I configured SSH so the switches could be managed remotely.

The SSH configuration included:

- A device hostname and local domain name
- 2048-bit RSA keys
- SSH version 2
- A local administrative user
- Local authentication on the VTY lines
- SSH-only inbound VTY access
![SSH](img/SSH_1.png)
![SSH](img/SSH_v2.png)

All VTY lines were configured with `login local` and `transport input ssh`, requiring authentication against the switch's local user database and preventing Telnet access.

I was successfully able to log in.
![SSH](img/SSH_login_succesful.png)


## Firewall & Internet Edge

I extended the lab by adding a Cisco ASA firewall, a simulated ISP router, and an external server.

The firewall sits between R1 and the ISP and separates the internal network from the simulated Internet.


### Point-to-Point Transit Links

| Connection | Network | Addresses |
|---|---|---|
| R1 ↔ FW1 | 10.0.0.0/30 | R1: 10.0.0.1, FW1: 10.0.0.2 |
| FW1 ↔ ISP-R1 | 203.0.113.0/30 | FW1: 203.0.113.1, ISP-R1: 203.0.113.2 |

I used /30 networks for the point-to-point transit links because only two usable addresses were needed on each link.

![ISP-Router](img/ISP_R1.png)


### Simulated External Network

The `198.51.100.0/24` network represents a network outside of MCorp and provides a destination for testing end-to-end connectivity through the firewall and ISP.

| Device | Address |
|---|---|
| ISP-R1 | 198.51.100.1/24 |
| INTERNET-SRV | 198.51.100.10/24 |


### Routing

R1 uses a default route pointing toward FW1 for traffic destined outside the internal VLANs.

FW1 has static routes back to the internal VLAN networks through R1 and a default route toward ISP-R1.

This provides a complete path from the internal VLANs through the firewall while also giving return traffic a route back to the correct internal networks.

I used `show route` on the ASA to verify the internal static routes, directly connected networks, and default route toward the ISP.

![ASA Routing Table](img/FW1_Inside_static_routes.png)
![ASA Routing Table](img/FW1_Default_outside_Gateway.png)


### PAT

I configured Port Address Translation (PAT) on FW1 for outbound traffic.

Each internal VLAN was defined as a network object and configured for dynamic translation from the ASA inside interface to the outside interface address.

This allows multiple internal hosts to share FW1's outside address of `203.0.113.1` when communicating with the simulated external network.

![PAT](img/FW1_PAT_VLANS.png)
![PAT](img/PAT_config.png)


### Troubleshooting: PAT and ICMP

After configuring the routes and PAT, the internal PCs failed to ping the ISP router at `203.0.113.2`.

I checked the path one section at a time. R1 could reach FW1, FW1 could reach ISP-R1, and the routing tables looked correct. I verified the PAT configuration was present.  The `show xlate` was not showing an active translation when the client ping failed.

After doing some research, I suspected the problem might be related to ICMP inspection on the ASA in Packet Tracer. I checked the firewall policy configuration and there was no active global inspection policy.

I configured an inspection class for the default traffic, added ICMP inspection to the global policy, and applied the policy globally:

![TroubleICMP](img/Class_Inspection.png)
![TroubleshootICMP](img/Policy-map_icmp.png)
![TroubleshootICMP](img/Successful_ping_ISP_R1.png)
![TroubleshootICMP](img/Show_xlate_PAT.png)
