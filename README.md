# swan-vpn

A small bash tool that manages **IKEv2 split-tunnel VPNs using a pre-shared key
plus EAP (username/password)** with strongSwan's `swanctl`, on Linux with
`systemd-resolved`.

It exists because the vendor client (FortiClient for Linux) could not use the
pre-shared key we were given, and NetworkManager's strongSwan plugin cannot
express this kind of gateway at all. See [Why this exists](#why-this-exists).

```
swan-vpn add <name> --gateway HOST --user USER --psk-file FILE --remote-ts CIDR[,CIDR] [--dns-domains D[,D]]
swan-vpn up <name>
swan-vpn down <name>
swan-vpn status
swan-vpn list
swan-vpn remove <name>
```

## Why this exists

The gateway is a FortiGate configured for IKEv2 with:

- **server authentication by pre-shared key** (the gateway proves itself with the PSK),
- **client authentication by EAP-MSCHAPv2** (a username and password),
- a virtual IP and DNS server handed out through IKEv2 configuration payloads.

We were given the PSK as plain text and needed a working client on openSUSE.

### FortiClient: stored PSK cannot be decrypted

FortiClient 7.4.3 (the `el7` RPM, running on openSUSE) never established a tunnel
for this profile. Every attempt got through `IKE_SA_INIT` (so the gateway was
reachable and the proposals matched) and then failed locally, before anything
was sent to the gateway:

```
ikev2_init_auth: failed to decrypt pre-shared key
sa_free: failed to initiate IKE_AUTH exchange
```

What we established:

- It is **not a wrong-PSK problem.** A wrong PSK produces an authentication
  failure *from the server*. Here the client daemon (`iked`) gave up while
  preparing the AUTH payload.
- FortiClient stores the PSK encrypted (`EncX <hex>`). That value failed to
  decrypt. We then imported a profile through the GUI's XML option with the PSK in
  plain text; FortiClient wrote it unencrypted into the generated `iked.conf`,
  and `iked` **still** reported the same decrypt failure. So neither the encrypted
  nor the plain-text form worked.
- The PSK file itself was clean (printable ASCII, no CRLF, no stray whitespace).
- The failure predates any edit: the daemon log showed it on every attempt, and
  the log contained no successful IPsec tunnel at all.
- `forticlient-cli vpn edit` has no PSK prompt, and the GUI only offered
  SSL-VPN or XML when creating a profile, so there was no supported way to fix
  the profile from the client side.

Fixing this would have meant patching FortiClient's internals, so we stopped
there.

### NetworkManager: cannot do PSK

`NetworkManager-strongswan` only offers certificate, SSH-agent, smartcard and EAP
authentication. Its EAP mode expects the *gateway* to authenticate with a
certificate, which this gateway does not use, and there is no PSK mode and no
"server PSK + client EAP" combination. So NetworkManager was ruled out, not
merely inconvenient.

### strongSwan can do it, but only through `swanctl`

strongSwan itself supports exactly this combination:

```
local  { auth = eap; id = USER; eap_id = USER }
remote { auth = psk; id = <gateway ip> }
```

In the logs of a successful run you can see the flow: the PSK check on the
gateway's AUTH payload succeeds, the gateway requests an EAP identity, then
EAP-MSCHAPv2, and the client finishes with its own AUTH. We also considered a
two-round variant (client PSK followed by EAP) because the FortiClient log
showed it computing a PSK AUTH first, but the gateway accepted the simple form,
so it is not needed.

The remaining problems were not authentication. They were about making the
tunnel useful on a normal desktop, which is what the rest of this tool does.

## Design decisions (and the dead ends behind them)

### 1. Traffic never matched the IPsec policy: firewalld masquerade

After the first successful handshake the tunnel was established but passed no
traffic: the SA counters stayed at 0 bytes and the XFRM policy had 0 hits. There
was no `tcpdump` available, so we used a small `AF_PACKET` sniffer on the
Wi-Fi interface and saw the pings leaving as plain ICMP with the *physical*
source address, not the tunnel's virtual IP.

Cause: firewalld's `public` zone had masquerade enabled for everything leaving
the interface. NAT happens before the XFRM policy lookup, so the virtual IP was
rewritten to the LAN address and no longer matched the policy.

Two attempts:

- An early `accept` rule for the virtual IP in an earlier NAT chain **did not** help.
- An explicit `snat to <virtual ip>` rule did.

That worked, but it is a firewall workaround. The design below removes the need
for it.

### 2. Full tunnel broke other things: split tunnel

The first configuration used `0.0.0.0/0` as the remote traffic selector. With
that, **all** IPv4 traffic went through the gateway, and the gateway's upstream
blocked some of it: HTTPS to Slack never connected, and ICMP to the Internet was
dropped. DNS through the gateway resolved the names fine, so it was the
connections that failed. The user had to stop the VPN to use Slack.

We switched to a **split tunnel**: only the networks you list with `--remote-ts`
are routed through the tunnel. Everything else uses the normal route. The tool
now refuses `0.0.0.0/0` (see the next point for a second reason).

### 3. XFRM interface per connection

With a plain route-based setup, strongSwan installs routes in a separate routing
table and puts the virtual IP on the physical interface. That is what made the
masquerade interaction possible. The tool instead gives each connection its own
**XFRM interface** (`swan<id>`, where `<id>` is derived from the connection name):

- the child SA is configured with `if_id_in` / `if_id_out` equal to that id,
- the virtual IP and the routes for `--remote-ts` live on the XFRM interface,
- traffic for those networks is routed into it and encrypted; nothing else is,
- masquerade is no longer involved, so the SNAT workaround is gone,
- bringing the tunnel down is just deleting the interface, which removes the
  address, routes and DNS settings in one step.

The XFRM interface is created on top of `lo` and given an explicit MTU of 1400,
so large packets do not fragment after encryption over NAT-T.

A consequence: with an XFRM interface, a `0.0.0.0/0` selector would route the
ESP packets *to the gateway itself* into the tunnel and loop. That is the second
reason the tool rejects it. If you need a full tunnel you must pin a host route
to the gateway yourself; this tool does not do that.

charon is also told not to install the virtual IP itself
(`/etc/strongswan.d/swan-vpn.conf`: `install_virtual_ip = no`), since the tool
places it on the XFRM interface. With an XFRM-referencing child, strongSwan does
not install its own routes by default (`install_routes_xfrmi = no`).

### 4. DNS: per-domain routing needs a real interface

Sending *every* DNS query to the gateway's server works but is a poor default:
all your lookups leak to a third party's resolver, and name resolution breaks if
the tunnel drops.

The first attempt to route only some domains used a **dummy link** with the
gateway's DNS server configured on it through `resolvectl`. That did not work:
`systemd-resolved` could not use the dummy link, and even a query for
`example.com` forced onto that link failed with "No appropriate name servers".
(Our reading is that resolved sends queries out of the link's own interface, and
a dummy device has no path into the tunnel; we did not prove that.)

With the XFRM interface it does work, because the interface genuinely carries
traffic into the tunnel. `swan-vpn up` runs:

```
resolvectl dns            swan<id> <gateway dns>
resolvectl domain         swan<id> ~<domain> ... ~<reverse zones>
resolvectl default-route  swan<id> no
```

so only the domains you pass with `--dns-domains` (plus the reverse
`in-addr.arpa` zones for any `/8`, `/16` or `/24` in `--remote-ts`) go to the
gateway's DNS server. Everything else keeps using your normal resolver.

## Requirements

- Linux with a kernel that has the `xfrm_interface` module (any recent one)
- strongSwan with `swanctl` and `charon-systemd` (tested with 6.1.0); the
  `strongswan.service` unit must run `charon-systemd`
- iproute2, `systemd-resolved` (`resolvectl`), `sudo`
- bash 4+, `awk`, `sed`, `cksum`

No NetworkManager plugin is needed, and the tool does not touch your firewall.

## Usage

Create a connection once. `--remote-ts` is required and lists the networks to
send through the tunnel:

```
swan-vpn add work \
    --gateway vpn.example.com \
    --user alice \
    --psk-file ~/secrets/work-psk.txt \
    --remote-ts 10.0.0.0/8,192.168.50.0/24 \
    --dns-domains corp.example.com
```

`add` prompts for the EAP password (use `--eap-secret-file FILE` to read it from
a file, or `--no-eap-secret` to reuse an existing secret for the same user).

Then:

```
swan-vpn up work       # starts strongswan if needed, connects, sets routes and DNS
swan-vpn status
swan-vpn down work     # tears down; stops strongswan when no tunnel is left
swan-vpn remove work   # down + delete the connection's config
```

Other `add` options: `--ike` and `--esp` proposals (defaults:
`aes256-sha256-modp2048,aes128-sha1-modp2048` and
`aes256-sha256-ecp384,aes128-sha1-ecp384`).

`up` is idempotent: it will not re-initiate a tunnel that is already established.

## What it writes

| Path | Purpose |
|------|---------|
| `/etc/swanctl/conf.d/<name>.conf` | connection + PSK secret, mode 600, marked `# managed-by: swan-vpn` |
| `/etc/swanctl/conf.d/<name>-eap.conf` | EAP secret, mode 600 |
| `/etc/strongswan.d/swan-vpn.conf` | `install_virtual_ip = no` |
| interface `swan<id>` | created on `up`, deleted on `down` |

The tool only touches files it created (it checks the marker) and refuses to
overwrite anything else in `/etc/swanctl/conf.d/`.

## Security notes

- The PSK is read from a file and written to a root-owned file with mode 600
  through stdin. It never appears on a command line or in output.
- PSKs and passwords containing `"` or `\` are rejected, because they would need
  escaping in the swanctl config.
- Secrets live in plain text in root-only files. That is how swanctl reads them;
  anyone with root on the machine can read them.
- `sudo` is called without `-n`, so you may be prompted for your password.
- The tunnel is **not** enabled at boot. `strongswan.service` is started and
  stopped by `up` and `down`.

## Known limitations

- **IPv4 only.** The tunnel carries IPv4 selectors; IPv6 traffic uses your normal
  route.
- **One gateway DNS server's worth of domains.** DNS routing is per connection and
  uses the servers the gateway hands out.
- **`.local` domains and glibc.** With `hosts: files mdns_minimal [NOTFOUND=return] dns`
  in `/etc/nsswitch.conf`, glibc (`getent`, `ssh`, `curl`) never queries DNS for
  `*.local` names, so a `.local` corporate domain will not resolve through
  `--dns-domains` for those programs. `resolvectl query` does honour it. Use IP
  addresses, add the name to `/etc/hosts`, or adjust `nsswitch.conf`.
- **Gateway quirks.** FortiClient also sends a vendor notify (a "network id") in
  `IKE_SA_INIT`; strongSwan does not. It was not needed for the gateway we tested
  against, but a gateway that uses it to pick between dial-up tunnels could behave
  differently.
- **Not linted.** There is no test suite and `shellcheck` was not available when
  this was written; it was tested by running it against a real gateway.

## Troubleshooting

- Look at `sudo swanctl --list-sas` for byte counters. Zero bytes with an
  established tunnel means traffic is not reaching the XFRM interface (check
  `ip route get <host in remote-ts>`; it should show `dev swan<id>`).
- `up` failing with "no EAP key found" means the EAP secret is missing; run
  `add` again without `--no-eap-secret`.
- `sudo journalctl -u strongswan` shows the full IKE exchange.
- If the gateway rejects the narrower traffic selectors, widen `--remote-ts`.
