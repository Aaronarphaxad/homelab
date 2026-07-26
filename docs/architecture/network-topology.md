# Network & Topology

This page is the simple architecture reference for the lab.

## Diagram

I will add a network image here later.

## High-level flow

```text
Internet
  |
Upstream residential gateway
  |
Proxmox host
  |
vmbr0 -> OPNsense WAN
vmbr1 -> OPNsense LAN
  |
VLAN-aware internal network
  |
|- Management
|- Engineering / Infrastructure
|- Applications
  |
Remote access through WireGuard
```

## Topology notes

### Proxmox bridges

| Bridge | Purpose |
| --- | --- |
| `vmbr0` | WAN-facing bridge connected to the upstream network |
| `vmbr1` | Internal bridge used for lab traffic and VLAN-backed services |

### OPNsense role

OPNsense sits between the external network and the internal lab. It handles:

- routing
- firewall rules
- NAT
- DHCP and DNS where needed
- WireGuard remote access

### Internal segmentation

The internal side is split by function so the lab stays easier to manage and safer to expand:

| Segment | Purpose |
| --- | --- |
| Management | Admin access, infrastructure control, jump-box style tools |
| Engineering / Infrastructure | Shared internal services, monitoring, automation, platform services |
| Applications | App workloads, reverse proxies, and public-style experiments |

## Design rules I want to keep

- keep WAN and LAN on different subnets
- add new services to the internal side, not the WAN side
- use VLANs only when they help organization or isolation
- keep remote admin access behind VPN instead of exposing management ports directly

## Related pages

- [IPAM & subnets](ipam.md)
- [Hardware & hosts](hardware.md)
- [OPNsense firewall](../services/opnsense.md)
- [WireGuard VPN](../services/wireguard-vpn.md)
