# Proxmox VE Host Setup

This guide is based on my Proxmox bare-metal setup notes and keeps the install path short, practical, and easy to revisit.

## Goal

Install Proxmox VE on the physical host, get the base network working cleanly, and prepare the node for VM provisioning.

## Host role

This machine is the main virtualization host for the homelab.

It is responsible for:

- running OPNsense
- hosting internal VMs and future LXCs
- providing the bridge layout used by the lab network

## Phase 1: Prepare the installation media

1. download the official Proxmox VE installer ISO
2. write it to a USB drive
3. validate the boot media before moving on

Using a tool like BalenaEtcher is a simple, reliable path here.

## Phase 2: Identify the upstream LAN details

Before installation, confirm the network you are plugging the host into:

- local subnet
- router or default gateway
- whether the environment is friendly to static addressing during first boot

For public docs, I keep these values sanitized rather than publishing live ones.

## Phase 3: BIOS and hardware checks

Before installing:

- enable CPU virtualization
- enable VT-d or equivalent IOMMU support
- disable Secure Boot if it interferes
- confirm which physical NIC is actually cabled

!!! warning "Confirm the correct NIC first"
    If the host has multiple physical network ports, make sure you know which one is connected before you start debugging Proxmox networking.

## Phase 4: Install Proxmox VE

Boot from the installer and complete the normal installation flow.

At this point, the priority is not a perfect network design. The priority is a stable host you can reach reliably.

## Phase 5: Troubleshoot first-boot networking

One of the main issues in my original notes was a static network configuration that looked valid but did not route cleanly through the upstream gateway.

If the host can talk locally but cannot reach the internet:

1. open the Proxmox console
2. inspect `/etc/network/interfaces`
3. if needed, temporarily switch the main bridge from static to DHCP
4. restart networking
5. confirm the new bridge address

Example commands:

```bash
nano /etc/network/interfaces
systemctl restart networking
ip addr show dev vmbr0
ip neigh show
```

## Example bridge pattern

The current lab uses:

| Bridge | Purpose |
| --- | --- |
| `vmbr0` | Upstream/WAN-facing bridge |
| `vmbr1` | Internal lab bridge |

The first install only needs `vmbr0` working well. The rest can come after.

## Why DHCP helped during troubleshooting

In the original setup, switching from a manual static address to DHCP helped confirm whether the problem was:

- the Proxmox network file
- the upstream router policy
- or a simple mismatch between expected and accepted addressing

That was a useful first-pass sanity check before making the design more permanent.

## Router ping caveat

If the host cannot ping the upstream router, that does not always mean the path is broken.

Some gateways drop ICMP. The ARP or neighbor table can be a better test:

```bash
ip neigh show
```

## Phase 6: Post-install cleanup

A fresh Proxmox install often points at enterprise repositories that are not useful on a personal non-subscription lab host.

The cleanup path is:

1. disable or replace the enterprise repository
2. enable the no-subscription or community path
3. update the host
4. confirm the host is ready for VM deployment

## Verification checklist

- [ ] Proxmox boots cleanly
- [ ] the main bridge has a working network path
- [ ] the web UI is reachable
- [ ] package repositories work cleanly
- [ ] the host is ready for OPNsense and other VMs

## Related pages

- [Network & Topology](../architecture/network-topology.md)
- [Hardware & Host](../architecture/hardware.md)
- [OPNsense Firewall](opnsense.md)
