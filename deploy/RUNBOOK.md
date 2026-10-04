# ThunderVox deployment runbook

One host, Docker Compose, published images. The host keeps this directory only: `docker-compose.yml`,
`Caddyfile`, `.env`, `local.cfg`, `tls.cfg` and the `edge-data/` directory the edge proxy writes its
certificates into. Component sources are never cloned on the server and nothing is built there.

## Prerequisites

- Linux host with Docker Engine and the Compose plugin (`docker compose version`).
- DNS: an A record for each public name, all pointing at this host. The names are issued a certificate over an
  HTTP challenge, so they must resolve **before** the first start of the edge container.

  | Name | `.env` | Serves |
  |---|---|---|
  | `sip.<domain>` | `TVX_SIP_HOST`, and `TVX_SIP_DOMAIN` in `local.cfg` | SIP: devices register and dial by this name |
  | `console.<domain>` | `TVX_CONSOLE_HOST` | operator console (and its own `/api` to the server) |
  | `server.<domain>` | `TVX_SERVER_HOST` | provisioning API under its own name |

- Firewall open to the internet: UDP+TCP 5060 (SIP), UDP 29000–30000 (RTP), TCP 5061 (SIPS, with `TVX_TLS`),
  TCP 80 and 443 (edge proxy). Port 80 stays open even though everything redirects to HTTPS - the ACME
  challenge needs it. Everything else stays on loopback: the database, the server API and the console's nginx.
- If the GHCR packages are private: `docker login ghcr.io` with a token that has `read:packages`. Public packages
  pull anonymously.

## First bring-up (bare core, no provisioning)

```bash
git clone https://github.com/beiroun/thundervox.git /opt/thundervox
cd /opt/thundervox/deploy
cp .env.example .env            # fill the host names, TVX_ACME_EMAIL and TVX_PUBLIC_IP (TVX_LOCAL_IP on 1:1 NAT)
docker compose pull
# the core's config templates ship inside its image - one source, no copies in this repository
docker compose run --rm --entrypoint cat core /etc/kamailio/local.cfg.example > local.cfg
docker compose run --rm --entrypoint cat core /etc/kamailio/tls.cfg.example > tls.cfg
#   in local.cfg: fill TVX_SIP_DOMAIN / TVX_PUBLIC_IP; leave the switches off for the first run
# the edge proxy runs as uid 1001 (the core's uid, so the core can read the SIP certificate) and needs its
# data directory to belong to that uid
mkdir -p edge-data && sudo chown -R 1001:1001 edge-data
docker compose run --rm core -c -f /etc/kamailio/kamailio.cfg   # config check: must end without "ERROR"
docker compose up -d
docker compose logs -f core | grep --line-buffered TVX
```

`tls.cfg` is created even with TLS switched off: the core mounts it as a file, and a bind mount of a missing
path would silently turn into a directory.

Then the proof: register a softphone, register the intercom panel, place a call. Expected log lines are described
in `thundervox-core/README.md` ("Test with Zoiper").

### Moving from `thundervox-core/deployment` (hosts set up before 0.7.0, when the core repository still carried its own compose)

The core used to run from a clone of `thundervox-core`. The configuration files are the same; only their home
changes.

```bash
cd /path/to/thundervox-core/deployment && docker compose down
cp configuration/local.cfg /opt/thundervox/deploy/local.cfg
# TVX_PUBLIC_IP from the old .env goes into /opt/thundervox/deploy/.env
cd /opt/thundervox/deploy && docker compose pull && docker compose up -d
```

The old clone can be deleted afterwards; the server does not need component sources.

## Enabling the provisioning layer (profile `provisioning`)

Arrives with core 0.7 and server/web 0.1. Order, once the images exist:

1. In `.env` set `TVX_DB_PASSWORD` and `TVX_SIP_DB_PASSWORD` (long random values; they never leave this host).
2. `docker compose --profile provisioning up -d postgres server` – the server runs the migrations and creates the
   `tvx_sip` role. Check: `docker compose logs server | grep -i flyway`.
3. `docker compose --profile provisioning up -d web` – console on port 80; `GET /api/v1/info` answers through nginx.
4. In the console: create the site, the devices, issue passwords; enter them into the panels and softphones.
5. In `local.cfg`: `#!define TVX_PROVISIONING` and `TVX_DB_URL` with the `tvx_sip` password (the template of
   core 0.8 documents both; refresh it from the new image with the same `cat` command when upgrading);
   `docker compose run --rm core -c -f /etc/kamailio/kamailio.cfg`, then `docker compose up -d core`.
   From now on REGISTER without credentials gets `401`, unknown accounts cannot register.

Day-to-day start of everything: `docker compose --profile provisioning up -d`. The edge proxy comes up with
this profile and takes over the public names: `https://console.<domain>` is the console,
`https://server.<domain>` the API. Nothing listens on a public port except the edge and the SIP core.

