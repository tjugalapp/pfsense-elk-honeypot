# Architecture

## Networks

I created three networks in Hyper-V:

- **Honeypot-WAN** (External) - bridged to the physical network card
  (Realtek PCIe GbE), so pfSense gets real internet access
- **Honeypot-DMZ** (Internal) - 10.20.20.0/24, an isolated network for
  the honeypots
- **Honeypot-MGMT** (Internal) - 10.20.30.0/24, an isolated network for
  the ELK stack

The honeypots and ELK have no internet access of their own. Everything
goes through pfSense.

### Internal vs Private switches

DMZ and MGMT use Hyper-V **Internal** switches, not **Private** ones.
The difference matters: an Internal switch gives the Hyper-V host its
own network card on that network. That means my host machine can reach
both segments directly, without going through pfSense.

For a lab that's OK, since I trust my own machine. But it means pfSense
only controls traffic between the VMs and the internet, not traffic
from the host itself. I only noticed this later, when testing the web
honeypot (see [Web Honeypot](web-honeypot.md)).

## Hosts on the DMZ

- **Honeypot** (10.20.20.10) - Cowrie SSH/Telnet honeypot
- **Web-honeypot** (10.20.20.11) - SNARE + TANNER web honeypot

Each honeypot type has its own VM. I tried to keep one service per VM
throughout the lab.
