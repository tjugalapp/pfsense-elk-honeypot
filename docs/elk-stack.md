# ELK Stack

## Installation

- VM: Ubuntu Server 24.04 LTS, Generation 1, 2 vCPU, 8 GB RAM, 40 GB disk
- Single NIC on Honeypot-MGMT only, static IP 10.20.30.10/24
- Elasticsearch 8.19.22 - security auto-enabled (TLS, built-in users),
  password reset after initial install
- Kibana 8.19.22 - server.host set to 10.20.30.10 to allow access from
  outside the VM, enrolled via token generated on Elasticsearch
- Logstash 8.19.22 - configured with:
  - Beats input on port 5044 (for Filebeat from both honeypots)
  - Elasticsearch output, indexing to `%{[log_type]}-%{+YYYY.MM.dd}`
    (separate `cowrie-*` and `tanner-*` indices rather than mixing
    both honeypots' data together), using Elasticsearch's self-signed
    CA cert (copied from /etc/elasticsearch/certs/http_ca.crt to
    /etc/logstash/)
- All services verified running via systemctl and journalctl

Firewall: see [pfsense.md](pfsense.md) for the MGMT/LAN and log-shipping
rules affecting this VM.

## Log Pipeline: Cowrie / SNARE+TANNER -> Filebeat -> Logstash -> Elasticsearch -> Kibana

- Filebeat 8.19.22 installed on both honeypot VMs:
  - Cowrie VM: filestream input reading
    /home/cowrie/cowrie/var/log/cowrie/cowrie.json, ndjson parser
    (target: "") to flatten Cowrie's JSON fields to the top level;
    tagged with `log_type: cowrie`
  - Web-honeypot VM: filestream input reading
    ~/tanner/docker/data/tanner/tanner_report.json (one JSON object
    per line, same NDJSON-style format as Cowrie's log); tagged with
    `log_type: tanner`
- Output: both point at output.logstash -> 10.20.30.10:5044 (TLS not
  configured between Filebeat and Logstash - acceptable for an
  internal, already-segmented lab network)
- Verified end-to-end for both: Kibana Data Views `cowrie-*` and
  `tanner-*` created, events appear in Discover with fields correctly
  parsed (Cowrie: session, eventid, src_ip, username, password,
  message; TANNER: peer.ip, path, method, status, headers,
  response_msg.response.message.detection)

## GeoIP Enrichment and Map Visualization

Added a geoip filter to the Logstash pipeline (10-geoip-filter.conf),
enriching each event's source IP using the bundled GeoLite2-City
database. Target field set to "source" (not the default "geoip") to
follow ECS naming conventions, after Logstash flagged a warning about
this.

Since Cowrie and TANNER use different field names for the source IP
(`src_ip` vs `peer.ip`), the filter uses an if/else based on
`log_type` to pick the right source field for each event type, while
still writing both into the same `source.geo.*` fields - allowing a
single Kibana Maps layer to plot both honeypots' attacker locations
together if needed.

Elasticsearch only creates field mappings the first time a field
actually appears in a document. Since all test traffic came from a
private IP (no GeoIP data resolves for private addresses), the geo
fields never appeared in any document, so Kibana couldn't detect them
as geospatial fields. Fixed by explicitly defining the mapping via an
index template (applies to all future `cowrie-*` and `tanner-*`
indices) and a direct mapping update on the current day's index -
covers: source.geo.location (geo_point), source.geo.country_name,
source.geo.city_name, source.geo.region_name, source.geo.country_code2.

## Kibana Dashboard

"Cowrie Honeypot Overview" dashboard, covering both honeypots:

**Cowrie (SSH/Telnet) panels:**
- Events Over Time (line chart, date histogram on timestamp)
- Top Source IPs (table, terms on src_ip)
- Top Usernames / Top Passwords (tables, terms on username.keyword /
  password.keyword, filtered to exclude Cowrie's own placeholder "-"
  values)
- Top Commands Executed (table, terms on message.keyword, filtered to
  eventid.keyword: cowrie.command.input)
- Login Outcomes (pie chart, terms on eventid.keyword, filtered to
  cowrie.login.success / cowrie.login.failed)
- Event Types (table, terms on eventid.keyword, unfiltered - shows the
  full event breakdown including connection/version/kex events with no
  login or command activity)
- SSH Client Versions (table, terms on version.keyword) - surfaces the
  SSH client string a connecting scanner presents, including
  non-SSH payloads (e.g. a raw TLS ClientHello sent against the SSH
  port by a protocol-blind scanner)

**Web Honeypot (SNARE + TANNER) panels:**
- Web Honeypot - Requested Paths (table, terms on path.keyword)
- Web Honeypot - Top Source IPs (table, terms on peer.ip)
- Web Honeypot - Top User-Agents (table, terms on
  headers.user-agent.keyword)
- Web Honeypot - Detection Types (table, terms on
  response_msg.response.message.detection.name.keyword)
- Web Honeypot - Status Codes (table, terms on status)

**Shared:**
- Attacker Locations (Kibana Maps) - two Documents layers, one per
  honeypot's data view (`cowrie` and `tanner`), each styled with a
  distinct marker color, both mapped through the shared
  `source.geo.location` field. Tooltip fields differ per layer since
  Cowrie and TANNER don't share field names (Cowrie: eventid, session,
  username, src_ip; TANNER: path, peer.ip, method, status, uuid) -
  both include source.geo.city_name and source.geo.country_name.

Early findings from the web honeypot side worth noting for future
investigations: SNARE appears to fall back to serving its cloned index
page (HTTP 200) for most unrecognized paths rather than a genuine 404,
which explains why "Detection Types" skews heavily toward "index" even
when a range of different paths were requested - a behavioral
difference from a real WordPress install worth accounting for when
interpreting the Status Codes panel. Separately, one high-volume source
(300+ requests) presented a "Googlebot/2.1" User-Agent while hosted on
Linode/Akamai Connected Cloud rather than any Google-owned range - a
clear case of User-Agent spoofing rather than a legitimate crawler.
