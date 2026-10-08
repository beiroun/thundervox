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
# every host path the core mounts has to exist before the first `docker compose run core` - compose refuses to
# start the container otherwise (create_host_path: false). The edge proxy runs as uid 1001 (the core's uid, so
# the core can read the SIP certificate) and needs its data directory to belong to that uid
touch local.cfg tls.cfg
mkdir -p edge-data && sudo chown -R 1001:1001 edge-data
# the core's config templates ship inside its image - one source, no copies in this repository
docker compose run --rm --no-deps --entrypoint cat core /etc/kamailio/local.cfg.example > local.cfg
docker compose run --rm --no-deps --entrypoint cat core /etc/kamailio/tls.cfg.example > tls.cfg
#   in local.cfg: fill TVX_SIP_DOMAIN / TVX_PUBLIC_IP; leave the switches off for the first run
docker compose run --rm --no-deps core -c -f /etc/kamailio/kamailio.cfg   # config check: must end without "ERROR"
docker compose up -d
docker compose logs -f core | grep --line-buffered TVX
```

`tls.cfg` is created even with TLS switched off: the core mounts it as a file. Compose does not create missing
host paths here, so a forgotten file stops `up` with "bind source path does not exist" instead of turning into
an empty directory (that is how the first host got a `tls.cfg` directory and a core that died with "cannot make
tmp file" once `TVX_TLS` was switched on).

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

Needs server 0.4, console 0.2 and core 0.10 (images pinned in `docker-compose.yml`); the Integration page (service tokens, wake push) needs server 0.6 / console 0.7 / core 0.11, see below. Order:

1. In `.env`: `COMPOSE_PROFILES=provisioning`; `TVX_DB_PASSWORD` and `TVX_SIP_DB_PASSWORD` (`openssl rand -hex 24`
   each - hex, because `TVX_SIP_DB_PASSWORD` also goes into `local.cfg`, where `!` would break the line);
   `TVX_JWT_SECRET` (`openssl rand -hex 32`); `TVX_SUPERADMIN_LOGIN` and `TVX_SUPERADMIN_PASSWORD`. Both database
   passwords are fixed by the first start - changing them later needs `ALTER ROLE` in the database as well.
2. `docker compose up -d postgres server` – the server runs the migrations, creates the `tvx_sip` role and the
   super administrator. Check: `docker compose logs server | grep -iE "flyway|super administrator"`.
3. `docker compose up -d web edge` – `https://console.<domain>`: log in as the super administrator, add
   administrators / readers if needed.
4. In the console, **SIP numbers**: create a number for every panel and softphone (number and password are
   generated unless typed), enter number, SIP domain and password into each device. Devices keep registering
   without a password until step 5 - the core does not check yet.
5. In `local.cfg` (refresh the template from the new image with the `cat` command above and carry the values
   over): `#!define TVX_PROVISIONING` (exactly one `#` - the template ships the line as `##!define`) and the
   `TVX_DB_URL` substdef with the `tvx_sip` password; `TVX_SIP_DOMAIN` must equal `TVX_SIP_HOST` of `.env` - it
   is the digest realm the passwords were hashed with. Then `systemctl reload thundervox` (or by hand: the config
   check, then `docker compose up -d --force-recreate core`). From now on REGISTER and INVITE without valid
   credentials get `401` / `407`, and the console shows who is online. Devices that registered before step 5
   stay registered until their next re-REGISTER, which then has to authenticate.
6. Prove that the **running** core has the switch - the config check only proves that the file parses:

   ```bash
   docker exec thundervox-core kamcmd -s unix:/tmp/kamailio_ctl core.ppdefines | grep TVX_      # TVX_PROVISIONING must be listed
   docker exec thundervox-core kamcmd -s unix:/tmp/kamailio_ctl core.modules | grep -E "auth_db|db_postgres"
   docker compose logs --since 5m core | grep -E "REGISTER ok|auth"    # one 401 round, then "REGISTER ok" per device
   docker compose exec postgres psql -U thundervox -d thundervox -c "select username, received, expires from location"
   ```

   `TVX_PROVISIONING` missing from the list = the core runs without it: the define is misspelled or still
   `##!define`, it sits in a file other than the mounted one (`docker inspect thundervox-core --format
   '{{range .Mounts}}{{.Source}} -> {{.Destination}}{{"\n"}}{{end}}'` shows which host file the core reads), or
   the container was never recreated after the edit (a plain `docker compose up -d` does not do that).

## Service API for the operator's backend

The operator's backend (for Modus: tv-sip in modusclientapi) provisions SIP accounts itself through
`https://server.<domain>/api/v1/service/...`. Access is by a **service token** the super administrator issues on
the console's **Integration** page (shown once, stored as a hash, revocable; any number of named tokens), sent
in the `X-SERVICE-TOKEN` header. The page also lists every call below with the public address filled in and a
copy button. Numbers are addressed by the endpoint's id in the operator's own system, the `external_id`: a panel
by its device id (Modus: the `host:port` of `controls/devices`, which selects the video shown when it calls), an
app client by the subscriber account (whom to wake with a push). The same id always gets the same number; a
disabled number comes back with the next PUT. `kind` is `PANEL` or `CLIENT`.

