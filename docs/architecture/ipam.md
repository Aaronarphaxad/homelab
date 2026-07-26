# IPAM & Subnets

This page tracks the lab's addressing plan in a public-safe way.

!!! warning "Sanitized values"
    The ranges and examples below are intentionally sanitized. They show the structure of the environment, not the exact live values.

## Addressing approach

I want the lab to stay easy to reason about, so each major area gets its own purpose-driven subnet.

## Example subnet plan

| Segment | Sanitized example | Purpose |
| --- | --- | --- |
| Upstream WAN | `198.51.100.0/24` | Documentation-only example for the upstream side |
| VPN clients | `10.42.5.0/24` | Remote clients connecting through WireGuard |
| Core lab LAN | `10.42.0.0/24` | Base internal lab network |
| Management VLAN | `10.42.10.0/24` | Admin tools, firewall access, jump-box style services |
| Engineering VLAN | `10.42.20.0/24` | Internal tooling, monitoring, automation, shared services |
| Applications VLAN | `10.42.30.0/24` | App workloads, proxies, and client-facing experiments |

## Gateway pattern

Each internal subnet should have:

- a clearly documented gateway
- either DHCP reservations or a documented static plan
- a naming convention that stays readable later

## Naming guidance

Instead of publishing real names, I use patterns like:

- `pve-01`
- `fw-01`
- `mgmt-01`
- `lxc-monitoring-01`
- `svc-proxy-01`

## Public-safe documentation habits

When I update this page, I want to avoid including:

- real public endpoints
- real internal management addresses
- real DDNS names
- real DNS overrides
- anything credential-adjacent

## Update checklist

When a new subnet or service network is added, I want to record:

- subnet purpose
- sanitized example range
- gateway role
- DHCP or static approach
- what kind of systems live there
