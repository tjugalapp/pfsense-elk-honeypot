# Web Honeypot (SNARE + TANNER)

## Why a web honeypot

Cowrie only covers SSH and Telnet, but a lot of internet scanning goes
after web services instead: WordPress logins, admin panels, old CMS
endpoints. I wanted to cover that too.

I picked SNARE + TANNER because TANNER classifies requests
automatically (SQL injection, XSS, LFI/RFI and so on). That gives a
starting point for investigations instead of only raw request logs.
SNARE is the part that serves a cloned web page, TANNER is the backend
that analyzes the requests.

## VM

- Ubuntu Server 24.04 LTS, Generation 1, 2 vCPU, 4 GB RAM
- One network card, on Honeypot-DMZ only, static IP 10.20.20.11/24,
  gateway/DNS 10.20.20.2
- Its own VM, separate from Cowrie. TANNER's documentation recommends
  running TANNER on a different host than SNARE in production, but here
  SNARE and TANNER share one VM. I only kept Cowrie fully separate,
  since its VM only has 2 GB RAM and I wanted one service per VM

## Installation

Docker and the Docker Compose plugin installed via `get.docker.com`.

### TANNER

Cloned to `~/tanner`, built and run with docker compose from
`~/tanner/docker` (five services: `tanner`, `tanner_api`, `tanner_web`,
`tanner_redis`, `tanner_phpox`). Ports 8090 (tanner) and 8091
(tanner_web) were published to the host at first.

Two problems during the build, both config issues and not bugs in the
code:

- `tanner_redis` was set to `read_only: true`, but Redis still tried to
  save snapshots to disk and crashed. I fixed it by adding
  `command: ["redis-server", "--save", "", "--appendonly", "no",
  "--stop-writes-on-bgsave-error", "no"]` so it never writes to disk.
- The `tanner` log folder (`/tmp/tanner`) was a `tmpfs` mount, so the
  logs disappeared on restart and couldn't be seen from the host. I
  replaced it with a bind mount (`./data/tanner:/tmp/tanner`), created
  the folder with `mkdir -p data/tanner` and ran `chown 65534:65534`
  to match the container's user.

### SNARE

Cloned to `~/snare` and pointed at TANNER with `TANNER=10.20.20.11` in
`docker-compose.yml` (SNARE hardcodes port 8090, so only the IP is
needed).

I had to fix two bugs in SNARE itself to get it working:

- **The build failed on an old base image.** The Dockerfile used
  `python:3.6-alpine3.8`, which is end-of-life, and `requirements.txt`
  needed `gitpython==3.1.30`, which doesn't support Python 3.6. I
  changed the base image to `python:3.9-alpine`.
- **Broken HTTP responses from cloned pages**
  (`snare/tanner_handler.py`, `parse_tanner_response()`, the
  `detection["type"] == 1` branch). SNARE copied all the saved headers
  from the cloned page, including `Transfer-Encoding: chunked`, which
  clashes with the `Content-Length` header SNARE sets itself. Browsers
  and `curl` couldn't read the body, even though a `HEAD` request
  looked fine. I added a `skip_headers` filter that drops
  `transfer-encoding`, `content-length`, `content-encoding` and
  `connection` before the headers are copied.

By default SNARE clones `http://example.com` as its decoy page.

## Changing the decoy page to WordPress

`example.com` is just a static page, so TANNER had nothing to classify
except "index": no login form, no CMS, no plugin paths. To get more
variety I set up a throwaway WordPress site just to clone. I didn't
want to clone a real site, since that raises legal and consent
questions even in a private lab.

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

It was installed as a generic "Company Blog" with WordPress's default
content and no real data. Pointing SNARE at it caused three new
problems. Each fix looked like it solved everything, until the next
problem showed up underneath:

1. **Docker's build cache hid a failed clone.**
   `docker compose build --build-arg PAGE_URL=10.20.20.11:8080` said it
   succeeded (`RUN clone` finished with exit code 0), but the `clone`
   tool had actually failed to reach the WordPress container and just
   swallowed the error. Docker cached the empty result, so every
   rebuild repeated the same failure. Fix: `sudo ufw allow 8080/tcp` so
   the build could reach the site, and `docker compose build
   --no-cache ...` to force a real clone.
