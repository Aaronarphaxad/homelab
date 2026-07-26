# OPNsense Firewall

This page captures the step-by-step setup and the main lessons from the first OPNsense deployment in the lab.

## What this service does

OPNsense is the main network boundary for the homelab. It handles:

- WAN to LAN routing
- firewall policy
- NAT
- internal segmentation
- VPN access

## Network design summary

The firewall VM is attached to two Proxmox bridges:

| Interface | Proxmox side | Role |
| --- | --- | --- |
| `net0` / `vtnet0` | `vmbr0` | WAN |
| `net1` / `vtnet1` | `vmbr1` | LAN |

That creates a simple pattern:

- upstream network on the WAN side
- isolated lab traffic on the LAN side
- all routing and policy centralized at OPNsense

## Step-by-step setup

### 1. Create the Proxmox bridge topology

Make sure Proxmox has:

- `vmbr0` for the WAN-facing path
- `vmbr1` for the isolated internal lab network

The internal bridge should not be treated like an external-facing network.

### 2. Create the OPNsense VM

Provision the OPNsense VM with:

- two virtual NICs
- VirtIO adapters
- one NIC mapped to the WAN bridge
- one NIC mapped to the internal bridge

### 3. Assign interfaces in OPNsense

At first boot:

- map the WAN interface to the NIC connected to `vmbr0`
- map the LAN interface to the NIC connected to `vmbr1`
- give LAN its own subnet that is different from WAN

!!! warning "Avoid the same-subnet trap"
    If WAN and LAN end up on the same subnet, management access can break and routing becomes ambiguous very quickly.

This was one of the biggest lessons from the original setup notes. When the LAN side reset onto the same subnet as WAN, it caused a management lockout.

### 4. Verify outbound connectivity

Before adding VLANs or extra rules, confirm:

- the WAN interface has upstream connectivity
- the LAN interface has the expected gateway IP
- internal clients can use the firewall as their gateway
- outbound NAT is working

### 5. Configure outbound NAT

If internal clients can reach the firewall but not the internet:

1. open the NAT section in OPNsense
2. confirm outbound NAT mode is set appropriately
3. apply the rules
4. retry internet access from an internal client

In the original notes, this was a major troubleshooting checkpoint. If the client could reach the firewall but not the public internet, source NAT was one of the first things to verify.

### 6. Add VLAN-backed internal segmentation

Once the base LAN works, create logical internal segments for roles like:

- management
- engineering / infrastructure
- applications

For each VLAN:

1. create the VLAN on the LAN-side parent interface
2. assign and enable the new interface
3. give it a gateway IP
4. enable DHCP if needed
5. add explicit firewall rules

Example purpose-driven structure:

| VLAN | Purpose |
| --- | --- |
| Management | Admin tools, jump boxes, identity services |
| Engineering / Infrastructure | Monitoring, automation, internal services |
| Applications | App workloads, reverse proxies, client-facing experiments |

### 7. Enable DHCP or static assignment deliberately

For each internal segment, decide whether systems should use:

- DHCP
- DHCP reservations
- static assignments

One lesson from the original notes was to make sure the intended interfaces were actually enabled in the DHCP service configuration.

### 8. Add firewall rules deliberately

New interfaces default to deny behavior. For each internal segment, decide:

- what should reach the internet
- what should reach other internal segments
- what should remain isolated

Start with the minimum rules needed to move forward.

### 9. Set up the internal management station

The original guide also included a Debian management VM on the internal network.

Key idea:

- keep a management machine inside the protected network
- let it use OPNsense for DHCP and routing
- administer the firewall from inside the lab when possible

For a simple Debian interface profile:

```bash
auto lo
iface lo inet loopback

auto ens18
iface ens18 inet dhcp
```

### 10. Optional: inbound SSH port forwarding

One documented pattern was forwarding a non-standard external SSH port on OPNsense to an internal management VM.

That approach can work, but it should be used carefully and documented clearly because it is much less private than using WireGuard first.

If I keep this pattern in the lab, I want it to be:

- limited to a specific host
- on a non-default external port
- protected by host hardening and firewall intent
- reviewed alongside the VPN approach

## Common issues

### Same-subnet WAN and LAN

If WAN and LAN share the same subnet, fix that first before debugging anything else.

### LAGG misconfiguration

Trying link aggregation too early can create a full lockout. If that happens:

- return to the OPNsense console
- reassign interfaces
- keep the layout simple again

### Internal clients have no internet

Check this order:

1. client IP and gateway
2. firewall rule on the source interface
3. outbound NAT mode and rules
4. whether changes were actually applied

### DHCP service does not seem to work on a VLAN

Check whether the intended interface is actually enabled in the DHCP configuration. That was one of the issues called out in the original notes.

## Verification checklist

- [ ] WAN and LAN are on different subnets
- [ ] internal clients reach the firewall gateway
- [ ] internal clients reach the internet
- [ ] VLAN interfaces have the intended policies
- [ ] admin access is only available from trusted paths

## Related pages

- [Network & topology](../architecture/network-topology.md)
- [IPAM & subnets](../architecture/ipam.md)
- [Proxmox VE Host Setup](proxmox.md)
- [WireGuard VPN](wireguard-vpn.md)
