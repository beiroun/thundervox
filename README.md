# ThunderVox

**The REAL SIP server** — a thin, scalable SIP endpoint platform for physical
devices (intercoms, elevators, gates, SOS points) that place calls to mobile apps
which are *not* continuously registered.

Built on **Kamailio** (signaling + registrar) and **rtpengine** (media + NAT).

---

## The core idea: push-wait

Mobile apps cannot hold a permanent SIP registration — iOS kills background
sockets, and keeping 100k+ apps registered would be pointless load. So the callee
is normally *offline* when an endpoint calls it. ThunderVox bridges that gap by
**parking** the call and **waking** the device with a push:

```
Intercom (always registered)            ThunderVox               Mobile app (asleep)
        │                                    │                            │
        │ 1. INVITE sip:1001@tvx ───────────▶│                            │
        │                                    │ 2. lookup(1001) → offline  │
        │                                    │ 3. ts_store()  (park call) │
        │◀──────── 180 Ringing ──────────────│                            │
        │                                    │ 4. push (HTTP) ───────────▶│ wakes up
        │                                    │                            │
        │                                    │◀──── 5. REGISTER 1001 ─────│
        │                                    │ 6. ts_append() → resume    │
        │                                    │ 7. INVITE ────────────────▶│
        │◀════════ 8. RTP/RTCP media via rtpengine ══════════════════════▶│
```

- **1–3** — the endpoint calls; the app is not in the location table, so the INVITE
  transaction is **parked** (`tsilo` module), keyed by the callee AoR.
- **4** — a push notification wakes the device.
- **5–7** — the app REGISTERs; `tsilo` **resumes** the parked INVITE and routes it to
  the freshly-registered contact.
- **8** — `rtpengine` relays and NAT-fixes the audio/video for both legs.

If the callee is **already registered** (e.g. a Zoiper softphone during testing),
steps 3–6 are skipped and the INVITE is relayed immediately.

---

## Components

| Path | Role |
|---|---|
| `deployment/configuration/kamailio.cfg` | SIP signaling, registrar, NAT detection, push-wait routing |
| `deployment/configuration/local.cfg.example` | Template for the site-local values (public IP, push URL, service token) |
| `deployment/docker-compose.yml` | Kamailio + rtpengine, host-networked |
| `deployment/.env.example` | Template for the host address docker compose feeds to rtpengine |
| *(external)* push gateway | HTTP endpoint that delivers APNs/FCM pushes — **not** in this repo |

**NAT strategy:** the *received* method for registered devices
(`fix_nated_register` + `received_avp`) and the *alias* method for the other
in-dialog party (`set_contact_alias` / `handle_ruri_alias`). Media is always
anchored on `rtpengine`, so RTP never needs a direct path between the two UAs.

---

## Configure

Site-local values are **not** in this repository — no address, endpoint or token
is committed. Create both files from their templates:

```bash
cd deployment
cp configuration/local.cfg.example configuration/local.cfg
cp .env.example .env
```

`configuration/local.cfg` is pulled into `kamailio.cfg` by `include_file` and
defines:

| Constant | Meaning |
|---|---|
| `TVX_PUBLIC_IP` | Server public IP — advertised in SIP and used by rtpengine. |
| `TVX_PUSH_URL` | Push gateway HTTP endpoint. |
| `TVX_PUSH_TOKEN` | Service token sent as `X-SERVICE-TOKEN`; must equal `SERVICE_TV_SIP_TOKEN` on the backend. |

`.env` holds `TVX_PUBLIC_IP` for docker compose, which passes it to rtpengine as
`RTPENGINE_PUBLIC_IP`. **Both copies of `TVX_PUBLIC_IP` must be the same IP** —
compose refuses to start when `.env` is missing.

Non-secret tunables stay in `kamailio.cfg` itself: `TVX_RTP_FLAGS` (rtpengine
per-leg flags, plain RTP/AVP with ICE stripped) and `TVX_RTPENGINE_SOCK`.

Both `local.cfg` and `.env` are gitignored — keep them that way.

---

## Run

```bash
cd deployment
docker compose up -d
docker compose logs -f kamailio     # watch [TVX] routing logs
```

**Firewall — open to the internet:**

| Port | Proto | Purpose |
|---|---|---|
| 5060 | UDP | SIP signaling |
| 29000–30000 | UDP | RTP/RTCP media (rtpengine range; narrow test pool, widen for production) |

---

## Test with Zoiper (softphone stands in for the mobile app)

1. Register a Zoiper account against `TVX_PUBLIC_IP:5060` as user `1001`.
   Because it stays registered, it exercises the **online** path (no push).
2. Point an intercom/second softphone at the server and dial `sip:1001@TVX_PUBLIC_IP`.
3. Expect two-way audio (rtpengine relays it). Watch `[TVX]` logs for the routing
   decision (`callee ONLINE` vs `callee OFFLINE, parking + push`).

To exercise the **push-wait** path, let the callee go **unregistered**, place the
call (caller hears ringing, a push fires), then REGISTER the callee — the parked
call connects.

---

## Known limitations / TODO

- **Synchronous push.** `route[PUSH]` calls the gateway synchronously and can block
  a SIP worker for up to `connection_timeout` (2s). Production: async push-gateway
  microservice, fire-and-forget.
- **No auth/ACL.** Any host may originate a call. Add authentication / source ACL
  before public exposure.
- **No persistence.** `usrloc` is in-memory (`db_mode=0`); registrations are lost on
  restart. Add DB-mode + Redis for HA.
- **Single node.** One Kamailio + one rtpengine. Horizontal scale (dispatcher to a
  media pool, HA registrar) is the v1 target.
- **rtpengine image not pinned.** Pin a tag/digest before production.

---

## Roadmap

- **v0** — Docker Compose, single host *(current)*.
- **v1** — Kubernetes: stateless Kamailio edge (HA), rtpengine media pool behind
  `dispatcher`, Redis-backed presence, async push service, DB-backed location.
