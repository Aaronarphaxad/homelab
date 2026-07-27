# OPNsense Firewall

This document outlines the architectural design, configuration steps, critical troubleshooting benchmarks, and core networking principles discovered during the deployment of an OPNsense virtual firewall and a Debian minimal management workstation inside a Proxmox VE environment.

## 1. Network Architecture & Bridge Topology

To establish a secure, firewalled perimeter for the internal lab network, the virtual environment utilizes two distinct Linux network bridges.

This topology physically and logically isolates internal sandbox traffic from the primary residential local area network.

| Bridge Interface | Network Type | IP Assignment Strategy | Purpose |
| --- | --- | --- | --- |
| `vmbr0` | External WAN | DHCP via upstream router on `192.168.1.X` | Bridges the firewall to the physical network card and the internet gateway |
| `vmbr1` | Isolated LAN | Static IP on firewall `10.0.0.X/24` | Acts as a purely virtual switch for internal lab machines with no physical ports assigned |

## 2. OPNsense Provisioning & Interface Mapping

The OPNsense virtual machine acts as the perimeter guard.

In Proxmox, the VM is provisioned with two network interface cards configured under the VirtIO driver model for high throughput.

### Interface Mapping Layout

- `vtnet0` (`net0` in Proxmox): bound to `vmbr0` and configured as the WAN interface, automatically pulling an upstream IP address via DHCP
- `vtnet1` (`net1` in Proxmox): bound to `vmbr1` and configured as the LAN gateway interface, assigned the static IP address `10.0.0.X/24`

### The Same-Subnet Routing Conflict Trap

A critical learning point occurred when OPNsense reverted to factory defaults, mapping its LAN to `192.168.1.X/24` while the WAN interface sat on the identical `192.168.1.X` network via DHCP.

This state creates an asymmetric routing deadlock where the kernel cannot determine which interface should process local traffic, causing a total management lockout.

Correcting the LAN interface to an independent IP block such as `10.0.0.X/24` resolves the routing collision.

## 3. Core Pitfalls & Troubleshooting Solutions

### A. The Link Aggregation (LAGG) Lockout

A `lagg` network refers to Link Aggregation.

It combines multiple physical network cables or ports into a single logical connection and can provide:

- fault tolerance
- increased bandwidth

Accidentally initializing a LAGG interface bundle without pre-configuring LACP or failover mechanisms on the corresponding Proxmox virtual switch disrupts the logical link layer.

The operating system kernel may immediately register:

```text
ping: sendto: Network is down
```

Recovery procedure:

1. use the raw text-based OPNsense serial console
2. select `Assign interfaces`
3. explicitly decline LAGG and VLAN creation
4. re-bind `vtnet0` and `vtnet1` to their respective WAN and LAN wrappers

### B. Source NAT (Outbound Translation) Verification

When local clients can ping the internal gateway `10.0.0.X` but fail to reach public nodes such as `1.1.1.1`, the network address translation engine is typically the first thing to verify.

Navigate to `Firewall -> NAT -> Source NAT` and confirm the mode is configured correctly.

After making changes, apply them so the translation rules are injected into the packet filter state table.

## 4. Debian Management Station Configuration

To establish an isolated terminal inside the lab capable of administrative control, a minimal Debian instance was mapped to `vmbr1`.

### A. Network Profile Standardization

Rather than relying on persistent manual runtime address injection, the interface configuration should rely on the OPNsense DHCP daemon.

Example interface profile:

```bash
auto lo
iface lo inet loopback

auto ens18
iface ens18 inet dhcp
```

### B. Resolving the Minimal Installation Repository Lockout

When executing a network-isolated minimal installation, APT may remain locked to the installation media and cause package lookup failures.

Correction script pattern:

```bash
echo -e "deb http://deb.debian.org/debian/ trixie main non-free-firmware
deb http://security.debian.org/debian-security trixie-security main non-free-firmware
deb http://deb.debian.org/debian/ trixie-updates main non-free-firmware" > /etc/apt/sources.list
```

Following a mandatory index sync via `apt update`, the desktop presentation layer and display manager stack can be downloaded cleanly:

```bash
apt install task-xfce-desktop lightdm -y
```

## 5. Operational State Verification Checklist

- verify OPNsense console presents independent subnet structures for WAN and LAN
- ensure the internal network packet filter is operational using native state filtering rules
- confirm firewall dashboard access originates natively from the isolated workstation terminal interface

High-level path:

```text
Internet
  |
TELUS Router
  |
vmbr0 (WAN)
  |
OPNsense WAN
  |
OPNsense LAN (vtnet1)
  |
vmbr1 (VLAN-aware virtual switch)
  |
|- VLAN10
|- VLAN20
|- VLAN30
```

## Corporate Department Segmentations (VLAN Creation Architecture)

To replicate an enterprise corporate infrastructure layout, the internal bridge network maps traffic to isolated department subnets.

