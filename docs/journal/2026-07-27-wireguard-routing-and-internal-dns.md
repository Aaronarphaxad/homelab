# 2026-07-27 - WireGuard, routing, and internal DNS

## Goal

Make the internal lab VLANs reachable through WireGuard from:

- a laptop on the home Wi-Fi
- a laptop using a 5G hotspot
- a laptop on another external Wi-Fi network

The intended security model is for internal VLAN services to be reachable through WireGuard without publishing their web or SSH ports to the internet.

For now, Proxmox management remains directly accessible from the trusted TELUS home LAN. This is an intentional recovery convenience while the lab is still experimental, not the final hardened design.

## Starting problem

Remote access worked over a 5G hotspot, but the same WireGuard profile failed on the home Wi-Fi. Proxmox remained reachable because its management address was on the same upstream network as the laptop, while the isolated VLANs were available only through OPNsense.

This initially looked like a VLAN or firewall problem. It was actually a sequence of independent problems at different layers:

1. the home client could not reliably use the public WireGuard endpoint
2. broad client routes could overlap with networks at other locations
3. the recreated client did not authenticate because its public key did not match the registered peer
4. after authentication worked, the WireGuard interface still needed a correct forwarding rule for the VLANs

## Hairpin NAT and the home endpoint

The remote WireGuard profile connects to a dynamic DNS name that resolves to the TELUS public IP. From an external network, the path is:

```text
Laptop
  -> public internet
  -> TELUS public IP and UDP 51820 port forward
  -> OPNsense WAN
  -> WireGuard
```

Using that same public address from inside the TELUS home network depends on NAT loopback, also called hairpin NAT. The upstream router must accept a connection from its own LAN to its public address and send it back inside through the port-forward.

Rather than depend on an unconfirmed router feature, I created two WireGuard profiles:

- a remote profile using the dynamic DNS endpoint
- a home-internal profile using the private OPNsense WAN address

Both profiles represent the same client. They use the same client key pair, tunnel address, DNS server, and internal routes. Only the endpoint changes, and only one profile is active at a time.

## `AllowedIPs` is also a routing decision

The original client configuration routed entire private address families through WireGuard:

```ini
AllowedIPs = 10.0.0.0/8, 192.168.0.0/16
```

That is risky for a roaming client because other homes, hotels, and workplaces commonly use those ranges. WireGuard treats `AllowedIPs` as routing information on send and peer authorization information on receive.

I narrowed the client routes to only the networks that belong to the lab:

```ini
AllowedIPs = 10.0.10.0/24, 10.0.20.0/24, 10.0.30.0/24, 10.10.10.0/24
```

This keeps the connection split-tunnel: lab traffic uses WireGuard while normal internet traffic continues to use the current local network.

## OPNsense WAN behavior on a private upstream network

The OPNsense WAN interface receives a private address from the TELUS router. Two OPNsense behaviors mattered for direct home-LAN WireGuard traffic:

- `Block private networks` had to be disabled on WAN because private source addresses are legitimate in this double-NAT design.
- The home WireGuard WAN rule needed `Disable reply-to`.

OPNsense normally adds `reply-to` behavior to WAN rules so responses leave through the configured WAN gateway. That is useful for normal internet traffic and multi-WAN systems, but it can misroute a reply when the sender is another device on the same private WAN subnet.

The dedicated home rule therefore permits UDP 51820 from the TELUS LAN to the OPNsense WAN address and disables `reply-to`. The general internet-facing UDP 51820 rule remains below it.

## An active adapter is not a successful VPN connection

At one point, Windows showed the WireGuard adapter with its tunnel address, but OPNsense showed no recent handshake.

This taught me to distinguish several kinds of evidence:

| Evidence | What it proves |
| --- | --- |
| WireGuard adapter exists | Windows created a local interface |
| Windows route selects WireGuard | The packet is directed toward the tunnel |
| OPNsense WAN state exists | Outer UDP traffic reached and matched the firewall |
| Recent WireGuard handshake | The two peers authenticated successfully |
| State with the tunnel source address | Decrypted traffic entered firewall processing |
| Application port responds | Routing, firewalling, return path, and service are working |

A WAN state for the laptop did not prove that WireGuard accepted the packet. WireGuard silently ignores traffic that fails cryptographic authentication.

## Public-key relationships

The missing handshake was caused by a public-key mismatch after recreating the Windows profile.

The correct relationships are:

