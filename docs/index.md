# Homelab Documentation

This site documents the architecture, services, labs, and troubleshooting history of my homelab.

The environment is built on Proxmox VE, with OPNsense providing internal routing, network segmentation, DNS, and WireGuard VPN access.

## Start here

| Area | Description |
| --- | --- |
| [Architecture](architecture/index.md) | Network topology, VLANs, addressing, hardware, and design decisions |
| [Labs](labs/index.md) | Planned and active learning environments |
| [Infrastructure and services](services/index.md) | Installation and configuration notes for the core platform |
| [Journal](journal/index.md) | Dated troubleshooting notes, decisions, and lessons learned |

## Environment overview

- **Compute:** One Dell mini PC running Proxmox VE
- **Firewall and router:** OPNsense virtual machine
- **Network segmentation:** Management, infrastructure, and application VLANs
- **Remote access:** WireGuard VPN
- **Internal DNS:** Unbound DNS overrides
- **Documentation:** MkDocs Material

See the [network topology](architecture/network-topology.md) for the current logical map and [IPAM](architecture/ipam.md) for addressing details.

## Accessing the homelab

Internal lab services are accessed through WireGuard. They are not intended to be published directly to the internet.

### 1. Install WireGuard

Install the official client for the device:

- [Windows](https://download.wireguard.com/windows-client/wireguard-installer.exe)
- [macOS](https://apps.apple.com/us/app/wireguard/id1451685025)
- [Android](https://play.google.com/store/apps/details?id=com.wireguard.android)
- [Linux and other platforms](https://www.wireguard.com/install/)

### 2. Create a peer in OPNsense

Create a separate WireGuard peer for each device. Give every peer its own key pair and tunnel address so it can be revoked without affecting other devices.

### 3. Import the configuration

- On a phone or tablet, scan the QR code generated for the peer.
- On a computer, save or copy the configuration and import it into the WireGuard client.

### 4. Verify access

After activating the tunnel:

1. Confirm that the client shows a recent handshake.
2. Test the WireGuard gateway at `10.10.10.1`.
3. Test an internal service by IP address.
4. Test its internal DNS name.

See [WireGuard VPN](services/wireguard-vpn.md) for the complete setup and troubleshooting notes.

!!! warning "Protect peer configurations"
    A WireGuard QR code or configuration file contains private key material. Do not publish or commit it to the documentation repository. Use one peer per device and revoke peers for lost or retired devices.

## Lab roadmap

| Lab | Status | Intended outcome |
| --- | --- | --- |
| [Linux administration](labs/linux-administration.md) | Planned | Practise storage, LVM, systemd services, permissions, networking, logs, and recovery |
| [Hybrid Active Directory and Entra ID](labs/active-directory.md) | Planned | Build an on-premises identity environment and integrate it with Microsoft Entra ID |
| [CI/CD](labs/ci-cd.md) | Planned | Move a change from source control through testing and into a controlled deployment |
| [Argo CD and Kubernetes](labs/kubernetes.md) | Planned | Learn Kubernetes fundamentals and GitOps-based reconciliation |
| [Monitoring](labs/monitoring.md) | Planned | Collect useful metrics and logs, create dashboards, and design actionable alerts |
| [Automation](labs/automation.md) | Planned | Use Ansible and PowerShell for repeatable configuration and recovery |
| [Nginx reverse proxy and load balancing](labs/nginx-reverse-proxy.md) | Planned | Add internal HTTPS entry points, proxy services, health checks, and load balancing |
| [SIEM](labs/siem.md) | Planned | Centralize security events and practise detection and investigation |

## Documentation approach

The documentation should capture:

- the intended design
- the configuration that was implemented
- how the result was validated
- failures encountered along the way
- why one option was chosen over another
- recovery or rollback steps

This keeps the site useful as both a reference and a record of the learning process.