This prevents cross-department traffic contamination by separating services at the firewall layer.

### A. Interface Device Generation Summary

Virtual 802.1Q devices are layered directly on top of the parent internal LAN interface under `Interfaces -> Other Types -> VLAN`.

- VLAN 10: IT & Operations / Management
- VLAN 20: Engineering & Core Products
- VLAN 30: Applications / Client Facing

### B. Subnet Gateway Assignments

Each virtual device is then explicitly bound and activated under `Interfaces -> Assignments` as a standalone routing point.

- `IT_OPS`: static address `10.0.10.X/24`
- `ENGINEERING`: static address `10.0.20.X/24`
- `APPS`: static address `10.0.30.X/24`

### C. Modernized Dynamic Pool Handlers (DNSMasq & DHCP)

Due to the deprecation of the traditional ISC DHCP suite, automated addressing policies are executed through the modern DHCP engine under the DNSMasq and DHCP section.

One issue in the original setup was that the intended VLAN interface was not added to the list of enabled interfaces under the general DHCP configuration.

General fix pattern:

1. add the required interfaces to the enabled list
2. add the DHCP ranges for each VLAN

### D. Initial Firewall Isolation Policy

By default, new network assignments maintain an implicit deny policy.

To allow out-of-band updates, add an explicit outbound permission rule on the relevant interface:

- Action: Pass
- Direction: In
- Protocol: Any
- Source: interface network
- Destination: Any

This enables internet traversal while keeping inter-department lateral movement more controlled.

## SSH access from Home Network

## Inbound Management Port Forwarding

### Architectural Summary

To enable native SSH access into the isolated IT & Operations management VM `10.0.10.X` from a physical device residing on the external network, a destination NAT port forward policy was implemented.

Because OPNsense drops all outside traffic by default, destination NAT intercepts specific incoming traffic hitting the WAN interface, modifies the packet's receiving IP and port, and securely forwards it to the target system inside the private network.

### Step-by-Step Implementation Pipeline

#### 1. Host-Level Service Provisioning

- install the `openssh-server` daemon on the target Debian VM
- verify service persistence

#### 2. OPNsense Destination NAT Engine Mapping

Navigate to the destination NAT section in the OPNsense web UI and create a rule using values like:

- Interface: WAN
- Protocol: TCP
- Destination Address: WAN address
- Destination Port: custom external port `2222`
- Redirect Target IP: `10.0.10.X`
- Redirect Target Port: `22`
- Filter Rule Association: automatic NAT rule association

#### 3. External Verification

Verify from an external client using a command like:

```bash
ssh -p 2222 your_username@192.168.1.X
```

## Deployment & Local Domain Mapping Guide

This procedure outlines how to deploy a new LXC container or application into an isolated VLAN, assign it a fixed network identity, and configure Unbound DNS so it can be accessed using custom local domain names while connected to the WireGuard VPN.

### Step 1: Provision & Map to the Correct VLAN Switch

1. deploy the LXC container or virtual machine
2. by default, newly provisioned containers may bind to `vmbr0`
3. reconfigure the network interface to the internal bridge `vmbr1`
4. assign the correct VLAN tag
5. restart the workload so it requests an address on the correct subnet, such as `10.0.30.X`

### Step 2: Establish a Static DHCP Lease in OPNsense

To prevent DNS records from breaking due to IP rotation, bind the container MAC address to a fixed IP.

1. open the DHCP service for the target VLAN interface
2. add a static mapping
3. assign the MAC address, target IP, and hostname
4. save and apply changes

Example target IP: `10.0.30.X`

### Step 3: Create an Unbound Host Override

Map the assigned IP address to a human-readable local domain.

1. navigate to the DNS override section
2. add a host override
3. set the short host name, domain suffix, record type, and IP
4. save and apply changes

Example target IP: `10.0.30.X`

### Step 4: Verify VPN DNS Resolution

To ensure remote WireGuard clients can resolve internal names:

1. configure the WireGuard client to use the internal DNS resolver
2. ensure the DNS service listens on the WireGuard-reachable interface
3. test name resolution while connected through WireGuard

Example client configuration:

```ini
[Interface]
DNS = 10.10.10.X
```

Example verification:

```bash
nslookup app-name.home
```

Then test the service in a browser using its internal name and port.

## Official Framework Documentation Resources

For structural reference, baseline configuration models can be cross-verified using the official vendor validation manuals:

- [OPNsense Core Interface Configurations](https://docs.opnsense.org/manual/interfaces.html)
- [OPNsense 802.1Q VLAN Logical Foundations](https://docs.opnsense.org/manual/how-tos/vlan_background.html)

## Related pages

- [Network & topology](../architecture/network-topology.md)
- [IPAM & subnets](../architecture/ipam.md)
- [Proxmox VE Host Setup](proxmox.md)
- [WireGuard VPN](wireguard-vpn.md)
