# Architecture

## Network

Three networks created in Hyper-V:

- **Honeypot-WAN** (External) - bridged to the physical Ethernet adapter (Realtek PCIe GbE), gives pfSense real internet access
- **Honeypot-DMZ** (Internal) - 10.20.20.0/24, isolated network where the honeypots sit
- **Honeypot-MGMT** (Internal) - 10.20.30.0/24, isolated network where the ELK stack sits

The honeypots and ELK have no internet access of their own - all traffic is controlled via pfSense.

**Note on switch type:** Honeypot-DMZ and Honeypot-MGMT are Hyper-V
**Internal** switches, not **Private** switches. An Internal switch
gives the Hyper-V host itself a virtual NIC directly on that network
segment, meaning the management workstation has a direct path into
both segments that never passes through pfSense. This is an accepted
tradeoff for a lab (the admin machine is trusted), but it means
pfSense's segmentation only governs traffic between VMs and the
internet/each other - not traffic originating from the Hyper-V host
itself. See [Web Honeypot](web-honeypot.md) for where this became
relevant in practice.

## Hosts on the DMZ

- **Honeypot** (10.20.20.10) - Cowrie SSH/Telnet honeypot
- **Web-honeypot** (10.20.20.11) - SNARE + TANNER web honeypot

Each honeypot type has its own VM, per the one-service-per-VM pattern
used throughout this lab.
