# Investigation 002: Recurring Credential Validation (Cowrie) and Identity-Spoofed Recon (Web Honeypot)

## Summary

Two separate, notable behaviors observed across the two honeypots during
the same reporting window:

1. A single source repeatedly logged into the Cowrie SSH honeypot with a
   known credential pair, then attempted to use the session as an SSH
   port-forward relay to test outbound connectivity - not brute-forcing,
   but *validating* an already-known set of credentials on a schedule.
2. A high-volume source hit the web honeypot while presenting a spoofed
   `Googlebot` User-Agent string, despite being hosted on infrastructure
   with no relation to Google.

## Part 1: Cowrie - 176.53.159.196

**Source:** 176.53.159.196 - Zorntech Web Solutions (AS154383), registered
with APNIC under a Bangladesh contact, geolocated to Istanbul, Turkey,
with an `ip-api` org field ("BearShield Technologies S.R.O.") pointing to
a Slovak company name. Three different countries across
registration/org-name/geolocation is a pattern seen before on this
project (see Investigation 001's EMBNEX finding on the web honeypot side)
- consistent with small, newly-registered hosting providers rather than
a well-known cloud platform.

**Pattern:** at least 15 sessions captured in the reviewed window, spaced
roughly 54-64 seconds apart, each one near-identical:

1. Connect, `SSH-2.0-Go` client, same hassh fingerprint every time
   (`eff4c24daffc8532c160e86e5f006e53`)
2. Login with `support`/`support` - succeeds immediately, no failed
   attempts beforehand
3. A single `direct-tcpip` forwarding request to `1.1.1.1:53`
4. The forwarded payload, decoded as a DNS-over-TCP query, asks for the
   A record of `a.to`
5. Cowrie discards the forward request (`discarded direct-tcp forward
   request`) - the tunnel is never actually established
6. Disconnect, ~212-223ms total session length

The dashboard's Top Usernames/Passwords panel shows 22 total successful
logins under `support`/`support`, consistent with this same pattern
continuing beyond the 15 sessions captured in this review.

**Interpretation:** the identical timing, client fingerprint, and
payload across every session point to a scripted, scheduled check
rather than a live attacker typing commands. The behavior looks like
verification that a previously-obtained credential pair still works,
combined with a check of whether the host can be used to relay traffic
elsewhere via SSH port forwarding - a common step before committing to
using a compromised host as a proxy or pivot point.

The `a.to` domain itself: `.to` is Tonga's country-code TLD, popular
globally as a short "domain hack" and often used for URL-shortening
services. Its exact role here isn't confirmed - it's consistent with
either a minimal, low-overhead connectivity-check target or a
URL-shortener redirect, and this write-up doesn't treat either as
established fact.

Cowrie's own behavior is worth noting directly: the login was allowed
to succeed (by design), but the tunnel request was rejected rather than
forwarded - the honeypot did not become a usable relay for this traffic.

**MITRE ATT&CK:**
- **Valid Accounts (T1078)** - repeated successful authentication with a
  known credential pair, not a brute-force attempt
- **Protocol Tunneling (T1572)** - the SSH port-forward request to
  reach an external host through the compromised session, blocked by
  Cowrie before it could be used

## Part 2: Web Honeypot - 139.144.160.162

**Source:** 139.144.160.162 - Linode / Akamai Connected Cloud, Frankfurt.
Generic commercial VPS hosting, unrelated to Google.

**Behavior:** 300 requests recorded from this source, all presenting the
User-Agent `Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)`
(the standard Googlebot string). Real Googlebot traffic only ever
originates from Google-owned, publicly verifiable IP ranges - never
from a third-party VPS provider - so this is User-Agent spoofing, not
an actual search-engine crawler.

Separately, the dashboard's Detection Types panel recorded 3 `wp-content`
classifications (out of ~1,211 total detections, the rest overwhelmingly
`index`) - indicating at least one visitor probed WordPress-specific
paths rather than just the site root. This write-up has not isolated
which source IP triggered those specific detections, so it's noted here
as a separate observation rather than attributed to the Googlebot-spoofing
source without evidence.

**Interpretation:** spoofing a well-known, generally-trusted crawler's
identity is a deliberate evasion technique - some naive filtering rules
allowlist anything claiming to be Googlebot to avoid harming SEO, so an
attacker or scanner adopting that identity can blend into normal traffic
logs.

**MITRE ATT&CK:**
- **Masquerading (T1036)** - most directly defined for files/processes,
  applied here by analogy to network-identity spoofing via a forged
  User-Agent header
- **Active Scanning: Vulnerability Scanning (T1595.002)** - the
  `wp-content` detections suggest at least some CMS-specific structure
  probing, separate from generic path requests

## Conclusion

Both honeypots captured behavior beyond simple scanning in this window.
Cowrie's incident shows a credential-validation-and-relay-test pattern
rather than brute-forcing, and demonstrates the segmentation working as
intended - the login succeeded, but the attempted onward connection was
blocked. The web honeypot's incident shows identity spoofing as an
evasion technique, layered with at least limited CMS-specific
enumeration. Neither honeypot was used successfully for anything beyond
what it was designed to permit and log.
