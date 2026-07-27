# WireGuard VPN

This page is based on the WireGuard deployment guide you shared and keeps the original setup logic in a cleaner public-safe form.

## Goal

Provide secure remote access to the internal lab without exposing the Proxmox UI, OPNsense UI, or SSH directly to the public internet.

## Architectural Summary

To enable secure, encrypted remote access to the OPNsense firewall, Proxmox VE host, and all internal VLAN subnets from external networks such as mobile 5G or remote Wi-Fi, a WireGuard remote access VPN tunnel was implemented.

This architecture provides full administrative capability without exposing management web UIs or SSH ports directly to the public internet.

### A. WireGuard Device Assignment & Interface Binding

#### 1. Server Instance Provisioning

Navigate to `VPN -> WireGuard -> Instances` in the OPNsense web UI to create the primary WireGuard server instance.

- Device / Tunnel: `wg0` listening on UDP port `51820`
- Tunnel Subnet: dedicated internal VPN network block defined as `10.10.10.X/24`
- Instance Key Generation: generate a dedicated instance private key and instance public key pair exclusively for the OPNsense server instance

#### 2. Peer Creation & Cryptographic Key Isolation

Navigate to `VPN -> WireGuard -> Peers` to establish remote client configuration profiles.

- Peer Profile: provision a client profile and assign it the static tunnel IP `10.10.10.X/32` under `Allowed IPs`
- Cryptographic Key Isolation Principle: the peer profile must not reuse the server instance key pair
- Generate a separate, unique client key pair
- Save the client public key in OPNsense under the peer settings
- Save the client private key only inside the client device configuration file
- Client `AllowedIPs` Routing: configure `AllowedIPs` for the internal lab ranges, management ranges, or full-tunnel routing if intended

#### 3. Interface Assignment & Activation

Navigate to `Interfaces -> Assignments` in OPNsense and bind the underlying `wg0` virtual device as an assigned logical interface.

Activating this interface allows OPNsense to:

- apply stateful firewall filtering
- handle outbound source NAT translation for VPN client traffic

### B. Upstream Port Forwarding (Double NAT via TELUS Gateway)

Because the upstream TELUS residential gateway operates in routed mode and assigns OPNsense WAN a private address on the `192.168.1.X` subnet, incoming VPN connections traverse a double NAT path.

- Gateway Access: access the upstream TELUS modem portal
- Forwarding Policy Configuration: create a port forwarding rule to relay external VPN traffic to OPNsense

Suggested forwarding values:

- Protocol: UDP
- External Port: `51820`
- Internal Destination IP: `192.168.1.X`
- Internal Destination Port: `51820`

WireGuard relies strictly on UDP. This rule ensures incoming UDP handshake packets hitting the public IP are forwarded directly through the TELUS firewall layer down to OPNsense.

### C. Dynamic DNS Integration (DuckDNS Engine)

Because TELUS assigns dynamic public IP addresses via upstream DHCP, the home public IP periodically changes, which breaks static endpoint domain mappings.

- Functional Purpose: Dynamic DNS monitors the WAN interface IP and automatically updates an external domain record in real time whenever TELUS assigns a new IP
- Plugin Installation: install the `os-ddclient` plugin package via `System -> Firmware -> Plugins`
- DuckDNS Account Provisioning: create a DuckDNS account and generate a dedicated subdomain with an account API token
- OPNsense DDNS Account Configuration:
  - Service: Duck DNS
  - Username: leave blank
  - Password: DuckDNS account API token
  - Hostname: DuckDNS hostname
  - Check IP Method: Interface
  - Interface to Monitor: WAN
- Client Endpoint Mapping: configure the endpoint parameter in the WireGuard client configuration to the DuckDNS hostname and port `51820`

### D. Firewall Policy Set Architecture

To permit WireGuard handshake establishment and allow decrypted tunnel traffic to reach internal network segments, specific rules are required.

#### 1. Inbound WAN Rule

Location: `Firewall -> Rules -> WAN`

- Action: Pass
- Direction: In
- Interface: WAN
- Protocol: UDP
- Destination: WAN address `192.168.1.X`
- Destination Port: `51820`

This allows external encrypted WireGuard connection requests forwarded by the upstream modem to enter the WAN interface and reach the OPNsense WireGuard listening service.

#### 2. WireGuard Traffic Rule

Location: `Firewall -> Rules -> WireGuard / WG_VPN`

- Action: Pass
- Direction: In
- Interface: WireGuard
- Protocol: Any
- Source: WireGuard network `10.10.10.0/24`
- Destination: Any

This permits authenticated VPN clients originating from the `10.10.10.0/24` network to traverse OPNsense and access local subnets, administrative interfaces, and internal VLANs.

### E. Outbound NAT / Source NAT Translation

Location: `Firewall -> NAT -> Source NAT`

- Mode Selection: change NAT mode from automatic source NAT rule generation to hybrid source NAT rule generation
- Rule Configuration:
  - Interface: LAN and target VLAN interfaces
  - TCP/IP Version: IPv4
  - Protocol: Any
  - Source: WireGuard network `10.10.10.0/24`
  - Destination: Any
  - Translation / Target: interface address (masquerade)

Technical justification:

Without source NAT, when a remote VPN client `10.10.10.X` requests access to an internal LAN host like Proxmox `192.168.1.X`, the packet arrives at Proxmox displaying `10.10.10.X` as the source address.

If Proxmox does not have an explicit static route back to `10.10.10.0/24` via OPNsense, or if host firewalls reject non-local subnets, response packets are dropped.

Enabling hybrid source NAT instructs OPNsense to rewrite the packet source IP to OPNsense's own local interface IP such as `192.168.1.X`. Internal hosts then treat the request as coming locally from OPNsense, allowing return packets to route back cleanly through OPNsense's NAT state table to the VPN client.

## Related pages

- [OPNsense firewall](opnsense.md)
- [IPAM & subnets](../architecture/ipam.md)
