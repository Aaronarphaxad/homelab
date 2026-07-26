# Overview

This site is the public documentation for my homelab.

I am using it to track what I build, how I structure the environment, what I learn, and how I troubleshoot along the way.

The goal is to keep it useful for:

- my own future reference
- showing employers how I think and build
- documenting changes in a way that stays simple to maintain

!!! note "Public-safe documentation"
    Live IPs, exact hostnames, domains, credentials, and other sensitive details are intentionally sanitized in these docs.

## Site structure

This site is intentionally split into a few stable sections:

- Architecture for the current state of the environment
- Labs for focused project work
- Infrastructure & Services for step-by-step setup guides
- Journal for dated updates and lessons learned

The navigation is the main source of truth for where pages live, so I do not repeat large page lists here.

## Updating philosophy

To keep this easy to maintain:

- network layout lives in one place
- subnet planning lives in one place
- VM and LXC inventory lives in one place
- dated updates live in one place

That way I do not have to edit multiple pages just to reflect one change.

## Single source of truth

| If I change... | I update... |
| --- | --- |
| topology or network shape | `Architecture -> Network & Topology` |
| subnets or addressing | `Architecture -> IPAM & Subnets` |
| hosts, VMs, or LXCs | `Architecture -> Hardware & Host` |
| service setup steps | the specific page under `Infrastructure & Services` |
| progress or lessons learned | `Journal` |

## Current focus

- Proxmox virtualization
- OPNsense routing and segmentation
- WireGuard remote access
- service organization with VMs, LXCs, and containers
