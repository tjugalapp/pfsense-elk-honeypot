# ELK Stack

## Installation

- VM: Ubuntu Server 24.04 LTS, Generation 1, 2 vCPU, 8 GB RAM, 40 GB disk
- Single NIC on Honeypot-MGMT only, static IP 10.20.30.10/24
- Elasticsearch 8.19.22 - security auto-enabled (TLS, built-in users), password reset after initial install
- Kibana 8.19.22 - server.host set to 10.20.30.10 to allow access from outside the VM, enrolled via token generated on Elasticsearch
- Logstash 8.19.22 - configured with:
  - Beats input on port 5044 (for Filebeat from the honeypot)
  - Elasticsearch output, indexing to cowrie-YYYY.MM.dd, using Elasticsearch's self-signed CA cert (copied from /etc/elasticsearch/certs/http_ca.crt to /etc/logstash/)
- All three services verified running via systemctl and journalctl

Firewall: see [pfsense.md](pfsense.md) for the MGMT/LAN and log-shipping rules affecting this VM.

## Log Pipeline: Cowrie -> Filebeat -> Logstash -> Elasticsearch -> Kibana

- Filebeat 8.19.22 installed on the honeypot, configured with a filestream input reading /home/cowrie/cowrie/var/log/cowrie/cowrie.json, using an ndjson parser (target: "") to flatten Cowrie's JSON fields directly to the top level rather than nesting everything under "message"
- Output: output.logstash pointed at 10.20.30.10:5044 (TLS not configured between Filebeat and Logstash - acceptable for an internal, already-segmented lab network)
- Verified end-to-end: created a Kibana Data View (cowrie-*), confirmed all 15 test session events appear in Discover with fields correctly parsed (session, eventid, src_ip, username, password, message, etc.)

## GeoIP Enrichment and Map Visualization

Added a geoip filter to the Logstash pipeline (10-geoip-filter.conf), enriching each event's src_ip using the bundled GeoLite2-City database. Target field set to "source" (not the default "geoip") to follow ECS naming conventions, after Logstash flagged a warning about this. Elasticsearch only creates field mappings the first time a field actually appears in a document. Since all test traffic came from a private IP (no GeoIP data resolves for private addresses), the geo fields never appeared in any document, so Kibana couldn't detect them as geospatial fields. Fixed by explicitly defining the mapping via an index template (applies to all future cowrie-* indices) and a direct mapping update on the current day's index - covers: source.geo.location (geo_point), source.geo.country_name, source.geo.city_name, source.geo.region_name, source.geo.country_code2.

## Kibana Dashboard

Built a "Cowrie Honeypot Overview" dashboard with five panels:

- Events Over Time (line chart, date histogram on timestamp)
- Top Source IPs (table, terms on src_ip)
- Top Commands Executed (table, terms on message.keyword, filtered to eventid.keyword: cowrie.command.input)
- Login Outcomes (pie chart, terms on eventid.keyword, filtered to cowrie.login.success / cowrie.login.failed)
- Attacker Locations (Kibana Maps, Documents layer on source.geo.location, tooltips showing eventid, session, source.geo.city_name, source.geo.country_name, username)

Current dashboard only reflects test session data (single source IP, 5 unique commands, no geo data since the test IP is private) - will look substantially more interesting once port forwarding exposes the honeypot to real internet traffic.