## TLS

The edge proxy (Caddy) obtains and renews every certificate itself over an ACME HTTP challenge and keeps them
in `edge-data/`. There is no certbot, no cron job and no renewal hook to maintain. Private keys never leave the
host - `edge-data/` is gitignored, and so are `.env`, `local.cfg` and `tls.cfg`.

First start, and after any change of a name in `.env`:

```bash
docker compose --profile provisioning up -d edge
docker compose logs edge | grep -iE "certificate|obtain|error"   # expect "certificate obtained successfully"
curl -sI https://console.<domain> | head -3
curl -s  https://server.<domain>/api/v1/info
```

A name that does not resolve to this host fails the challenge; Caddy then retries with a growing backoff, so
fix the DNS record and `docker compose restart edge` instead of waiting.

### SIP over TLS (`TVX_TLS`)

The core reads the certificate the edge proxy already holds for `sip.<domain>`; plain 5060 keeps working, so
panels whose firmware has no TLS are unaffected.

```bash
# 1. find the real path - it carries the ACME directory the certificate came from
docker compose exec edge find /data/caddy/certificates -name '*.crt'
# 2. put that path (with /data replaced by /tls, which is where the core mounts the same directory) into tls.cfg
# 3. switch it on in local.cfg: #!define TVX_TLS      (requires TVX_SIP_DOMAIN - a certificate is issued for a
#    name, never for an IP; without it the config check fails with a TVX_TLS_NEEDS_TVX_SIP_DOMAIN… token)
docker compose run --rm core -c -f /etc/kamailio/kamailio.cfg
docker compose up -d core
docker compose logs core | grep -i tls
```

Renewal: Caddy rewrites the files in place roughly 30 days before expiry, and the running core keeps the old
certificate in memory until it is told to re-read `tls.cfg`. A daily reload on the host covers it - idempotent,
and cheaper than discovering an expired certificate:

```
# /etc/cron.d/thundervox-tls-reload
23 4 * * * root docker exec thundervox-core kamcmd -s unix:/tmp/kamailio_ctl tls.reload >/dev/null 2>&1
```

If a reload ever does not pick a renewed file up, `docker compose up -d --force-recreate core` always does -
at the cost of the in-memory registrations (devices re-REGISTER within about ten seconds).

## Updating a component

1. Bump the image tag of the service in `docker-compose.yml` (component repositories publish
   `ghcr.io/beiroun/<repo>:X.Y.Z` on their `vX.Y.Z` tags; `latest` does not exist).
2. Commit and push the umbrella repository.
3. On the server:

```bash
cd /opt/thundervox/deploy && git pull
docker compose pull <service>
docker compose run --rm core -c -f /etc/kamailio/kamailio.cfg   # when the core or local.cfg changed
docker compose up -d <service>
```

Core and rtpengine always move together (same repository, same tag). A `kamailio.cfg` change ships as a new core
tag; a `local.cfg` change is a config check plus `docker compose up -d core`.

## Rollback

Revert the tag in `docker-compose.yml` (or `git revert` the bump), `git pull` on the server,
`docker compose up -d <service>`. The previous image is still in the local cache, so the rollback is seconds.
Database migrations are forward-only: rolling the server back past a migration needs a restore of
`postgres-data`, not just an older image.

## Checks

```bash
docker compose ps
docker compose logs --since 10m core | grep TVX                    # routing decisions, one line per step
docker exec thundervox-core kamcmd -s unix:/tmp/kamailio_ctl ul.dump      # registrations
docker exec thundervox-core kamcmd -s unix:/tmp/kamailio_ctl dlg.list     # live calls
docker exec thundervox-core kamcmd -s unix:/tmp/kamailio_ctl rtpengine.show all
curl -s http://127.0.0.1:8080/api/v1/system/health                 # server, directly on loopback
curl -s http://127.0.0.1:8081/api/v1/info                          # console nginx -> server
curl -s https://console.<domain>/api/v1/info                       # edge -> console nginx -> server
curl -s https://server.<domain>/api/v1/info                        # edge -> server
docker compose exec edge find /data/caddy/certificates -name '*.crt'   # which names hold a certificate
```

SIP on the wire: `sngrep` on the host (`apt install sngrep`) shows every dialog; one call is one `grep <Call-ID>`
in the core log.

## Backup

What matters on the host: `.env`, `local.cfg`, `tls.cfg` and the `postgres-data` volume (accounts, HA1 hashes,
registrations). `docker compose exec postgres pg_dump -U thundervox thundervox > thundervox-$(date +%F).sql`
for a logical dump. Images are reproducible from the registry and need no backup.

`edge-data/` is worth keeping too, though it is not critical: the certificates would be re-issued on a fresh
host automatically. What a backup saves is the ACME account and a brush with the CA's rate limits when a host
is rebuilt repeatedly.
