\## pfSense



\- VM: Generation 1, 2 vCPU, 2 GB RAM, 20 GB disk (UFS)

\- pfSense CE 2.9.0-RELEASE

\- Interface-tilldelning:

&#x20; - WAN (hn0) → Honeypot-WAN, DHCP från hemrouter (192.168.10.114/24)

&#x20; - LAN (hn2) → Honeypot-MGMT, statisk 10.20.30.2/24

&#x20; - DMZ/OPT1 (hn1) → Honeypot-DMZ, statisk 10.20.20.2/24

\- SSH aktiverat på LAN, åtkomst bekräftad från värddatorn (ssh admin@10.20.30.2)

\- Admin-lösenord ändrat från default under installationen

