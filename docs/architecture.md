# Architecture

## Network

Three networks created in Hyper-V:

- **Honeypot-WAN** (External) - bridged to the physical Ethernet adapter (Realtek PCIe GbE), gives pfSense real internet access
- **Honeypot-DMZ** (Internal) - 10.20.20.0/24, isolated network where the honeypot sits
- **Honeypot-MGMT** (Internal) - 10.20.30.0/24, isolated network where the ELK stack sits

The honeypot and ELK have no internet access of their own - all traffic is controlled via pfSense.
