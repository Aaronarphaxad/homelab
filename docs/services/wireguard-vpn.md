# WireGuard VPN

This page is based on the WireGuard deployment guide you shared and keeps the original setup logic in a cleaner public-safe form.

## Goal

Provide secure remote access to the internal VLANs without exposing OPNsense, SSH, or application services directly to the public internet.

For the current learning phase, the Proxmox management UI remains on the trusted TELUS home LAN. This is an intentional convenience and recovery exception: Proxmox is not exposed to the public internet, but it is not yet WireGuard-only.

## Architectural Summary

To enable secure, encrypted remote access to the OPNsense firewall and internal VLAN subnets from external networks such as mobile 5G or remote Wi-Fi, a WireGuard remote access VPN tunnel was implemented.

The only intentionally forwarded public service is the WireGuard UDP listener. Internal web applications and SSH services are reached after the client authenticates to WireGuard.

### A. WireGuard Device Assignment & Interface Binding

#### 1. Server Instance Provisioning

Navigate to `VPN -> WireGuard -> Instances` in the OPNsense web UI to create the primary WireGuard server instance.

- Device / Tunnel: `wg0` listening on UDP port `51820`
- Tunnel Subnet: dedicated internal VPN network block defined as `10.10.10.X/24`
- Instance Key Generation: generate a dedicated instance private key and instance public key pair exclusively for the OPNsense server instance

#### 2. Peer Creation & Cryptographic Key Isolation

Navigate to `VPN -> WireGuard -> Peers` to establish remote client configuration profiles.

- Peer Profile: provision a client profile and assign it the static tunnel IP `10.10.10.X/32` under `Allowed IPs`
- Cryptographic Key Isolation Principle: the peer profile must not reuse the server instance key pair
- Generate a separate, unique client key pair
- Save the client public key in OPNsense under the peer settings
- Save the client private key only inside the client device configuration file
- Client `AllowedIPs` Routing: configure `AllowedIPs` for the internal lab ranges, management ranges, or full-tunnel routing if intended

#### 3. Interface Assignment & Activation

Navigate to `Interfaces -> Assignments` in OPNsense and bind the underlying `wg0` virtual device as an assigned logical interface.

Activating this interface allows OPNsense to:

- apply stateful firewall filtering
- handle outbound source NAT translation for VPN client traffic

### B. Upstream Port Forwarding (Double NAT via TELUS Gateway)

Because the upstream TELUS residential gateway operates in routed mode and assigns OPNsense WAN a private address on the `192.168.1.X` subnet, incoming VPN connections traverse a double NAT path.

- Gateway Access: access the upstream TELUS modem portal
- Forwarding Policy Configuration: create a port forwarding rule to relay external VPN traffic to OPNsense

Suggested forwarding values:

- Protocol: UDP
- External Port: `51820`
- Internal Destination IP: `192.168.1.X`
- Internal Destination Port: `51820`

WireGuard relies strictly on UDP. This rule ensures incoming UDP handshake packets hitting the public IP are forwarded directly through the TELUS firewall layer down to OPNsense.

### C. Dynamic DNS Integration (DuckDNS Engine)

Because TELUS assigns dynamic public IP addresses via upstream DHCP, the home public IP periodically changes, which breaks static endpoint domain mappings.

- Functional Purpose: Dynamic DNS monitors the WAN interface IP and automatically updates an external domain record in real time whenever TELUS assigns a new IP
- Plugin Installation: install the `os-ddclient` plugin package via `System -> Firmware -> Plugins`
- DuckDNS Account Provisioning: create a DuckDNS account and generate a dedicated subdomain with an account API token
- OPNsense DDNS Account Configuration:
  - Service: Duck DNS
  - Username: leave blank
  - Password: DuckDNS account API token
  - Hostname: DuckDNS hostname
  - Check IP Method: Interface
  - Interface to Monitor: WAN
- Client Endpoint Mapping: configure the endpoint parameter in the WireGuard client configuration to the DuckDNS hostname and port `51820`

### D. Firewall Policy Set Architecture

