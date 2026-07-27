# Proxmox VE Bare-Metal Setup & Troubleshooting Guide

This document outlines the step-by-step process of installing and configuring Proxmox VE on a bare-metal physical node, troubleshooting initial network routing limitations, and preparing the host for virtualization.

## Phase 1: Preparing the Installation Media

To deploy Proxmox VE, write the hypervisor image to a physical USB drive.

1. Download the latest official Proxmox VE installer ISO
2. Flash the media to an 8 GB or larger USB drive

BalenaEtcher is a simple option for creating and validating the installer media.

## Phase 2: Identifying the Local Network (LAN) Topology

Before installation, identify the local router or gateway IP so the server is placed on the correct subnet.

### Finding your network parameters from a daily device

- On macOS: hold the `Option` key and click the Wi-Fi icon in the menu bar
- On Windows: run `ipconfig` and look for the IPv4 address and default gateway
- On mobile: open the Wi-Fi connection details and find the router or gateway IP

Example topology:

- Local Subnet: `192.168.1.0/24`
- Router / Gateway IP: `192.168.1.X`

## Phase 3: Hardware Provisioning & Installation

### 1. BIOS Configurations

- access the BIOS during boot
- enable CPU virtualization
- enable `VT-d` or equivalent directed I/O support
- disable Secure Boot if it interferes with the Proxmox kernel

### 2. Booting the Installer

- insert the bootable USB drive
- open the one-time boot menu
- boot via UEFI

### 3. Selecting the Network Interface

If the hardware contains multiple physical ports:

- identify the default onboard interface
- confirm the physical Ethernet cable is connected to the intended port

## Phase 4: Troubleshooting Network Isolation (Static vs DHCP)

During a standard installation, setting a manual static IP can sometimes lead to routing isolation if the upstream router restricts manual IP assignments.

If the Proxmox host can access the local network but fails to ping the WAN or internet, switching the interface from static to DHCP can let the router assign a validated lease.

### Steps to switch Proxmox from Static IP to DHCP

1. log in to the physical Proxmox console or SSH session
2. open the Debian network configuration file:

```bash
nano /etc/network/interfaces
```

3. modify the configuration by commenting out the static lines and changing the bridge interface `vmbr0` from `static` to `dhcp`

Before:

```bash
iface lo inet loopback

iface nic1 inet manual

auto vmbr0
iface vmbr0 inet static
        address 192.168.1.X/24
        gateway 192.168.1.X
        bridge-ports nic1
        bridge-stp off
        bridge-fd 0
```

After:

```bash
auto lo
iface lo inet loopback

iface nic1 inet manual

auto vmbr0
iface vmbr0 inet dhcp
#       address 192.168.1.X/24
#       gateway 192.168.1.X
        bridge-ports nic1
        bridge-stp off
        bridge-fd 0
```

4. save and exit the file
5. reload the network stack:

```bash
systemctl restart networking
```

6. check the newly assigned bridge IP:

```bash
ip addr show dev vmbr0
```

The Proxmox web UI can then be reached at:

```text
https://<NEW_IP_ADDRESS>:8006
```

### Tip: Why can't I ping my router?

Some residential gateways drop ICMP by default.

If the router does not answer pings, check the ARP or neighbor table instead:

```bash
ip neigh show
```

If the router appears reachable there, the physical and logical local connection may still be working correctly.

## Phase 5: Post-Install Optimization & Package Repository Fix

By default, Proxmox may attempt to use enterprise repositories that are not useful on a non-subscription lab host.

To transition to the free community update path and optimize the system:

1. open the shell of the main node in the Proxmox web UI
2. run the post-install helper script or manually switch to the no-subscription repository path

Example helper script:

```bash
bash -c "$(wget -qLO - https://github.com/community-scripts/ProxmoxVE/raw/main/misc/post-pve-install.sh)"
```

Key optimizations performed:

- disables the default paid enterprise repository
- enables the free no-subscription repository
- disables the no active subscription warning pop-up on login
- updates core OS packages and system utilities

Your hypervisor is then ready for VM provisioning.

## Related pages

- [Network & Topology](../architecture/network-topology.md)
- [Hardware & Host](../architecture/hardware.md)
- [OPNsense Firewall](opnsense.md)
