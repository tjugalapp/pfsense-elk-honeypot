# pfSense

## Installation

- VM: Generation 1, 2 vCPU, 2 GB RAM, 20 GB disk (UFS)
- pfSense CE 2.9.0-RELEASE
- Interface assignment:
  - WAN (hn0) -> Honeypot-WAN, DHCP from home router (192.168.10.114/24)
  - LAN (hn2) -> Honeypot-MGMT, static 10.20.30.2/24
  - DMZ/OPT1 (hn1) -> Honeypot-DMZ, static 10.20.20.2/24
- SSH enabled on LAN, access confirmed from the host machine (ssh admin@10.20.30.2)
- Admin password changed from default during installation

## Firewall Rules

### DMZ outbound lockdown

Decided to restrict DMZ outbound traffic to TCP 80/443 only, rather than
leaving it fully open or fully closed. This lets Cowrie capture real
file download attempts from attackers (a built-in feature) while keeping
the attack surface small. Cowrie runs in medium-interaction mode, so
commands like wget in the fake shell are emulated by Cowrie itself and
never reach the real OS - the main residual risk considered was a
theoretical vulnerability in Cowrie/Python/Twisted itself allowing a
breakout, in which case port 443 could be used for C2 traffic. Accepted
this risk for a home lab given the mitigations (kept patched, logs
monitored, rule can be disabled without breaking honeypot functionality).

Rules on the DMZ interface:
- Allow DMZ outbound TCP 80 (HTTP)
- Allow DMZ outbound TCP 443 (HTTPS)
- Allow DMZ outbound UDP 53 to DMZ address (pfSense DNS resolver)
- Allow DMZ outbound TCP 53 to DMZ address (pfSense DNS resolver)

Locking down outbound access broke DNS, since the honeypot was originally
pointed at 1.1.1.1/8.8.8.8. Fixed by pointing the honeypot's DNS to
pfSense's own DMZ interface (10.20.20.2) instead via netplan, and adding
firewall rules allowing DNS queries specifically to pfSense itself.

Also required enabling the DNS Resolver on the DMZ interface itself
(Services > DNS Resolver > Network Interfaces), alongside Localhost -
pfSense relies on the resolver via 127.0.0.1 for its own lookups, so
Localhost had to stay selected alongside DMZ rather than DMZ alone.

The default WAN "Block private networks" and "Block bogon networks"
rules remain enabled - confirmed active (non-zero blocked-traffic
counters) after initial concern that their red "block" icon indicated
they were disabled.

### DMZ -> MGMT (log shipping)

Narrow rules allowing only each honeypot's specific IP to reach the
ELK VM's specific IP on TCP 5044 (Filebeat -> Logstash). Kept as tight
as possible even though DMZ is otherwise locked down - no broader DMZ
subnet or MGMT-wide access granted.

- `10.20.20.10 -> 10.20.30.10:5044` (Cowrie)
- `10.20.20.11 -> 10.20.30.10:5044` (Web-honeypot / SNARE + TANNER)

### MGMT/LAN

Uses pfSense's default "allow all" rule (auto-created on the LAN
interface). Not internet-facing, so left as-is rather than restricted
like DMZ.

## Port Forwarding (Home Router -> pfSense -> Honeypot)

Two layers of NAT since pfSense sits behind the home router (double NAT):

Home router (WAN services):
- SSH-honeypot: WAN port 2022 -> LAN 192.168.10.114:22
- Telnet-honeypot: WAN port 2023 -> LAN 192.168.10.114:23
- Web-honeypot: WAN port 80 -> LAN 192.168.10.114:80

pfSense (Firewall -> NAT -> Port Forward):
- WAN:22 -> 10.20.20.10:2222 (Cowrie SSH)
- WAN:23 -> 10.20.20.10:2223 (Cowrie Telnet)
- WAN:80 -> 10.20.20.11:80 (Web-honeypot / SNARE)

Two ISP-related issues discovered during setup:
- Port 22 on the home router's WAN side is reserved by the ISP for
  their own remote management - had to use an arbitrary external port
  (2022) instead, forwarding internally to the expected port 22 that
  pfSense's rule already listens for.
- Port 23 (Telnet) was silently blocked by the ISP outbound-to-inbound
  (likely standard ISP-level filtering, common due to Telnet's history
  with IoT botnets like Mirai) - same workaround, external port changed
  to 2023.

Port 80 had no such ISP reservation and was mapped directly with no
workaround needed.

Verified externally using canyouseeme.org and a direct SSH connection
from a mobile data connection (not on the home network) - all three
ports confirmed reachable from the real internet.

The honeypots are now exposed to real internet traffic. Standard port
scanners and bots typically discover a newly-exposed IP within hours
to a few days.

## Host-level firewalling (ufw) on the honeypot VMs

Beyond pfSense, `ufw` is used on the Web-honeypot VM to restrict
TANNER's admin interfaces (ports 8090/8091) to local access only. This
surfaced two non-obvious issues worth noting here since they affect
how host-level firewalling interacts with both Docker and pfSense's
own network topology - see [Web Honeypot](web-honeypot.md) for the
full detail: `ufw` rules targeting `INPUT` do not restrict
Docker-published ports (which route through `FORWARD`), and enabling
`ufw` after Docker is already running can leave the `FORWARD` chain's
default policy set to deny, silently blocking all Docker-forwarded
traffic including pfSense's own port-forwarded connections.
