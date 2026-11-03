# ThunderVox deployment runbook

## Portable configuration reference

This public document describes the application contract, not any production
host. Server coordinates, routing, service-manager units and operating procedures
are maintained privately by the operator.

- Copy `.env.example` to ignored `.env` and configure the site. Example values
  must not replace an existing installation's settings or credentials.
- Set `TVX_TLS_DATA_DIR` to the existing certificate directory, `TVX_DB_VOLUME`
  to the existing external database volume and `TVX_SERVER_PORT` to the API listener.
  A new installation must provision its external volume before starting PostgreSQL.
- Copy `console.conf.example` to ignored `console.conf`; align its listener and
  API upstream with the site configuration. Keep the internal API blocked.
- Copy and configure `local.cfg.example` and `tls.cfg.example` in their ignored
  counterparts. `TVX_SERVER_URL` in local.cfg must match the selected API listener.
- Real certificate paths, private keys and credentials never enter Git.

Validate from this directory with
`docker compose -p thundervox -f docker-compose.yml config --quiet`.
Do not print the rendered configuration because it contains resolved secrets.

`push.sh --dry-run` previews an upload. With no file arguments it sends the
Compose file and ignored `console.conf`, then validates remotely. SSH parameters
come from ignored `.deploy.env` or environment overrides. Credentials and core
configuration are sent only when explicitly named. The script does not pull
images or activate services; `--restart` is rejected. Activation, nginx reload,
runtime checks and rollback follow the operator's private procedure.

## Service API for the operator's backend

The operator's backend provisions SIP accounts itself through
`https://server.<domain>/api/v1/service/...`. Access is by a **service token** the super administrator issues on
the console's **Integration** page (shown once, stored as a hash, revocable; any number of named tokens), sent
in the `X-SERVICE-TOKEN` header. The page also lists every call below with the public address filled in and a
copy button. Numbers are addressed by the endpoint's id in the operator's own system, the `external_id`: a panel
by its device id, an
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
  -d '{"name":"Example client"}' https://server.<domain>/api/v1/service/sip-accounts/CLIENT/example-client
```

The response carries `username`, `realm` / `sip_domain` and - only on creation or rotation - `password`: exactly
what goes into the app or the panel. Every change made this way is in the console's audit trail under the actor
`token:<name>`.

## Wake push (the Integration page)

A call to a subscriber whose phone is asleep is parked by the core (`TVX_PUSH_WAIT`), which asks the server on
loopback to wake the callee; the server turns the numbers into the operator's ids, makes up the `call_id` (UUID)
and posts the wake request to the operator's backend. Where it goes is **not** in any file on the host: the super
administrator sets the URL, the auth header and its value, and the timeouts on the Integration page, and anyone
with the administrator role can send a **test push** there; what happened to every push (outcome, HTTP status,
duration, the backend's answer - kept for a week) is on the console's **Events** page, category "Pushes". The
request body (contract v2) is shown on the Integration page.

To switch the core's side on:

1. `.env`: `TVX_CORE_TOKEN` (`openssl rand -hex 32`), then `docker compose up -d server` - the server log says
   `Internal API enabled`.
2. `local.cfg`: the same value as `#!substdef "!TVX_CORE_TOKEN!…!g"` and `#!define TVX_PUSH_WAIT` (one `#`),
   then validate and apply the core configuration under the operator's procedure. `TVX_PUSH_URL` / `TVX_PUSH_TOKEN`
   of core 0.10 and earlier are gone - remove them from `local.cfg`.
3. On the Integration page: URL and header of the operator's backend, **Test push** → the result shows
   `DELIVERED` and the backend's echo (and so does the Events page). Then enable the switch on the page.
4. A live call to an unregistered subscriber: `docker compose logs core | grep "wake accepted"` shows the
   server's answer with the `call_id`; on the Events page the call is a line with its wake push right after it.

