# Honeypot (Cowrie)

- VM: Ubuntu Server 24.04 LTS, Generation 1, 2 vCPU, 2 GB RAM, 20 GB disk

- Single NIC on Honeypot-DMZ only, static IP 10.20.20.10/24

- Cowrie 3.0.16 (source checkout, editable install via pip install -e .)

- Dedicated non-root service user: cowrie

- SSH honeypot on port 2222, Telnet honeypot on port 2223 (both enabled)

- Hostname set to "svr04" to avoid revealing it's a honeypot

- Default userdb.txt used as-is (blocks a few obvious honeypot-revealing

  passwords like "honeypot" and "123456" against root, allows most others)

- Verified: login, fake shell interaction, and full JSON session logging

  to var/log/cowrie/cowrie.json all working correctly

Firewall rules affecting this VM (DMZ outbound lockdown, DNS fix, and

the log-shipping rule to ELK) are documented in [pfsense.md](pfsense.md).
