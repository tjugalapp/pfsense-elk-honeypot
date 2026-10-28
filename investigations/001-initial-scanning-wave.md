# Investigation 001: The first 24 hours

## Summary

I opened the honeypot to the internet on 2026-09-24. During the first
24 hours, at least 7 different external IPs connected. Nobody tried to
log in and no commands were run. Everything looks like automated
scanning, not someone actually targeting the honeypot.

## Who connected

- **52.202.215.126** - AWS (us-east-1)
- **8.209.83.42** - Alibaba Cloud (Frankfurt)
- **162.216.150.148** - Google Cloud (us-east1)
- **85.217.149.1, .3, .9, .13, .15, .39** - Modat B.V. (AS209334)

The Modat addresses are inside the ranges Modat publishes for its
scanners (85.217.140.0/24, 85.217.149.0/24). So that's a known scanning
service, similar to Shodan or Censys, not a botnet.

## What I found

### A TLS handshake sent to the SSH port

Session ab0944ec0fbc, from 85.217.149.1:

1. `New connection: 85.217.149.1:53310` (SSH port 2222)
2. `cowrie.client.version` - instead of an SSH version string, the
   client sent the raw bytes `\x16\x03\x01\x00\xee\x01\x00\x00\xea\x03...`
3. `Connection lost after 2 milliseconds`

`\x16\x03\x01` is the start of a TLS ClientHello (0x16 = handshake,
0x03 0x01 = TLS 1.0). So the scanner sent a TLS handshake to a port that
only speaks SSH, and hung up after 2 ms.

My interpretation: it sends the same probe to every open port it
finds, without checking what service is actually running there. That
fits a scanner that wants to cover as many hosts as possible, fast.

### SSH versions

Other sessions did a normal SSH version exchange:

- `SSH-2.0-Go` - probably a scanning tool written in Go
- `SSH-2.0-OpenSSH_9.6p1 Ubuntu-3ubuntu1`
- `SSH-2.0-OpenSSH_for_Windows_9.5`

These could be the real clients the scanners use, or fake banners. I
can't tell from this data. None of them did anything after the version
exchange.

### A note on the map

The Kibana map puts these sessions in the US, Germany and Canada. That
is where the servers are hosted, not necessarily where the person
running the scan is.

## MITRE ATT&CK

- **Active Scanning: Scanning IP Blocks (T1595.001)** - broad scanning,
  nothing targeted
- **Active Scanning: Vulnerability Scanning (T1595.002)** - the TLS
  probe against the SSH port looks like generic service fingerprinting

Nothing beyond reconnaissance: no login attempts, no commands, no
persistence.

## Conclusion

The first day was only scanning from known scanning services. As far
as I understand, that's normal for a newly exposed IP: scanning
services find it first, and brute-force attempts usually come later.
