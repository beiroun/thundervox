# ThunderVox

**SIP endpoint platform.** Intercoms, elevators, gates and SOS points place
calls to mobile apps that are *not* continuously registered. ThunderVox parks
the call, wakes the phone with a push, and connects the two when the app
registers. Built on **Kamailio** and **rtpengine**, with its own provisioning
server and web console.

This is the **umbrella repository**: the project charter, the architecture
overview, licensing and the deployment of the whole system. Code lives in the
component repositories below.

---

## Components

| Repository | Role | Image | Status |
|---|---|---|---|
| [`thundervox-core`](https://github.com/beiroun/thundervox-core) | SIP signaling, registrar, NAT traversal, push-wait routing; media relay | `ghcr.io/beiroun/thundervox-core`, `ghcr.io/beiroun/thundervox-rtpengine` | v0.6 — stable calls on a test host with a real intercom panel |
| [`thundervox-server`](https://github.com/beiroun/thundervox-server) | Provisioning: tenants, sites, devices, app clients, SIP accounts; the core's authentication source; admin and service API | `ghcr.io/beiroun/thundervox-server` | in design |
| [`thundervox-web`](https://github.com/beiroun/thundervox-web) | Operator console: devices, clients, registrations, live calls | `ghcr.io/beiroun/thundervox-web` | in design |
| `thundervox` (this repo) | Charter, architecture, licensing, `docker-compose` of the whole system, deployment runbook | — | documents only |

## How it fits together

```
 intercom / elevator / SOS panel        mobile app (asleep until pushed)
            │ SIP (digest auth)                  ▲ SIP after wake-up
            ▼                                    │
   ┌──────────────────┐   RTP   ┌───────────┐   │
   │  thundervox-core │◀───────▶│ rtpengine │◀──┘  media anchored on the relay
   │  (Kamailio)      │         └───────────┘
   └───────┬──────────┘
           │ auth_db / usrloc (SQL)          JSON-RPC (localhost)
           ▼                                   ▲
   ┌──────────────────┐                ┌───────┴───────────┐      ┌────────────────┐
   │   PostgreSQL     │◀──────────────▶│ thundervox-server │◀────▶│ thundervox-web │
   └──────────────────┘   JDBC         └───────┬───────────┘ /api └────────────────┘
                                               │ service API
                                               ▼
                                     operator backend / push gateway (external)
```

- **Push-wait** is the core idea: the callee is normally offline, so the INVITE
  is parked, the device is woken by a push, and the call is resumed when the
  app registers. Details in `thundervox-core`.
- **Devices register directly** with the core — no PBX in between. The platform
  is the endpoint layer on top of plain SIP, and the real product is the set of
  vendor adapters that make real-world panels behave.
- **The core talks only to PostgreSQL** for authentication and registrations;
  there is no HTTP in the SIP path. The server owns the schema and reaches the
  core through a local JSON-RPC socket for live state.

## Principles

- **Own images for everything.** Each component repository builds its image
  from pinned sources (Kamailio and rtpengine are built from source) and
  publishes it to GHCR on a tagged release. Only Docker Official Images are
  used as bases.
- **The host keeps only this repository.** A `docker-compose.yml`, a `Caddyfile`,
  `.env` and the core's `local.cfg` / `tls.cfg` — no component sources, no builds
  on the server. Updating a component means bumping an image tag.
- **One public name per face, one edge.** `sip.<domain>` is SIP,
  `console.<domain>` the operator console, `server.<domain>` the provisioning
  API. The only container on 80/443 is the edge proxy, which obtains and renews
  every certificate itself; the console, the API and the database listen on
  loopback. The SIP core uses the same certificate for SIPS on 5061.
- **One host, Docker Compose, host networking.** Kubernetes comes with the
  second node, not before.
- **Security by provisioning, not by file.** Credentials live in the database
  and are managed through the console; the config files carry no secrets.

## Deployment (this repository)

```
deploy/
  docker-compose.yml    core, rtpengine, postgres, server, web, edge — pinned image tags, profiles
  Caddyfile             public names -> loopback services, automatic TLS
  .env.example          host names, ACME address, host IP, database passwords
  RUNBOOK.md            bring-up, TLS, migration from the old layout, update, rollback, checks, backup
```

`docker compose up -d` runs the bare core (`core` + `rtpengine`, as tested on
the stand); `docker compose --profile provisioning up -d` adds PostgreSQL, the
server, the console and the edge proxy. The profile `tls` brings up the edge
alone, for a host where only the SIP core needs a certificate. The host keeps
this directory only — see [`deploy/RUNBOOK.md`](deploy/RUNBOOK.md).

Certificates and private keys are written by the edge proxy into
`deploy/edge-data/` on the host and are never committed: this repository
contains the names, not the key material.

## Status

- 2026-10: core v0.6 — plain calls between registered devices are stable
  (NAT on both legs, cancel/bye/re-INVITE/early media, caller identity by
  registration, `403` for unregistered sources). Push-wait is implemented as a
  switch and is next in line for live testing.
- 2026-10: own images built from source (core 0.6.1), server and console
  skeletons, the deployment moved here. Next: the provisioning layer itself —
  accounts in PostgreSQL, the core authenticating against them, the console
  pages.

## License

ThunderVox is released under the **Business Source License 1.1** — see
[`LICENSE`](LICENSE). The same license and parameters apply to every
component repository. Non-production use is free; production use beyond the
Additional Use Grant requires a commercial license from the Licensor.

Third-party components (Kamailio, rtpengine, PostgreSQL, nginx, Spring,
React, …) keep their own licenses — see [`NOTICE`](NOTICE).

---

*Built by [Andrei Baranov](https://github.com/beiroun) · 84softworks*
