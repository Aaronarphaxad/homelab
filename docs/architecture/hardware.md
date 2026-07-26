# Hardware & Hosts

This page tracks the infrastructure inventory that is documented publicly so far.

## Physical host inventory

| Name | Type | Role | Status | Notes |
| --- | --- | --- | --- | --- |
| `pve-01` | Bare-metal host | Proxmox VE hypervisor | Active | Main lab virtualization host |

## Current VM inventory

These are the VMs clearly documented in the current notes.

| Name | Platform | Role | Status | Notes |
| --- | --- | --- | --- | --- |
| `fw-01` | OPNsense VM | Firewall, routing, NAT, WireGuard | Active | Connected to both WAN and internal bridges |
| `mgmt-01` | Debian VM | Internal management workstation | Active | Used for admin work from inside the lab |

## Current LXC inventory

No specific LXC instances are documented in the current public notes yet.

That is intentional here: I would rather list real deployed containers once they exist and have a stable purpose than invent filler entries now.

## Current Docker-host inventory

No Docker host is documented in the current public notes yet.

## How I want to expand this page

As the lab grows, I want to add tables for:

- additional Proxmox nodes
- all VMs
- all LXCs
- Docker hosts
- major services and which segment they live in

## Suggested inventory fields

For each VM or LXC, I want to capture:

- name
- platform or OS
- role
- network segment
- status
- notes

That should keep the inventory useful without becoming over-detailed.
