# Investigation 002: A bot re-checking a known login (Cowrie) and a fake Googlebot (web honeypot)

## Summary

Two different things stood out on the two honeypots during the same
period:

1. One source kept logging in to Cowrie with a known username and
   password, and then tried to use the session to forward traffic. It
   wasn't brute-forcing. It looked like it was checking, on a schedule,
   that a login it already had still worked.
2. A source that sent a lot of requests to the web honeypot claimed to
   be `Googlebot`, but was hosted somewhere that has nothing to do with
   Google.

## Part 1: Cowrie - 176.53.159.196

**Source:** 176.53.159.196 - Zorntech Web Solutions (AS154383).
Registered with APNIC with a contact in Bangladesh, geolocated to
Istanbul, Turkey, and the `ip-api` org field ("BearShield Technologies
S.R.O.") points to a Slovak company name. That's three different
countries for one IP. It seems typical of small, newly registered
hosting providers, rather than the big cloud platforms.

**What happened:** I captured at least 15 sessions in the period I
looked at, about 54-64 seconds apart, and almost identical:

1. Connect with an `SSH-2.0-Go` client, same hassh fingerprint every
   time (`eff4c24daffc8532c160e86e5f006e53`)
2. Log in with `support`/`support` - works right away, no failed
   attempts before
3. One `direct-tcpip` forwarding request to `1.1.1.1:53`
4. The forwarded data, decoded as a DNS-over-TCP query, asks for the A
   record of `a.to`
5. Cowrie drops the forward request (`discarded direct-tcp forward
   request`), so the tunnel never opens
6. Disconnect, about 212-223 ms per session

The Top Usernames/Passwords panel showed 22 successful logins with
`support`/`support` in total, so the same thing probably kept going
after the 15 sessions I looked at.

**My interpretation:** same timing, same client fingerprint and same
payload every time. That's a script on a schedule, not a person
typing. It looks like it checks that a login it already has still
works, and whether the host can be used to relay traffic with SSH port
forwarding. That's a common step before using a hacked machine as a
proxy.

About `a.to`: `.to` is Tonga's country code domain. It's popular as a
short domain name and often used for URL shorteners. I don't know what
it was used for here. It could be a simple connectivity check or a URL
shortener, and I'm not treating either as a fact.

Cowrie did what it should: the login was allowed (that's how it's
designed), but the tunnel request was rejected, so the honeypot never
became a working relay.

**MITRE ATT&CK:**
- **Valid Accounts (T1078)** - logging in again and again with a known
  username/password, not brute force
- **Protocol Tunneling (T1572)** - the SSH port forwarding request to
  reach an outside host, blocked by Cowrie

## Part 2: Web honeypot - 139.144.160.162

**Source:** 139.144.160.162 - Linode / Akamai Connected Cloud,
Frankfurt. Ordinary VPS hosting, not related to Google.

**What happened:** 300 requests from this source, all with the
User-Agent
`Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)`,
which is the normal Googlebot string. Real Googlebot only comes from
Google's own IP ranges, which can be checked, never from a third-party
VPS. So the User-Agent was faked.

Separately, the Detection Types panel showed 3 `wp-content`
classifications out of about 1,211 detections (almost all the rest were
`index`). So at least one visitor looked for WordPress-specific paths,
not just the front page. I haven't figured out which IP caused those,
so I'm not linking them to the fake Googlebot.

**My interpretation:** pretending to be a trusted crawler is a way to
hide. Some filters let anything that claims to be Googlebot through, so
they don't hurt SEO. A scanner using that name can then blend in with
normal traffic.

**MITRE ATT&CK:**
- **Masquerading (T1036)** - mostly defined for files and processes,
  but I'm using it here by analogy for a faked network identity
  (User-Agent)
- **Active Scanning: Vulnerability Scanning (T1595.002)** - the
  `wp-content` detections suggest some probing for CMS structure,
  separate from the general requests

## Conclusion

Both honeypots caught more than plain scanning during this period. On
Cowrie it was a bot checking a known login and testing if it could
relay traffic. The login worked, but Cowrie blocked the onward
connection. On the web honeypot it was a scanner hiding behind a fake
Googlebot name, plus at least some WordPress-specific probing. Neither
honeypot was used for anything beyond what it was set up to allow and
log.