To permit WireGuard handshake establishment and allow decrypted tunnel traffic to reach internal network segments, specific rules are required.

#### 1. Inbound WAN Rule

Location: `Firewall -> Rules -> WAN`

- Action: Pass
- Direction: In
- Interface: WAN
- Protocol: UDP
- Destination: WAN address `192.168.1.X`
- Destination Port: `51820`

This allows external encrypted WireGuard connection requests forwarded by the upstream modem to enter the WAN interface and reach the OPNsense WireGuard listening service.

#### 2. WireGuard Traffic Rule

Location: `Firewall -> Rules -> WireGuard / WG_VPN`

- Action: Pass
- Direction: In
- Interface: WireGuard
- Protocol: Any
- Source: WireGuard network `10.10.10.0/24`
- Destination: Any

This permits authenticated VPN clients originating from the `10.10.10.0/24` network to traverse OPNsense and access local subnets, administrative interfaces, and internal VLANs.

### E. Outbound NAT / Source NAT Translation

WireGuard-to-VLAN traffic does not normally need source NAT. OPNsense is directly connected to both the VPN subnet and the VLAN subnets, so it can route between them while preserving the real VPN client address in logs.

Source NAT is only needed for a separate use case, such as:

- sending full-tunnel VPN internet traffic out through OPNsense WAN
- reaching an upstream host that does not route replies for `10.10.10.0/24` back through OPNsense

The current Proxmox management address is an example of the second case because it remains on the upstream TELUS LAN. It is currently accessed directly from trusted home Wi-Fi instead of being included in the VPN-only design.

## F. Troubleshooting: Remote VPN Worked but Home Wi-Fi Failed

### 1. Initial Symptoms

The original client profile worked when the laptop used a 5G hotspot:

- WireGuard completed a handshake
- OPNsense was reachable through the tunnel
- internal VLAN services were reachable by IP

The same profile failed while the laptop was connected to the home TELUS Wi-Fi. Proxmox remained reachable because both the laptop and Proxmox were directly connected to the upstream `192.168.1.0/24` network, but the WireGuard tunnel did not establish a new handshake.

Testing from another residential Wi-Fi network also failed. This raised two separate concerns:

- the home router might not support NAT loopback
- broad private routes in the client could collide with networks used at other locations

### 2. Why the Public Endpoint Failed at Home

The remote profile uses a dynamic DNS hostname that resolves to the TELUS public IP:

```text
Laptop
  -> TELUS public IP:51820
  -> TELUS UDP port forward
  -> OPNsense WAN:51820
```

When the laptop is already on the TELUS home LAN, using that public address requires the TELUS router to send the connection back inside through NAT loopback, also called hairpin NAT. Support for this behavior was either unavailable or unreliable.

This failure happens on the upstream TELUS router before the connection reaches the internal VLANs. OPNsense NAT reflection cannot repair a packet path that never reaches OPNsense.

### 3. Routing Conflict in the Original Client

The original client routed entire private address families through WireGuard:

```ini
AllowedIPs = 10.0.0.0/8, 192.168.0.0/16, 10.10.10.0/24
```

These ranges are too broad for a roaming client. Hotels, offices, hotspots, and other homes commonly use addresses inside `10.0.0.0/8` or `192.168.0.0/16`.

The client was changed to route only the actual lab networks:

```ini
AllowedIPs = 10.0.10.0/24, 10.0.20.0/24, 10.0.30.0/24, 10.10.10.0/24
```

This minimizes collisions and keeps the VPN in split-tunnel mode. Normal internet traffic continues to use the laptop's current network.

### 4. Options Considered

#### Option A: TELUS NAT Loopback

Continue using the public DuckDNS endpoint everywhere and enable NAT loopback on the TELUS router.

This would provide a single client profile, but the required router feature was not confirmed to be available.

#### Option B: Split-Horizon DNS

Resolve the same VPN hostname differently depending on location:

- at home: resolve it to the private OPNsense WAN address
- away from home: resolve it to the TELUS public address

This would also permit one client profile. However, it requires a local DNS resolver that the laptop can reach before the VPN starts. The tunnel's own DNS server cannot solve this bootstrap problem because it is unavailable until after the handshake.

