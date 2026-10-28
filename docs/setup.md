# Setup overview

A quick index of how the lab is built. Details are in the linked files.

1. [Architecture](architecture.md) - network design and IP addresses
2. [pfSense](pfsense.md) - firewall/router install and all rules
3. [Honeypot](honeypot.md) - Cowrie install and config
4. [Web Honeypot](web-honeypot.md) - SNARE + TANNER, the decoy page, and
   the bugs I ran into
5. [ELK Stack](elk-stack.md) - Elasticsearch, Logstash, Kibana, log
   pipeline, GeoIP and dashboard

The order I built it in: network -> pfSense -> honeypot -> ELK stack ->
log pipeline -> GeoIP/dashboard -> port forwarding for real traffic ->
web honeypot (SNARE + TANNER) as a second honeypot on its own VM.
