# Web Honeypot (SNARE + TANNER)

## Why a web honeypot

Cowrie covers SSH/Telnet, but a large share of real internet scanning
targets web services instead (WordPress logins, exposed admin panels,
vulnerable CMS endpoints). Adding a web-facing honeypot broadens the
lab's coverage and, unlike Cowrie, comes with automatic attack
classification (SQL injection, XSS, LFI/RFI attempts, etc.) via
TANNER, rather than requiring manual log inspection to categorize
behavior.

SNARE (the low-interaction sensor that serves a cloned page) and
TANNER (the classification/analysis backend) were chosen over a
simpler self-built decoy specifically for this reason - the
classification output gives investigations a structured starting
point instead of raw request logs alone.

## VM

- Ubuntu Server 24.04 LTS, Generation 1, 2 vCPU, 4 GB RAM
- Single NIC on Honeypot-DMZ only, static IP 10.20.20.11/24, gateway/DNS 10.20.20.2
- Kept on its own VM rather than sharing the Cowrie VM, per TANNER's
  own documentation, which recommends running TANNER on a different
  host than SNARE in production. The two services here still share
  one VM (SNARE + TANNER together) - only Cowrie was kept fully
  separate, since combining a second honeypot stack onto the
  already-tight 2 GB Cowrie VM would have been both a resource
  squeeze and a deviation from the one-service-per-VM pattern used
  elsewhere in this lab

## Installation

Docker and the Docker Compose plugin installed via `get.docker.com`.

### TANNER

Cloned to `~/tanner`, built and run via `~/tanner/docker`
docker-compose (five services: `tanner`, `tanner_api`, `tanner_web`,
`tanner_redis`, `tanner_phpox`). Ports 8090 (tanner) and 8091
(tanner_web) initially published to the host.

Two issues hit during the build, both configuration mismatches
rather than application bugs:

- `tanner_redis` was set `read_only: true`, but Redis still tried to
  write RDB snapshots to disk, causing crashes. Fixed by adding
  `command: ["redis-server", "--save", "", "--appendonly", "no",
  "--stop-writes-on-bgsave-error", "no"]` to the redis service so it
  never attempts a disk write.
- The `tanner` service's log directory (`/tmp/tanner`) was mounted as
  `tmpfs` - contents vanish on restart and aren't visible from the
  host. Replaced with a real bind mount (`./data/tanner:/tmp/tanner`),
  with the host directory created via `mkdir -p data/tanner` and
  `chown 65534:65534` to match the container's unprivileged user.

### SNARE

Cloned to `~/snare`, pointed at the local TANNER instance via the
`TANNER=10.20.20.11` environment variable in `docker-compose.yml`
(SNARE's code hardcodes port 8090, so only the host/IP needs
setting).

Two application-level bugs were found and fixed in SNARE itself
during this setup, both worth documenting as they affect anyone
running current SNARE against a modern target:

- **Build failure on EOL base image.** The Dockerfile built on
  `python:3.6-alpine3.8` (end-of-life), and `requirements.txt`
  required `gitpython==3.1.30`, which isn't supported on Python 3.6.
  Fixed by upgrading the base image to `python:3.9-alpine`.
- **Invalid HTTP responses from cloned pages** (`snare/tanner_handler.py`,
  `parse_tanner_response()`, the `detection["type"] == 1` branch).
  SNARE copied the cloned page's saved headers verbatim, including
  `Transfer-Encoding: chunked`, which conflicts with the
  `Content-Length` header SNARE also sets. The result was a
  technically-invalid HTTP response: browsers and `curl` (without
  `-I`) couldn't parse the body, even though a `HEAD` request looked
  fine. Fixed by adding a `skip_headers` filter that excludes
  `transfer-encoding`, `content-length`, `content-encoding`, and
  `connection` before copying headers from the cloned page's metadata.

By default SNARE clones `http://example.com` as its decoy page.

## Replacing the decoy page (example.com -> a real WordPress site)

`example.com` is a static page with nothing for TANNER to actually
classify beyond "index" - no login form, no CMS structure, no
plugin/theme paths. To get meaningful classification variety, a
disposable WordPress instance was built specifically to be cloned
(rather than cloning an existing real-world site, which raises
consent/legal questions even for a private lab):

```yaml
# ~/wp-decoy/docker-compose.yml
services:
  db:
    image: mysql:8.0
    ...
  wordpress:
    image: wordpress:latest
    ports:
      - "8080:80"
    ...
```

Installed as a generic "Company Blog" site (default WordPress content,
no real data). Re-pointing SNARE at it surfaced three further,
unrelated bugs - a useful reminder that a fix that looks complete can
still be masking a second problem underneath:

1. **Docker build cache masked a silent clone failure.** `docker
   compose build --build-arg PAGE_URL=10.20.20.11:8080` reported
   success (`RUN clone` showed as completed with exit code 0), but
   the `clone` tool had actually failed to reach the WordPress
   container during the build step and swallowed the error instead of
   failing the build. Because Docker's build cache treated the empty
   result as valid, every subsequent rebuild just replayed the same
   silent failure. Fixed with `sudo ufw allow 8080/tcp` (build-time
   access to the target) and `docker compose build --no-cache ...`
   to force a genuine reclone.