#### Option C: Separate Home and Remote Profiles

Use two client profiles that share the same client identity and tunnel address but have different endpoints:

```ini
# Remote profile
Endpoint = <dynamic-dns-name>:51820

# Home profile (points to OPNsense WAN IP)
Endpoint = 192.168.1.xxx:51820
```

Only one profile is activated at a time.

This was selected because it is explicit, easy to troubleshoot, and does not depend on an unconfirmed TELUS router feature or additional DNS infrastructure.

### 5. OPNsense WAN Adjustments for the Home Profile

The OPNsense WAN address is private because it receives DHCP from the TELUS router. Under `Interfaces -> WAN`, `Block private networks` must therefore be disabled so the firewall can legitimately receive traffic from a home client such as `192.168.1.xxx`.

A dedicated rule was placed above the general internet WireGuard rule:

| Setting | Value |
| --- | --- |
| Interface | WAN |
| Direction | In |
| Protocol | IPv4 UDP |
| Source | TELUS LAN, for example `192.168.1.0/24` |
| Destination | WAN address |
| Destination port | `51820` |
| Gateway | Default |
| Disable reply-to | Enabled |

!!! The separate rule is needed because OPNsense normally adds `reply-to` behavior to WAN rules. For traffic arriving from another host on the same WAN subnet, this can incorrectly force the reply toward the TELUS default gateway instead of directly back to the laptop.

The general WAN rule remains below it for remote connections:

```text
IPv4 UDP from any source to WAN address port 51820
```

The TELUS router continues to forward public UDP port `51820` to the private OPNsense WAN address for remote access.

### 6. Windows Client Adapter Detour

During testing, the recreated home tunnel briefly failed before any network traffic could be sent:

```text
Unable to create network adapter:
The system cannot find the file specified.
```

This was a local Windows WireGuard adapter/service problem, not an OPNsense failure. Restarting or recreating the local tunnel restored adapter creation.

This distinction is useful:

- an enabled WireGuard adapter only confirms local interface creation
- it does not confirm peer authentication
- a recent handshake is the authoritative connection test

### 7. Handshake Root Cause: Peer Public-Key Mismatch

After the home profile could start, OPNsense showed a new outer UDP firewall state for the laptop's home address. However, the WireGuard status page still showed only an old handshake and old endpoint.

The outer firewall state proved:

- Windows routed the endpoint through Wi-Fi correctly
- the packet reached OPNsense WAN
- the WAN UDP rule matched

It did not prove that WireGuard authenticated the packet. WireGuard silently ignores packets that do not authenticate.

The final issue was a public-key mismatch between the Windows client identity and the registered OPNsense peer. The Windows client public key was corrected to match the single peer registered in OPNsense.

The required key relationships are:

```text
Windows interface public key
  = OPNsense peer public key

Windows peer public key
  = OPNsense WireGuard instance public key
```

The OPNsense peer configuration for this client uses:

```text
Allowed IPs: 10.10.10.x/32
Endpoint address: blank
Endpoint port: blank
```

OPNsense learns the client's current endpoint after a successful authenticated handshake. Both Windows profiles use the same client key pair and `10.10.10.x/32`; only their server endpoint differs.

### 8. Final Forwarding Issue: WireGuard-to-VLAN Rule

Correcting the peer key established a current handshake and made the OPNsense WireGuard address `10.10.10.1` reachable. Internal VLAN gateways and hosts were still unreachable, which separated the problem into two layers:

- the encrypted tunnel and peer authentication were working
- forwarding traffic from the WireGuard interface into the VLANs was not yet permitted correctly

The working diagnostic rule was placed at the top of the assigned WireGuard interface:

| Setting | Value |
| --- | --- |
| Action | Pass |
| Quick | Enabled |
| Interface | Assigned WireGuard interface |
| Direction | In |
| TCP/IP version | IPv4 |
| Protocol | Any |
| Source | `10.10.10.0/24` |
| Destination | Any |
| Gateway | Default |
| Disable reply-to | Enabled |
| Logging | Enabled during testing |

