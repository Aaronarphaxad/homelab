# Linux Administration

## Status

Planned

## Goal

Build confidence administering Linux systems without depending on a desktop interface or memorized commands. The lab should focus on understanding how the operating system starts, stores data, runs services, records events, and recovers from failures.

## Proposed environment

- two small Debian or Ubuntu Server virtual machines
- one additional virtual disk for storage exercises
- snapshots before destructive or recovery exercises
- access through the management VLAN and WireGuard VPN
- optional Ansible control node after the manual exercises are understood

## Learning areas

### Storage and LVM

- inspect disks, partitions, filesystems, and mount points
- create physical volumes, volume groups, and logical volumes
- create and mount filesystems
- make mounts persistent with `/etc/fstab`
- extend a logical volume and filesystem
- add a disk to an existing volume group
- understand thin provisioning and snapshots
- recover from an incorrect or unavailable mount

### Services and systemd

- inspect units with `systemctl`
- start, stop, restart, enable, and disable services
- understand unit dependencies and targets
- create a simple custom service unit
- use environment files and service overrides
- investigate a failed service
- understand the difference between a service, process, and daemon

### Users, groups, and permissions

- create and manage users and groups
- understand ownership and standard permission bits
- practise `sudo` delegation
- use setuid, setgid, and the sticky bit in a controlled exercise
- work with access control lists
- manage SSH keys and remove password-based access where appropriate

### Processes and resource management

- inspect running processes and process trees
- send signals and stop failed processes safely
- identify CPU, memory, disk, and I/O pressure
- understand foreground jobs, background jobs, and sessions
- use systemd resource limits
- schedule work with cron and systemd timers

### Networking

- inspect addresses, routes, gateways, and listening ports
- configure a static address
- troubleshoot DNS resolution
- test connectivity at each network layer
- understand host firewalls and basic rule management
- use tools such as `ip`, `ss`, `dig`, `curl`, and `tcpdump`

### Packages and updates

- install, remove, and inspect packages
- understand repositories and signing keys
- apply updates safely
- determine which package owns a file
- identify services that require a restart
- document a simple patching and rollback procedure

### Logs and troubleshooting

- use `journalctl` and traditional log files
- filter logs by service, boot, time, and severity
- inspect boot failures
- trace a problem from symptom to evidence
- document the root cause, fix, and validation

### Backup and recovery

- back up configuration and application data
- restore individual files
- recover from a broken `/etc/fstab` entry
- reset access from a recovery environment
- compare VM snapshots with application-aware backups
- write and test a basic recovery checklist

## Suggested exercises

1. Add a virtual disk and build an LVM-backed data volume.
2. Extend the logical volume without rebuilding the server.
3. Create a small web service and manage it with a custom systemd unit.
4. Break the service configuration, diagnose it from logs, and restore it.
5. Create an administrative group with limited `sudo` permissions.
6. Schedule a backup using a systemd timer.
7. Introduce a DNS or route problem and troubleshoot it methodically.
8. Restore the server or its data from the documented backup.
9. Rebuild the configuration with Ansible after completing it manually.

## Completion criteria

The lab is complete when I can:

- explain the Linux boot and service-management path
- add and expand storage safely
- diagnose a failed service using logs
- manage users and permissions without granting unnecessary access
- troubleshoot basic network and DNS problems
- patch the system and identify restart requirements
- recover configuration or data using a tested procedure
- reproduce the core build from documentation

## Notes to document during the lab

- commands used and what each one proves
- diagrams of the LVM storage layers
- expected output for validation checks
- failure scenarios and recovery steps
- differences between Debian and other distributions
- tasks that should later be automated
