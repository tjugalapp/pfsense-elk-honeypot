# Setup Overview

Quick index of how this lab is built. See the linked files for full detail on each component.

1. [Architecture](architecture.md) - network design, IP scheme
2. [pfSense](pfsense.md) - firewall/router installation and all rules
3. [Honeypot](honeypot.md) - Cowrie installation and configuration
4. [Web Honeypot](web-honeypot.md) - SNARE + TANNER installation, decoy page, and the bugs hit along the way
5. [ELK Stack](elk-stack.md) - Elasticsearch/Kibana/Logstash, log pipeline, GeoIP enrichment, and dashboard

Build order: network -> pfSense -> honeypot -> ELK stack -> log pipeline -> GeoIP/dashboard -> port forwarding for real traffic -> web honeypot (SNARE + TANNER) as a second, independent honeypot on its own VM.