Using the explicit source subnet avoided ambiguity around automatically generated interface aliases. Leaving the gateway as `Default` also prevented policy routing from incorrectly sending internal traffic toward the WAN gateway.

After this rule was applied, the home-internal profile could reach the VLANs successfully through WireGuard.

The broad `Destination: Any` setting is useful while validating the design. A later hardening step is to replace it with a `LAB_NETWORKS` alias containing only:

```text
10.0.10.0/24
10.0.20.0/24
10.0.30.0/24
```

Access to OPNsense itself can then be controlled with separate rules for required services such as DNS and the management UI.

### 9. Final Client Design

Remote profile:

```ini
[Interface]
Address = 10.10.10.2/32
DNS = 10.10.10.1

[Peer]
Endpoint = <dynamic-dns-name>:51820
AllowedIPs = 10.0.10.0/24, 10.0.20.0/24, 10.0.30.0/24, 10.10.10.0/24
PersistentKeepalive = 25
```

Home profile:

```ini
[Interface]
Address = 10.10.10.x/32
DNS = 10.10.10.1

[Peer]
Endpoint = 192.168.1.147:51820
AllowedIPs = 10.0.10.0/24, 10.0.20.0/24, 10.0.30.0/24, 10.10.10.0/24
PersistentKeepalive = 25
```

#### Internal DNS for BentoPDF

The WireGuard client uses OPNsense Unbound at `10.10.10.1` for internal name resolution.

Under `Services -> Unbound DNS -> Overrides`, the BentoPDF host override is:

| Setting | Value |
| --- | --- |
| Host | `bento-pdf` |
| Domain | `home` |
| Type | `A` |
| IP address | `10.0.30.173` |

The override previously pointed to `192.168.1.147`, which is the OPNsense WAN address on the TELUS LAN rather than the BentoPDF application host. It was corrected to the application's VLAN 30 address.

DNS maps the hostname to an IP address but does not include an application port. The current application URL is therefore:

```text
https://bento-pdf.home:8443
```

Use `http://` instead if the application listener on port `8443` is not configured for TLS.

Private keys, preshared keys, the real dynamic DNS hostname, and other credential-adjacent values must never be published in this documentation.

### 10. Troubleshooting Sequence Learned

The most effective order is:

1. Confirm that the WireGuard adapter starts.
2. Check `Latest handshake`; do not treat an enabled adapter as a connection.
3. Use `Find-NetRoute -RemoteIPAddress <address>` on Windows to verify:
   - the WireGuard endpoint uses the physical Wi-Fi interface
   - lab prefixes use the WireGuard interface
4. Check OPNsense WAN states or packet capture for outer UDP 51820 traffic.
5. If outer UDP arrives but there is no handshake, verify client/server public-key relationships and any preshared key.
6. After a current handshake exists, test the WireGuard address, VLAN gateways, and an actual application port separately.
7. Search OPNsense states and live firewall logs for the tunnel source such as `10.10.10.2`.
8. Verify that the pass rule is on the assigned WireGuard interface, uses the explicit VPN source subnet, and leaves the gateway set to `Default`.
9. Only then troubleshoot target host firewalls, default gateways, application listeners, and internal DNS.

### 11. Current Decision and Future Improvement

The current design intentionally keeps Proxmox management on the TELUS home LAN because the lab does not yet contain important data and direct access makes recovery easier while learning.

Consequences of this decision:

- Proxmox is reachable from trusted home Wi-Fi without WireGuard.
- Proxmox is not exposed through a public TELUS port forward.
- internal VLAN services are now confirmed accessible through the home-internal WireGuard profile.
- Proxmox is the documented exception to the goal of VPN-only management.
- remote Proxmox access is not treated as reliable because upstream private addresses commonly overlap with remote networks.

The future hardened design is to move the Proxmox management address to the management VLAN and retain a physical console or dedicated recovery path. That migration is deferred until the additional isolation is worth the reduced recovery convenience.

## Related pages

- [OPNsense firewall](opnsense.md)
- [IPAM & subnets](../architecture/ipam.md)
