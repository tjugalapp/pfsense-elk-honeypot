# Investigation 001: Initial Scanning Wave (First 24h of Exposure)

## Summary

Within the first 24 hours of exposing the honeypot to the internet (2026-09-24), it received traffic from at least 7 distinct external source IPs. No successful or attempted logins occurred, and no commands were executed in the fake shell. All observed activity is consistent with automated internet-wide scanning infrastructure rather than targeted or manual attacker behavior.

## Sources Observed

- **52.202.215.126** - AWS (us-east-1) - cloud-hosted scanner
- **8.209.83.42** - Alibaba Cloud (Frankfurt) - cloud-hosted scanner
- **162.216.150.148** - Google Cloud (us-east1) - cloud-hosted scanner
- **85.217.149.1, .3, .9, .13, .15, .39** - Modat B.V. (AS209334) - dedicated internet-scanning service

The Modat addresses all fall within Modat's published scanner ranges (85.217.140.0/24, 85.217.149.0/24), confirming they belong to a known, purpose-built scanning operation (comparable to Shodan/Censys) rather than a botnet or individual attacker.

## Technical Findings

### Protocol-blind probing (session ab0944ec0fbc, src 85.217.149.1)

Timeline:

1. `New connection: 85.217.149.1:53310` (SSH port 2222)
2. `cowrie.client.version` - payload was not an SSH version string, but the raw bytes `\x16\x03\x01\x00\xee\x01\x00\x00\xea\x03...`
3. `Connection lost after 2 milliseconds`

The byte sequence `\x16\x03\x01` is the standard TLS record header for a ClientHello (0x16 = handshake record, 0x03 0x01 = TLS 1.0). This means the scanner sent a generic TLS handshake probe against a port that only speaks SSH, and disconnected within 2ms without waiting for a meaningful response. **Interpretation:** this indicates a protocol-blind scanning approach - the same probe(s) fired at every open port discovered, regardless of the port's actual protocol, rather than an SSH-specific scanner performing a proper handshake. This is consistent with high-throughput internet-wide scanning, which optimizes for breadth of coverage over depth of interaction per target.

### SSH client version strings observed

Other sessions did complete a valid SSH version exchange:

- `SSH-2.0-Go` - consistent with a Go-based scanning tool
- `SSH-2.0-OpenSSH_9.6p1 Ubuntu-3ubuntu1`
- `SSH-2.0-OpenSSH_for_Windows_9.5`

These may be genuine SSH client implementations used by scanning tools, or spoofed banners designed to blend in with legitimate traffic. No further interaction (login attempt, command execution) followed any of these version exchanges.

### GeoIP caveat

The Kibana map plots these sessions in the US, Germany, and Canada. This reflects the geographic location of the cloud/hosting infrastructure the scanner runs from, not necessarily the physical location of the operator - an important distinction when interpreting GeoIP data from cloud-hosted sources.

## MITRE ATT&CK Mapping

- **Active Scanning: Scanning IP Blocks (T1595.001)** - broad, non-targeted probing across the exposed IP/port
- **Active Scanning: Vulnerability Scanning (T1595.002)** - protocol-blind TLS probe against an SSH port, consistent with generic service/vulnerability fingerprinting

No techniques beyond the Reconnaissance stage were observed - no credential access, execution, or persistence activity occurred.

## Conclusion

The first 24 hours of exposure produced exclusively reconnaissance traffic from known scanning infrastructure (AWS, Alibaba Cloud, Google Cloud, and the dedicated Modat scanning service). No brute force attempts, credential testing, or shell interaction occurred. This is a typical and expected pattern for a newly internet-facing honeypot - discovery by scanning services generally precedes targeted or automated brute-force activity, which is expected as the honeypot's exposure becomes more widely indexed over time.