```text
Windows interface public key
  = OPNsense peer public key

Windows peer public key
  = OPNsense WireGuard instance public key
```

The OPNsense road-warrior peer has the client's single tunnel address as its allowed IP. Its endpoint is left blank because OPNsense learns the client's current source address after an authenticated handshake.

Fixing the key mismatch established a current handshake and made the OPNsense WireGuard address reachable.

## Reaching the firewall is different from forwarding through it

After the handshake worked, I could ping the WireGuard address on OPNsense but not the VLAN hosts. That meant:

- client routing worked
- WireGuard encryption and authentication worked
- traffic addressed to the firewall worked
- forwarding from WireGuard into the VLANs still did not

The working rule was placed on the assigned WireGuard interface with:

```text
Source: 10.10.10.0/24
Destination: any during validation
Gateway: default
Disable reply-to: enabled
```

Using the explicit VPN subnet avoided ambiguity around generated aliases. Leaving the gateway at `Default` prevented policy routing from sending internal traffic toward the WAN.

The broad destination was useful for diagnosis. A future hardening step is to replace it with a `LAB_NETWORKS` alias containing only the approved VLAN networks and to use separate rules for services hosted by OPNsense itself.

## Routed VLAN traffic does not normally need NAT

OPNsense is directly connected to both the WireGuard network and the VLAN networks, so it already knows the return routes. Normal VPN-to-VLAN traffic can be routed without source NAT.

Preserving the real VPN client address also produces better logs and enables per-client policy later.

Source NAT is useful for different cases, such as:

- full-tunnel clients sending internet traffic through OPNsense WAN
- reaching an upstream device that has no route back to the WireGuard subnet

It should not be added merely because an internal firewall rule is missing.

## Unbound host overrides and application ports

Once IP access worked, I corrected the Unbound override:

```text
bento-pdf.home -> 10.0.30.173
```

The earlier override pointed to the private OPNsense WAN address rather than the BentoPDF application host.

I also learned that DNS maps a name to an address; it does not map a normal web URL to an arbitrary port. The direct application URL is therefore:

```text
https://bento-pdf.home:8443
```

The WireGuard client uses the OPNsense WireGuard address as its DNS server, and Unbound must listen for and permit queries from the VPN subnet.

## Why a reverse proxy is the next useful lab

A reverse proxy can accept a friendly URL on standard HTTP or HTTPS ports and forward the request to an application's actual address and port:

```text
https://bento-pdf.home
  -> Nginx on TCP 443
  -> BentoPDF on 10.0.30.173:8443
```

This creates a practical future lab for:

- Nginx virtual hosts
- reverse proxying
- Apache as a backend web server
- request and forwarding headers
- internal TLS
- access and error logs
- multiple backends and load balancing
- restricted inter-VLAN firewall rules

The planned design places the proxy in the infrastructure VLAN and application backends in the applications VLAN. That forces proxy-to-backend traffic through OPNsense, making the firewall policy visible and testable.

## Troubleshooting approach I want to keep

The most useful lesson was to test the path in layers instead of changing routing, NAT, DNS, and firewall rules simultaneously:

1. Verify that the local adapter starts.
2. Confirm the endpoint uses the physical network rather than the tunnel.
3. Look for a recent WireGuard handshake.
4. Test the WireGuard interface address.
5. Test each VLAN gateway.
6. Test the real application port instead of relying only on ping.
7. Follow outer and inner addresses separately in OPNsense states and logs.
8. Change one layer at a time and retest.

This approach turned each result into evidence and avoided hiding a firewall problem behind unnecessary NAT.

## Documentation improvement

I replaced the static topology placeholder with an embedded Mermaid flowchart. Keeping the diagram as text means the logical map can evolve alongside the lab without redrawing an image manually.

## Follow-up

- replace the temporary broad WireGuard destination with a `LAB_NETWORKS` alias
- build the Nginx, Apache, reverse-proxy, and load-balancing mini lab
- add internal HTTPS with a trusted private CA or a domain suitable for DNS-based certificate validation
- reconsider moving Proxmox management into VLAN 10 after a dependable recovery path exists
- continue updating the Mermaid topology as services and physical links are added

## Related pages

- [Network topology](../architecture/network-topology.md)
- [OPNsense firewall](../services/opnsense.md)
- [WireGuard VPN](../services/wireguard-vpn.md)
- [IPAM and subnets](../architecture/ipam.md)
