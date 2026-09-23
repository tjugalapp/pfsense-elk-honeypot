\# Arkitektur



\## Nätverk



Tre nätverk skapade i Hyper-V:



\- \*\*Honeypot-WAN\*\* (External) – bryggad mot fysiskt Ethernet-kort (Realtek PCIe GbE), 

&#x20; ger pfSense riktig internetåtkomst

\- \*\*Honeypot-DMZ\*\* (Internal) – 10.20.20.0/24, isolerat nät där honeypotten placeras

\- \*\*Honeypot-MGMT\*\* (Internal) – 10.20.30.0/24, isolerat nät där ELK-stacken placeras



Honeypot och ELK har ingen egen internetåtkomst – all trafik styrs via pfSense.

