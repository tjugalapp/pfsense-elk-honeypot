# Honeypot (Cowrie)

- VM: Ubuntu Server 24.04 LTS, Generation 1, 2 vCPU, 2 GB RAM, 20 GB disk
- One network card, on Honeypot-DMZ only, static IP 10.20.20.10/24
- Cowrie 3.0.16, installed from source (`pip install -e .`)
- Runs as its own non-root user: `cowrie`
- SSH honeypot on port 2222, Telnet on port 2223
- Hostname set to "svr04" so it doesn't look like a honeypot
- I kept the default `userdb.txt`. It blocks a few passwords that would
  give the honeypot away (like "honeypot" and "123456" for root) and
  allows most others
- Tested: login, the fake shell and JSON logging to
  `var/log/cowrie/cowrie.json` all worked

The firewall rules for this VM (DMZ lockdown, the DNS fix and log
shipping to ELK) are in [pfsense.md](pfsense.md).