`https://server.<domain>/api/v1/internal/…` and the console's `/api/v1/internal/…` answer `404` by design: the
internal API is reachable on loopback only, with the token.

Select application components with `COMPOSE_PROFILES=provisioning` in `.env`. The operator configures public routing:
`https://console.<domain>` is the console, `https://server.<domain>` the API. Nothing listens on a public port
except the edge and the SIP core.

## Component updates

Bump the exact image tag in docker-compose.yml after the release image exists.
Core and rtpengine move together. Applying the selected image follows the
operator's private deployment procedure. A database rollback across a migration needs
a database restore, not merely an older image. Existing domain release notes:

**Server 0.6 / console 0.7 / core 0.11 (the Integration page).** `.env`: drop `TVX_SERVICE_TOKEN`, add
`TVX_CORE_TOKEN`; the compose file passes `TVX_PUBLIC_API_URL` from `TVX_SERVER_HOST`. The service API refuses
every call until the super administrator issues a token on the Integration page - do that before the operator's
backend is switched to the new token. `local.cfg`: `TVX_PUSH_URL` / `TVX_PUSH_TOKEN` are replaced by
`TVX_CORE_TOKEN` (see "Wake push"); with `TVX_PUSH_WAIT` off nothing else changes for the core.

**Server 0.7 / console 0.9 / core 0.12 (journals, dashboard, forking).** The server adds the journal tables
(migration `V7`) and starts sampling `location` every 10 s; nothing to configure, the three retention
variables in `.env.example` are optional. The core now reports call events to the server whenever
`TVX_CORE_TOKEN` is set in `local.cfg` (`http_client` loads with the token, not only with `TVX_PUSH_WAIT`), so
set the token in `.env` and `local.cfg` together as before. Behaviour changes to know about: several devices may
register one number and all of them ring; with `TVX_PUSH_WAIT` the app client numbers (`1…`) are always parked
and woken, only devices that register after the push ring, and a second caller to a number that is being woken
hears `486`; TCP contacts leave `location` when their connection dies. The console's registration status in
the API changed shape (`online_since`, `last_online_at`, `last_offline_at` instead of `expires_at`,
`last_seen_at`) - upgrade server and console together.

**Server 0.10 / console 0.12 (the event log).** The server adds `service_call_log` (migration `V8`) and records
every call to the service API there, refused ones included; the console's new **Events** page reads it together
with registrations, calls and pushes, and the delivery log left the Integration page. Nothing to configure. The
retention variables of the journals (`TVX_*_RETENTION` in `.env.example`) now reach the server container: until
this release the compose file did not pass them, so a value set in `.env` had no effect.

**Server 0.11 / console 0.13 (quieter registrations).** A number goes offline only after 60 s missing from
`location`, so a softphone that reconnects its TCP within a minute no longer leaves the journal; the event log
shows registrations of panels only under "All" (app clients under "Registrations" and in the card of a call
through a wake push), and a panel gone offline counts as a problem. Nothing to configure.

**Server 0.12 / console 0.14 (registrations of panels only).** Migration `V9` deletes the registration
journal's lines of subscribers and numbers without an account and adds `call_log.callee_registered_at`; from then
on only panels get journal lines (every number still keeps its state). The dashboard's "registrations" are
panels coming online. Nothing to configure.

## TLS ownership

The operator supplies certificates through `TVX_TLS_DATA_DIR`, mounted read-only
at `/tls`. Set tls.cfg paths to the actual container-visible files and verify
read permissions for the core's runtime user. Renewal and application of renewed
certificates belong to the private operating procedure. Never commit key material.

## Acceptance

At the configured application endpoints, verify API info, console-to-API access,
blocked public internal routes, call journaling and a real wake push. An upload
or a configuration validation alone does not demonstrate a successful deployment.

## Backup

Securely retain ignored configuration and a PostgreSQL backup under the
operator's policy. Preserve existing database storage and certificate material.
Never use `docker compose down -v` as a migration or rollback step.
