# ThunderVox deployment runbook

One host, Docker Compose, published images. The host keeps this directory only: `docker-compose.yml`, `.env`,
`local.cfg`. Component sources are never cloned on the server and nothing is built there.

## Prerequisites

- Linux host with Docker Engine and the Compose plugin (`docker compose version`).
- DNS name of the SIP service pointing at the host (devices register and dial by name; `TVX_SIP_DOMAIN`).
- Firewall open to the internet: UDP+TCP 5060 (SIP), UDP 29000–30000 (RTP), TCP 80 (console). Everything else
  stays on loopback.
- If the GHCR packages are private: `docker login ghcr.io` with a token that has `read:packages`. Public packages
  pull anonymously.

## First bring-up (bare core, no provisioning)

```bash
git clone https://github.com/beiroun/thundervox.git /opt/thundervox
cd /opt/thundervox/deploy
cp .env.example .env            # fill TVX_PUBLIC_IP (and TVX_LOCAL_IP on a 1:1 NAT host)
cp local.cfg.example local.cfg  # fill TVX_SIP_DOMAIN / TVX_PUBLIC_IP; leave the switches off for the first run
docker compose pull
docker compose run --rm core -c -f /etc/kamailio/kamailio.cfg   # config check: must end without "ERROR"
docker compose up -d
docker compose logs -f core | grep --line-buffered TVX
```

Then the proof: register a softphone, register the intercom panel, place a call. Expected log lines are described
in `thundervox-core/README.md` ("Test with Zoiper").

### Moving from `thundervox-core/deployment` (hosts set up before 0.7.0)

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
5. In `local.cfg`: `#!define TVX_PROVISIONING` and `TVX_DB_URL` with the `tvx_sip` password;
   `docker compose run --rm core -c -f /etc/kamailio/kamailio.cfg`, then `docker compose up -d core`.
   From now on REGISTER without credentials gets `401`, unknown accounts cannot register.

Day-to-day start of everything: `docker compose --profile provisioning up -d`.

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
curl -s http://127.0.0.1:8080/api/v1/system/health                 # server (provisioning profile)
curl -s http://127.0.0.1/api/v1/info                               # console -> server through nginx
```

SIP on the wire: `sngrep` on the host (`apt install sngrep`) shows every dialog; one call is one `grep <Call-ID>`
in the core log.

## Backup

What matters on the host: `.env`, `local.cfg` and the `postgres-data` volume (accounts, HA1 hashes,
registrations). `docker compose exec postgres pg_dump -U thundervox thundervox > thundervox-$(date +%F).sql`
for a logical dump. Images are reproducible from the registry and need no backup.
