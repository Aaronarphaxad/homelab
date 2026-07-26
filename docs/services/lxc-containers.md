# LXC & Docker Services

This page is for service inventory and lightweight deployment notes for LXCs and container-hosted workloads.

## Scope

This page is for deployment patterns and service-specific notes.

The actual host and VM/LXC inventory lives in:

- [Hardware & Hosts](../architecture/hardware.md)

## Step-by-step deployment pattern

This section is based on the deployment and local domain mapping notes from the networking guide.

### 1. Place the workload on the correct internal bridge

When creating a new LXC or VM:

1. attach it to the internal bridge rather than the WAN-facing bridge
2. assign the correct VLAN tag if the service belongs to a segmented network
3. boot or restart it so it can request an address on the correct segment

### 2. Give it a stable identity

If the service needs a predictable address:

1. create a DHCP reservation or static mapping
2. record the hostname
3. make sure the service lands in the right subnet

### 3. Add local DNS mapping

If the service should be reachable by name:

1. create a host override or equivalent internal DNS record
2. map the service name to the reserved address
3. document the intended access path

### 4. Verify remote name resolution over VPN if needed

If you want the service reachable by internal name from outside the house:

1. make sure WireGuard clients use the internal DNS resolver
2. make sure the DNS service listens on the VPN-reachable interface
3. test the name while connected through VPN

## How I want to use this page later

When I start adding LXCs and Docker services, I want to track:

- deployment pattern changes
- service-specific quirks
- container access patterns
- DNS and VLAN notes that apply to services broadly

## Basic deployment checklist

When adding a new service, I want to confirm:

1. it is attached to the correct bridge
2. it has the correct VLAN tag if needed
3. it gets a stable IP or reservation
4. it has DNS or naming notes if needed
5. it does not expose management ports publicly by accident

## Related pages

- [Hardware & hosts](../architecture/hardware.md)
- [OPNsense firewall](opnsense.md)
- [WireGuard VPN](wireguard-vpn.md)