| Call | Meaning |
|---|---|
| `PUT /service/sip-accounts/{kind}/{external_id}`, body `{"name": "…", "rotate_password": false}` (both optional) | first call creates the number (generated password in the response), later calls return the existing account without a password; `rotate_password: true` issues a new password |
| `GET /service/sip-accounts/{kind}/{external_id}` | the number as it is - never a password; `410` when unknown |
| `DELETE /service/sip-accounts/{kind}/{external_id}` | out of service: the number is blocked, not deleted, and stays bound to the id |
| `GET /service/sip-accounts/{kind}/{external_id}/registration` | whether the endpoint is registered right now |

```bash
curl -s -X PUT -H "X-SERVICE-TOKEN: tvx_…" -H "Content-Type: application/json" \
  -d '{"name":"Маяковского 14, кв. 11"}' https://server.<domain>/api/v1/service/sip-accounts/CLIENT/1234567890
```

The response carries `username`, `realm` / `sip_domain` and - only on creation or rotation - `password`: exactly
what goes into the app or the panel. Every change made this way is in the console's audit trail under the actor
`token:<name>`.

## Wake push (the Integration page)

A call to a subscriber whose phone is asleep is parked by the core (`TVX_PUSH_WAIT`), which asks the server on
loopback to wake the callee; the server turns the numbers into the operator's ids, makes up the `call_id` (UUID)
and posts the wake request to the operator's backend. Where it goes is **not** in any file on the host: the super
administrator sets the URL, the auth header and its value, and the timeouts on the Integration page, and anyone
with the administrator role can send a **test push** there and read the **delivery log** (outcome, HTTP status,
duration, the backend's answer - kept for a week). The request body (contract v2) is shown on the page.

To switch the core's side on:

1. `.env`: `TVX_CORE_TOKEN` (`openssl rand -hex 32`), then `docker compose up -d server` - the server log says
   `Internal API enabled`.
2. `local.cfg`: the same value as `#!substdef "!TVX_CORE_TOKEN!…!g"` and `#!define TVX_PUSH_WAIT` (one `#`),
   then `systemctl reload thundervox` (config check + recreate of the core). `TVX_PUSH_URL` / `TVX_PUSH_TOKEN`
   of core 0.10 and earlier are gone - remove them from `local.cfg`.
3. On the Integration page: URL and header of the operator's backend, **Test push** → the delivery log shows
   `DELIVERED` and the backend's echo. Then enable the switch on the page.
4. A live call to an unregistered subscriber: `docker compose logs core | grep "wake accepted"` shows the
   server's answer with the `call_id`; the same call is a `LIVE` row in the delivery log.

`https://server.<domain>/api/v1/internal/…` and the console's `/api/v1/internal/…` answer `404` by design: the
internal API is reachable on loopback only, with the token.

Day-to-day start of everything: `COMPOSE_PROFILES=provisioning` in `.env`, then `docker compose up -d` (or the
system service below). The edge proxy comes up with this profile and takes over the public names:
`https://console.<domain>` is the console, `https://server.<domain>` the API. Nothing listens on a public port
except the edge and the SIP core.

## System service

`thundervox.service` runs this directory as one unit: config check of the core, then `docker compose up -d`;
stop is `docker compose down`. Which services start is `COMPOSE_PROFILES` in `.env` - the same set a manual
`docker compose` here sees. The committed unit points at `/opt/thundervox/deploy`; installing renders it with the
real path of the checkout:

```bash
cd /opt/thundervox/deploy               # wherever the umbrella repository is cloned
sed "s#/opt/thundervox/deploy#$PWD#" thundervox.service > /etc/systemd/system/thundervox.service
systemctl daemon-reload
systemctl enable --now thundervox
systemctl status thundervox             # "active (exited)" is right: the unit is a oneshot, the containers run
```

| Action | Command |
|---|---|
| start / stop the whole system | `systemctl start thundervox` / `systemctl stop thundervox` |
| apply a `git pull`, pulled images, an edited `.env` or `local.cfg` | `systemctl reload thundervox` (recreates what changed, and the core every time) |
| logs of one service | `docker compose logs -f <service>` (the unit itself logs only the compose calls: `journalctl -u thundervox`) |

`reload` recreates the core container every time, even when nothing else changed: Compose recreates a container
only when the service definition or the image changed, and an edited bind-mounted `local.cfg` is neither - without
the forced recreation a switch flipped in `local.cfg` would never reach the running core (Kamailio reads its config
at start only). With `TVX_PROVISIONING` the registrations survive it, they are in PostgreSQL; without it devices
re-REGISTER within their own interval.

`stop` removes the containers, so their `docker compose logs` go with them; the `postgres-data` volume and every
file in this directory stay. A crashed container is restarted by Docker itself (`restart: always`), the unit is
not involved. Re-render the unit after a change of `thundervox.service` in the repository (same `sed`, then
`systemctl daemon-reload`).

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

With the system service installed, the last two lines are `systemctl reload thundervox` - it runs the same check,
recreates what changed and always recreates the core (see "System service").

Core and rtpengine always move together (same repository, same tag). A `kamailio.cfg` change ships as a new core
tag; a `local.cfg` change is a config check plus `docker compose up -d --force-recreate core` - a plain `up -d`
sees no change in the service definition and leaves the running core, with its old config, alone.

**Server 0.6 / console 0.7 / core 0.11 (the Integration page).** `.env`: drop `TVX_SERVICE_TOKEN`, add
`TVX_CORE_TOKEN`; the compose file passes `TVX_PUBLIC_API_URL` from `TVX_SERVER_HOST`. The service API refuses
every call until the super administrator issues a token on the Integration page - do that before the operator's
backend is switched to the new token. `local.cfg`: `TVX_PUSH_URL` / `TVX_PUSH_TOKEN` are replaced by
`TVX_CORE_TOKEN` (see "Wake push"); with `TVX_PUSH_WAIT` off nothing else changes for the core.

### Pushing from the laptop (`push.sh`)

The same update without logging in by hand: `push.sh` uploads the deploy files over sftp and runs the
steps above on the host - `docker compose pull`, then `systemctl reload thundervox` (changed containers are
recreated, the core always), then `docker compose ps`. Where the host is and how to log in come from
`deploy/.deploy.env` (gitignored; copy `.deploy.env.example` and fill `TVX_DEPLOY_TARGET`, `TVX_DEPLOY_PORT`,
`TVX_DEPLOY_DIR` and `TVX_DEPLOY_KEY`). sftp runs in batch mode and cannot ask for a password, so the login is
a key: `ssh-keygen -t ed25519 -f ~/.ssh/thundervox -N ''`, then `ssh-copy-id -i ~/.ssh/thundervox.pub -p <port>
<user@host>` once (that asks the password), then `TVX_DEPLOY_KEY=~/.ssh/thundervox`. A password is possible
instead (`TVX_DEPLOY_PASSWORD`, needs `sshpass` on the laptop), the key is cleaner.

```bash
deploy/push.sh                 # docker-compose.yml, Caddyfile, thundervox.service -> pull -> reload
deploy/push.sh local.cfg       # one named file instead; also for .env / tls.cfg, on purpose only - the live
                               # copies belong to the host and a stale laptop copy would overwrite them
deploy/push.sh --restart       # stop + start of the whole stack instead of a reload: every call drops
deploy/push.sh --dry-run       # show the sftp batch and the remote commands, change nothing
```

When `thundervox.service` is among the sent files the unit is re-rendered with the host's path and
`systemctl daemon-reload` runs, as in "System service". The files still go through git as before - the script
only replaces the `git pull` on the host with an upload of what is on the laptop, committed or not.

## Rollback

Revert the tag in `docker-compose.yml` (or `git revert` the bump), `git pull` on the server,
`docker compose up -d <service>`. The previous image is still in the local cache, so the rollback is seconds.
Database migrations are forward-only: rolling the server back past a migration needs a restore of
`postgres-data`, not just an older image.

## Checks

```bash
docker compose ps
test -f local.cfg -a -f tls.cfg || echo "local.cfg or tls.cfg is missing or is a directory"
docker compose logs --since 10m core | grep TVX                    # routing decisions, one line per step
docker exec thundervox-core kamcmd -s unix:/tmp/kamailio_ctl ul.dump      # registrations
docker exec thundervox-core kamcmd -s unix:/tmp/kamailio_ctl dlg.list     # live calls
docker exec thundervox-core kamcmd -s unix:/tmp/kamailio_ctl rtpengine.show all
curl -s http://127.0.0.1:8080/api/v1/system/health                 # server, directly on loopback
curl -s http://127.0.0.1:8081/api/v1/info                          # console nginx -> server
curl -s https://console.<domain>/api/v1/info                       # edge -> console nginx -> server
curl -s https://server.<domain>/api/v1/info                        # edge -> server
curl -s -o /dev/null -w '%{http_code}\n' https://server.<domain>/api/v1/internal/push/wake   # 404: closed on the edge
docker compose exec edge find /data/caddy/certificates -name '*.crt'   # which names hold a certificate
```

SIP on the wire: `sngrep` on the host (`apt install sngrep`) shows every dialog; one call is one `grep <Call-ID>`
in the core log.

## Backup

What matters on the host: `.env`, `local.cfg`, `tls.cfg` and the `postgres-data` volume (Docker name `thundervox_postgres-data`; accounts, HA1
hashes, registrations). `docker compose exec postgres pg_dump -U thundervox thundervox > thundervox-$(date +%F).sql`
for a logical dump. Images are reproducible from the registry and need no backup.

`edge-data/` is worth keeping too, though it is not critical: the certificates would be re-issued on a fresh
host automatically. What a backup saves is the ACME account and a brush with the CA's rate limits when a host
is rebuilt repeatedly.