2. **Port stripped from the page directory name.** Even after a
   successful clone, SNARE still failed with `--page-dir:
   10.20.20.11:8080 does not exist`. The root cause, found in SNARE's
   `clone.py`: `self.target_path =
   '/opt/snare/pages/{}'.format(self.root.host)` - the URL parser's
   `.host` property strips the port, so the cloned page landed in
   `/opt/snare/pages/10.20.20.11` (no port), while the Dockerfile's
   `CMD` passed the full `$PAGE_URL` (with port) as `--page-dir`,
   a directory that never existed. This only surfaces when the
   target includes a port - `example.com` never hit it, since a bare
   domain has nothing to strip. Fixed with an explicit `command:`
   override in `docker-compose.yml`, decoupling the correct
   (port-less) `--page-dir` value from the build-time `PAGE_URL` used
   for cloning.
3. **SNARE ran, but external requests got refused - `FORWARD`
   chain, not the application.** With the page-dir bug fixed, SNARE
   logged a clean startup (`Running on http://0.0.0.0`), yet `curl`
   against the DMZ IP failed immediately. Diagnosed layer by layer:
   ufw rules were fine, Docker's NAT rule correctly pointed at the
   container's actual bridge IP, and curling that bridge IP directly
   returned a valid 200 OK - isolating the fault to the path between
   the DMZ interface and the bridge. `sudo iptables -L FORWARD`
   showed `Chain FORWARD (policy DROP)`. Enabling `ufw` after Docker
   was already running had set the `FORWARD` chain's default policy
   to deny, which silently blocks all Docker-forwarded traffic
   regardless of `DOCKER-USER` rules looking empty/permissive. Fixed
   with `sudo ufw default allow routed` + `sudo ufw reload`. Verified
   externally afterward via mobile data.

**Lesson worth keeping in mind for any future Docker + ufw host:**
enabling `ufw` after Docker is already running can silently break
container port forwarding through the `FORWARD` chain, even when
`ufw status` and `DOCKER-USER` both look unremarkable. Worth checking
`iptables -L FORWARD` explicitly on any Docker host where `ufw` is
introduced later.

## A second networking gap: Hyper-V "Internal" switches

While testing TANNER's web UI, it became reachable directly from the
Windows host's browser at `http://10.20.20.11:8091` with no SSH
tunnel - unexpected, since an earlier `ufw` rule was meant to restrict
that port to localhost only.

Two separate causes were involved:

- The Hyper-V switches backing Honeypot-DMZ and Honeypot-MGMT are
  type **Internal**, not **Private**. An Internal switch gives the
  Hyper-V host itself a virtual NIC directly on that network segment
  - so the management workstation has always had a direct,
  unfiltered path into the DMZ, bypassing pfSense entirely. This is a
  reasonable tradeoff for a lab (the admin machine is trusted), but
  worth being explicit about: DMZ segmentation only applies to
  traffic that actually passes through pfSense's interfaces, not to
  the Hyper-V host's own direct access.
- Independently, the earlier `ufw allow from 127.0.0.1 to any port
  8090/8091` + `deny Anywhere` rules never actually restricted
  Docker-published ports in the first place. Docker's port
  publishing (`0.0.0.0:8090->8090`) is implemented via `iptables` DNAT
  rules that traffic reaches through the `FORWARD` chain, not
  `INPUT` - the chain `ufw`'s `allow`/`deny` rules for a specific
  port actually govern. The same root-cause class as the WordPress
  connectivity bug above.

**Correct fix (to apply):** bind Docker's port publishing to loopback
only, rather than relying on `ufw` rules that never covered the
right chain:

```yaml
ports:
  - "127.0.0.1:8090:8090"
  - "127.0.0.1:8091:8091"
```

This makes the ports genuinely unreachable from anything but the VM
itself - including the Windows host's direct Internal-switch path -
while an SSH tunnel (`ssh -L 8091:127.0.0.1:8091 ...`) remains the way
to view TANNER's web UI when needed.

## Operational notes

- `tanner_web` (port 8091) is a human-facing status page only -
  SNARE never queries it, and it plays no part in classification or
  logging. It's stopped by default (`docker compose stop tanner_web`)
  and started on demand via SSH tunnel when its stats are needed.
  `tanner`, `tanner_api`, `tanner_redis`, and `tanner_phpox` are the
  services SNARE actually depends on and stay running continuously.
- The `wp-decoy` WordPress + MySQL containers were stopped
  (`docker compose stop`) once the clone was successfully captured by
  SNARE - a live WordPress install with a real database has no
  further purpose after cloning and is unnecessary attack surface to
  leave running. The temporary `ufw allow 8080/tcp` rule opened for
  the clone step was removed at the same time. The containers are
  left in place (not removed) so the page can be recloned later if
  the decoy content needs to change.

## Firewall

New NAT and firewall rules added to support the web honeypot:

- pfSense NAT: `WAN:80 -> 10.20.20.11:80`
- Home router: WAN port 80 -> `192.168.10.114:80` (no ISP reservation
  on port 80, unlike SSH's 22 - mapped directly, no workaround port
  needed)
- DMZ firewall rule: `10.20.20.11 -> 10.20.30.10:5044` (Filebeat to
  Logstash), matching the same narrow, IP-specific pattern used for
  Cowrie's equivalent rule
- Host-level `ufw` on the Web-honeypot VM, restricting SSH (22) plus
  the loopback-only exception for 8090/8091 - see the networking-gap
  section above for why this needed revisiting

## Verification

Confirmed reachable externally via mobile data and canyouseeme.org,
serving the cloned WordPress decoy correctly through SNARE at
`http://<public-ip>/`.

## Log pipeline to ELK

See [ELK Stack](elk-stack.md) for the Filebeat/Logstash/Kibana side of
the web honeypot's data - the `tanner-*` index, GeoIP field mapping,
and dashboard panels.
