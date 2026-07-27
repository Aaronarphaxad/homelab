# IPAM & Subnets

This page tracks the current addressing structure of the lab in a public-safe form.

!!! warning "Sanitized values"
    The addresses below preserve the real VLAN numbers, subnet relationships, and system roles, but not the live network and host addresses. The private source of truth is stored separately from this public repository.

## Sanitization scheme

The public documentation uses:

- `192.168.50.0/24` for the upstream residential LAN
- `10.42.0.0/16` for internal lab examples
- `10.42.5.0/24` for WireGuard clients
- generic host identifiers instead of live DHCP addresses
- RFC 5737 documentation ranges when a public internet address is needed in an example

The mapping is intentionally descriptive rather than reversible. It preserves the architecture without functioning as a live network inventory.

## Current subnet plan

| Segment | VLAN | Public-safe subnet | Public-safe gateway | Purpose |
| --- | ---: | ---: | ---: | --- |
| Upstream residential LAN | Untagged | `192.168.50.0/24` | `192.168.50.1` | TELUS router, home Wi-Fi, Proxmox management, and OPNsense WAN |
| Base internal LAN | Untagged | `10.42.0.0/24` | `10.42.0.1` | OPNsense parent LAN and internal transit/base network |
| Management | 10 | `10.42.10.0/24` | `10.42.10.1` | Administrative workloads and management VM |
| Infrastructure | 20 | `10.42.20.0/24` | `10.42.20.1` | Reserved for shared infrastructure and proxy experiments |
| Applications | 30 | `10.42.30.0/24` | `10.42.30.1` | Application workloads such as the PDF service |
| WireGuard | Tunnel | `10.42.5.0/24` | `10.42.5.1` | Authenticated remote-access clients |

## Current public-safe host inventory

| Public name | Platform or role | Segment | Public-safe address | Assignment |
| --- | --- | --- | ---: | --- |
| `upstream-gw-01` | Residential gateway | Upstream LAN | `192.168.50.1` | Router address |
| `pve-01` | Proxmox VE host | Upstream LAN | `192.168.50.10` | Upstream DHCP |
| `fw-01-wan` | OPNsense WAN | Upstream LAN | `192.168.50.2` | Upstream DHCP |
| `fw-01-lan` | OPNsense base LAN | Base internal LAN | `10.42.0.1` | Static |
| `fw-01-mgmt` | OPNsense VLAN 10 gateway | Management | `10.42.10.1` | Static |
| `mgmt-01` | Debian management VM | Management | `10.42.10.10` | DHCP lease or reservation |
| `fw-01-infra` | OPNsense VLAN 20 gateway | Infrastructure | `10.42.20.1` | Static |
| `fw-01-apps` | OPNsense VLAN 30 gateway | Applications | `10.42.30.1` | Static |
| `app-pdf-01` | BentoPDF LXC | Applications | `10.42.30.20` | DHCP lease or reservation |
| `fw-01-wg` | OPNsense WireGuard interface | WireGuard | `10.42.5.1` | Static tunnel address |
| `vpn-client-01` | Primary laptop peer | WireGuard | `10.42.5.10/32` | Static peer address |

No active workload is currently documented in VLAN 20. The planned Nginx, reverse-proxy, and load-balancing lab will use that segment when implemented.

## Current service records

| Public-safe name | Resolves to | Service |
| --- | ---: | --- |
| `pdf-app.home` | `10.42.30.20` | PDF application on a non-standard HTTPS port |
| `firewall.home` | `10.42.5.1` | OPNsense management through WireGuard |

DNS records map names to addresses, not application ports. Until a reverse proxy is installed, a service using a non-standard port still requires the port in its URL.

## Addressing and gateway rules

- OPNsense owns the gateway address in every internal subnet.
- The Proxmox host and OPNsense WAN remain on the upstream residential LAN.
- `vmbr0` carries the upstream network and OPNsense WAN.
- `vmbr1` is the VLAN-aware internal bridge and currently has no physical port.
- The Debian management VM is confirmed to be in VLAN 10.
- The BentoPDF LXC is in VLAN 30.
- WireGuard clients receive individual `/32` peer addresses from the tunnel subnet.
- Client `AllowedIPs` should contain only the exact internal networks that must traverse the VPN.

## Assignment policy

Infrastructure addresses should gradually move from incidental DHCP leases to either:

- DHCP reservations managed centrally
- documented static addresses outside the dynamic pool

Whichever method is selected, the gateway, DNS server, VLAN, and reservation owner should be recorded in the private source of truth.

## Public documentation rules

Safe to publish in sanitized form:

- VLAN numbers and purposes
- subnet relationships
- generic host roles
- gateway conventions
- routing and firewall concepts
- documentation-only addresses

Keep outside the public repository:

- live IP-to-host mappings
- actual public IP addresses and DDNS names
- MAC addresses and DHCP client identifiers
- WireGuard private and preshared keys
- API tokens
- configuration exports
- router serial numbers and recovery information

Files excluded from the MkDocs build are still visible when the Git repository itself is public. The private IPAM must therefore live outside this repository, not merely outside the `docs/` directory.

## Update checklist

When a subnet or host is added:

1. update the private source of truth with its real address and assignment method
2. add a sanitized equivalent here if it improves the public architecture reference
3. update the topology diagram when the relationship between components changes
4. verify that screenshots and Git diffs do not contain keys, tokens, or live public endpoints
