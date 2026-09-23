\## pfSense



\- VM: Generation 1, 2 vCPU, 2 GB RAM, 20 GB disk (UFS)

\- pfSense CE 2.9.0-RELEASE

\- Interface assignment:

&#x20; - WAN (hn0) -> Honeypot-WAN, DHCP from home router (192.168.10.114/24)

&#x20; - LAN (hn2) -> Honeypot-MGMT, static 10.20.30.2/24

&#x20; - DMZ/OPT1 (hn1) -> Honeypot-DMZ, static 10.20.20.2/24

\- SSH enabled on LAN, access confirmed from the host machine (ssh admin@10.20.30.2)

\- Admin password changed from default during installation