2. **The port disappeared from the folder name.** After a working
   clone, SNARE still failed with `--page-dir: 10.20.20.11:8080 does
   not exist`. In SNARE's `clone.py`,
   `self.target_path = '/opt/snare/pages/{}'.format(self.root.host)`
   uses `.host`, which drops the port. So the page was saved in
   `/opt/snare/pages/10.20.20.11`, while the Dockerfile's `CMD` used the
   full `$PAGE_URL` with the port as `--page-dir`, a folder that didn't
   exist. `example.com` never hit this because it has no port. I fixed
   it with a `command:` override in `docker-compose.yml` that sets the
   right `--page-dir`, without the port.
3. **SNARE was running, but requests from outside were refused.** SNARE
   started fine (`Running on http://0.0.0.0`), but `curl` against the
   DMZ IP failed right away. I went through it step by step: the ufw
   rules looked fine, Docker's NAT rule pointed at the right container
   IP, and curling the container's bridge IP directly gave a 200 OK. So
   the problem was somewhere between the DMZ interface and the
   container. `sudo iptables -L FORWARD` showed
   `Chain FORWARD (policy DROP)`. Turning on `ufw` after Docker was
   already running had set the `FORWARD` policy to deny, which blocks
   all traffic forwarded to containers, even though the `DOCKER-USER`
   rules looked fine. Fix: `sudo ufw default allow routed` and
   `sudo ufw reload`. Then I tested from outside over mobile data.

**What I take from this:** if you turn on `ufw` on a host where Docker
is already running, check `iptables -L FORWARD`. Container port
forwarding can break without `ufw status` or `DOCKER-USER` showing
anything strange.

## Another gap: Hyper-V "Internal" switches

When I tested TANNER's web UI, I could open it straight from the
Windows host's browser at `http://10.20.20.11:8091`, without an SSH
tunnel. That shouldn't have worked, since I had a `ufw` rule meant to
limit that port to localhost.

There were two separate reasons:

- The DMZ and MGMT switches in Hyper-V are **Internal**, not
  **Private**. An Internal switch gives the Hyper-V host its own network
  card on that network, so my host machine always had a direct path
  into the DMZ, past pfSense. For a lab that's an OK tradeoff since I
  trust my own machine, but it means pfSense's segmentation only covers
  traffic that actually goes through pfSense.
- The `ufw allow from 127.0.0.1 to any port 8090/8091` + `deny
  Anywhere` rules never applied to Docker's ports in the first place.
  Docker publishes ports (`0.0.0.0:8090->8090`) with `iptables` DNAT
  rules, and that traffic goes through the `FORWARD` chain, not
  `INPUT`, which is what `ufw`'s port rules cover. Same root cause as
  problem 3 above.

**The right fix** is to bind the Docker ports to loopback only, instead
of relying on `ufw`:

```yaml
ports:
  - "127.0.0.1:8090:8090"
  - "127.0.0.1:8091:8091"
```

Then nothing outside the VM can reach the ports, not even the Windows
host through the Internal switch. An SSH tunnel
(`ssh -L 8091:127.0.0.1:8091 ...`) is then the way to view TANNER's
web UI.

## Running it

- `tanner_web` (port 8091) is only a status page for humans. SNARE
  never uses it. I kept it stopped by default
  (`docker compose stop tanner_web`) and started it through an SSH
  tunnel when I wanted to look at the stats. `tanner`, `tanner_api`,
  `tanner_redis` and `tanner_phpox` are what SNARE actually needs, so
  those ran all the time.
- I stopped the `wp-decoy` WordPress + MySQL containers
  (`docker compose stop`) once SNARE had cloned the page. A live
  WordPress with a real database had no purpose after that and was just
  extra attack surface. I removed the temporary `ufw allow 8080/tcp`
  rule at the same time. The containers were kept (not removed), so the
  page could be cloned again if needed.

## Firewall

New rules for the web honeypot:

- pfSense NAT: `WAN:80 -> 10.20.20.11:80`
- Home router: WAN port 80 -> `192.168.10.114:80` (no ISP block on port
  80, unlike 22, so no workaround needed)
- DMZ firewall rule: `10.20.20.11 -> 10.20.30.10:5044` (Filebeat to
  Logstash), same narrow pattern as Cowrie's rule
- `ufw` on the web honeypot VM: SSH (22) plus the localhost-only
  exception for 8090/8091. See the section above for why that needed
  another look

## Testing

Tested from outside over mobile data and with canyouseeme.org. SNARE
served the cloned WordPress page at `http://<public-ip>/`.

## Log pipeline to ELK

See [ELK Stack](elk-stack.md) for the Filebeat/Logstash/Kibana side:
the `tanner-*` index, GeoIP mapping and dashboard panels.
