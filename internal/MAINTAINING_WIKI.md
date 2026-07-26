# Maintaining the Homelab Wiki

This file is for site maintenance notes only.

It is outside the `docs/` folder, so it will not show up in the published MkDocs site.

Do not put secrets here if the repository itself is public.

## Main rule

Update the page that owns the information.

Do not repeat the same information across multiple public pages unless there is a strong reason.

## Single source of truth

| Type of change | Update this file |
| --- | --- |
| Network shape, bridges, VLAN layout | `docs/architecture/network-topology.md` |
| Subnets, gateways, sanitized IP plan | `docs/architecture/ipam.md` |
| Physical hosts, VMs, LXCs, Docker hosts | `docs/architecture/hardware.md` |
| Proxmox setup steps | `docs/services/proxmox.md` |
| OPNsense setup steps | `docs/services/opnsense.md` |
| WireGuard setup steps | `docs/services/wireguard-vpn.md` |
| General LXC/container deployment patterns | `docs/services/lxc-containers.md` |
| Short dated update or lesson learned | `docs/journal/` |
| New lab work | matching file in `docs/labs/` |

## What not to keep updating manually

These pages should stay mostly stable:

- `docs/index.md`
- `docs/architecture/index.md`
- `docs/services/index.md`
- `docs/labs/index.md`
- `docs/journal/index.md`

Only change those if the structure of the site changes.

## Public update workflow

### If you add a new VM or LXC

1. update `docs/architecture/hardware.md`
2. if it changes addressing, also update `docs/architecture/ipam.md`
3. if it needs a dated note, add a journal entry

### If you change the network design

1. update `docs/architecture/network-topology.md`
2. update `docs/architecture/ipam.md` if subnet or gateway structure changed
3. update the relevant service guide only if the setup steps changed

### If you learn a better setup process

1. update the relevant service guide
2. add a journal entry only if the change is worth recording historically

### If you start a new lab

1. create or update the matching page under `docs/labs/`
2. add it to `mkdocs.yml` only if it is a brand-new page

## Navigation maintenance

You should only need to update `mkdocs.yml` when:

- adding a brand-new page
- renaming a page
- moving a page to a different section

You should not need to touch `mkdocs.yml` for normal content edits.

## Journal vs changelog

Use the journal only.

There is no separate public changelog anymore because it creates duplicate update tracking.

## Writing rules

- keep public docs sanitized
- keep one owner page per topic
- avoid repeating inventories in service pages
- avoid repeating subnet tables in multiple places
- use the journal for dated notes, not every minor detail
