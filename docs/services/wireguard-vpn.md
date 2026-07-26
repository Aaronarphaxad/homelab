# WireGuard VPN

This page is based on the WireGuard deployment guide you shared and keeps the original setup logic in a cleaner public-safe form.

## Goal

Provide secure remote access to the internal lab without exposing the Proxmox UI, OPNsense UI, or SSH directly to the public internet.

## Architecture summary

The VPN design is meant to:

- terminate on OPNsense
- give remote clients access to internal lab subnets
- keep management interfaces off the public internet
- support changing public IPs through dynamic DNS if needed

## Step-by-step setup

### 1. Create the WireGuard instance on OPNsense

In OPNsense:

1. go to the WireGuard instances section
2. create the main server instance
3. choose the listening port
4. define the VPN subnet
5. generate the server key pair

Use sanitized documentation values in public notes rather than publishing live ones.

### 2. Create a peer for each client

For each remote device:

1. create a peer entry
2. assign a client tunnel IP
3. generate a separate client key pair
4. store the public key on the firewall
5. keep the private key only on the client device

!!! warning "Do not reuse keys"
    The server and every client should have separate key pairs.

The original notes also called out a practical routing detail: the client `AllowedIPs` setting should reflect what you actually want the client to send through the tunnel.

### 3. Assign the WireGuard interface

In OPNsense:

1. assign the WireGuard tunnel as an interface
2. enable it
3. give it a clear name

This makes firewall policy easier to manage and troubleshoot.

### 4. Handle upstream port forwarding if the firewall is behind another router

If the firewall sits behind an upstream residential gateway, incoming VPN traffic may need to cross a double-NAT path.

In that case:

1. forward the WireGuard UDP port on the upstream router
2. point it at the OPNsense WAN address
3. verify the firewall is actually receiving the handshake traffic

### 5. Add the WAN-side rule for the handshake

Allow the WireGuard UDP port on the OPNsense WAN side so the firewall can receive VPN connection attempts.

### 6. Add the WireGuard interface rule

On the WireGuard interface:

1. allow traffic from the VPN client subnet
2. limit destination scope as needed
3. keep the rule broad only if that matches the intended access model

The original setup used a broad rule so authenticated VPN clients could reach internal services and VLANs.

### 7. Configure client routing

Choose what the client should send through the tunnel:

- only internal lab ranges
- all internal plus management networks
- full-tunnel traffic if that is intentional

Document the intent clearly so later troubleshooting is easier.

### 8. Configure outbound NAT if return traffic needs help

The original notes also documented a hybrid outbound NAT approach for WireGuard traffic.

Why this can matter:

- a remote VPN client may reach an internal host
- the internal host may not know how to route return traffic back to the VPN subnet
- source NAT can make the flow return cleanly through OPNsense

If this applies in the lab:

1. switch outbound NAT to the required mode
2. add a rule for the VPN client subnet
3. translate through the appropriate interface address
4. test access to internal services again

### 9. Configure DNS for VPN clients

If I want internal names to resolve over VPN, the client should use the internal DNS resolver through the tunnel.

### 10. Handle dynamic public IP changes

If the home connection changes public IPs:

1. set up dynamic DNS
2. point the WireGuard client endpoint to the DDNS name
3. verify the record updates when the WAN IP changes

This was part of the original guide because the upstream WAN address was not guaranteed to stay fixed.

## Troubleshooting order

If the VPN does not work, check in this order:

1. client config
2. key pair matching
3. upstream port forward
4. WAN UDP rule
5. WireGuard interface rule
6. return path or NAT behavior
7. DNS expectations

## Verification checklist

- [ ] handshake succeeds from an external network
- [ ] remote clients can reach intended internal services
- [ ] internal DNS works if configured
- [ ] admin interfaces remain private behind the VPN
- [ ] no live keys or endpoints are published in docs

## Related pages

- [OPNsense firewall](opnsense.md)
- [IPAM & subnets](../architecture/ipam.md)
