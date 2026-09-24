\# pfSense



\## Installation



\- VM: Generation 1, 2 vCPU, 2 GB RAM, 20 GB disk (UFS)

\- pfSense CE 2.9.0-RELEASE

\- Interface assignment:

&#x20; - WAN (hn0) -> Honeypot-WAN, DHCP from home router (192.168.10.114/24)

&#x20; - LAN (hn2) -> Honeypot-MGMT, static 10.20.30.2/24

&#x20; - DMZ/OPT1 (hn1) -> Honeypot-DMZ, static 10.20.20.2/24

\- SSH enabled on LAN, access confirmed from the host machine (ssh admin@10.20.30.2)

\- Admin password changed from default during installation



\## Firewall Rules



\### DMZ outbound lockdown



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

\- Allow DMZ outbound TCP 80 (HTTP)

\- Allow DMZ outbound TCP 443 (HTTPS)

\- Allow DMZ outbound UDP 53 to DMZ address (pfSense DNS resolver)

\- Allow DMZ outbound TCP 53 to DMZ address (pfSense DNS resolver)



Locking down outbound access broke DNS, since the honeypot was originally

pointed at 1.1.1.1/8.8.8.8. Fixed by pointing the honeypot's DNS to

pfSense's own DMZ interface (10.20.20.2) instead via netplan, and adding

firewall rules allowing DNS queries specifically to pfSense itself.



\### DMZ -> MGMT (log shipping)



Narrow rule allowing only the honeypot's specific IP (10.20.20.10) to

reach the ELK VM's specific IP (10.20.30.10) on TCP 5044 (Filebeat ->

Logstash). Kept as tight as possible even though DMZ is otherwise

locked down - no broader DMZ subnet or MGMT-wide access granted.



\### MGMT/LAN



Uses pfSense's default "allow all" rule (auto-created on the LAN

interface). Not internet-facing, so left as-is rather than restricted

like DMZ.

