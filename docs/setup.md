\## pfSense



\- VM: Generation 1, 2 vCPU, 2 GB RAM, 20 GB disk (UFS)

\- pfSense CE 2.9.0-RELEASE

\- Interface assignment:

&#x20; - WAN (hn0) -> Honeypot-WAN, DHCP from home router (192.168.10.114/24)

&#x20; - LAN (hn2) -> Honeypot-MGMT, static 10.20.30.2/24

&#x20; - DMZ/OPT1 (hn1) -> Honeypot-DMZ, static 10.20.20.2/24

\- SSH enabled on LAN, access confirmed from the host machine (ssh admin@10.20.30.2)

\- Admin password changed from default during installation



\## Honeypot (Cowrie)



\- VM: Ubuntu Server 24.04 LTS, Generation 1, 2 vCPU, 2 GB RAM, 20 GB disk

\- Single NIC on Honeypot-DMZ only, static IP 10.20.20.10/24

\- Cowrie 3.0.16 (source checkout, editable install via pip install -e .)

\- Dedicated non-root service user: cowrie

\- SSH honeypot on port 2222, Telnet honeypot on port 2223 (both enabled)

\- Hostname set to "svr04" to avoid revealing it's a honeypot

\- Default userdb.txt used as-is (blocks a few obvious honeypot-revealing

&#x20; passwords like "honeypot" and "123456" against root, allows most others)

\- Verified: login, fake shell interaction, and full JSON session logging

&#x20; to var/log/cowrie/cowrie.json all working correctly



\## Firewall: DMZ outbound lockdown



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



\## ELK Stack



\- VM: Ubuntu Server 24.04 LTS, Generation 1, 2 vCPU, 8 GB RAM, 40 GB disk

\- Single NIC on Honeypot-MGMT only, static IP 10.20.30.10/24

\- Elasticsearch 8.19.22 - security auto-enabled (TLS, built-in users),

&#x20; password reset after initial install

\- Kibana 8.19.22 - server.host set to 10.20.30.10 to allow access from

&#x20; outside the VM, enrolled via token generated on Elasticsearch

\- Logstash 8.19.22 - configured with:

&#x20; - Beats input on port 5044 (for Filebeat from the honeypot)

&#x20; - Elasticsearch output, indexing to cowrie-YYYY.MM.dd, using

&#x20;   Elasticsearch's self-signed CA cert (copied from

&#x20;   /etc/elasticsearch/certs/http\_ca.crt to /etc/logstash/)

\- All three services verified running via systemctl and journalctl

\- Firewall: LAN/MGMT interface uses pfSense's default "allow all" rule

&#x20; (not internet-facing, so left as-is rather than restricted like DMZ)



\## Log pipeline: Cowrie -> Filebeat -> Logstash -> Elasticsearch -> Kibana



\- Firewall: narrow rule on DMZ allowing only the honeypot's specific IP

&#x20; (10.20.20.10) to reach the ELK VM's specific IP (10.20.30.10) on TCP 5044,

&#x20; rather than opening the whole DMZ subnet or the whole MGMT interface.

&#x20; This keeps the DMZ->MGMT exposure as small as possible even though the

&#x20; honeypot is otherwise locked down.

\- Filebeat 8.19.22 installed on the honeypot, configured with a filestream

&#x20; input reading /home/cowrie/cowrie/var/log/cowrie/cowrie.json, using an

&#x20; ndjson parser (target: "") to flatten Cowrie's JSON fields directly to

&#x20; the top level rather than nesting everything under "message"

\- Output: output.logstash pointed at 10.20.30.10:5044 (TLS not configured

&#x20; between Filebeat and Logstash - acceptable for an internal, already-

&#x20; segmented lab network)

\- Verified end-to-end: created a Kibana Data View (cowrie-\*), confirmed

&#x20; all 15 test session events appear in Discover with fields correctly

&#x20; parsed (session, eventid, src\_ip, username, password, message, etc.)

