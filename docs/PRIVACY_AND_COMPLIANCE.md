# Privacy, security and compliance

A system that can follow vehicles needs explicit limits. This document states what **is implemented** and what is **planned**.

## 1. Purpose limitation

TraceNet processes **vehicles and number plates only** for traffic management, public safety and planning. There is no face
recognition, no biometric processing, and no individual-level profiling beyond the vehicle journey.

## 2. Implemented controls

| Control | Detail | Code |
| --- | --- | --- |
| **Authentication** | JWT with issuer, issued-at and expiry; token valid only while the user exists, is active and still holds the same role (directory cache 5 s) | `backend/api/auth.py` |
| **Password storage** | Salted PBKDF2-SHA256, 200,000 rounds; never plaintext | `db/schema.sql`, `auth.py` |
| **Role-based access** | `camera_admin`: cameras, ingestion, users, watchlist. `law_enforcement`: search, tracking, alerts | `require_roles(...)` |
| **Tamper-evident audit** | `row_hash = sha256(prev_hash │ user │ plate │ ts │ reason)`, appended under an advisory lock; `GET /api/v1/audit/verify` re-computes the chain | `backend/api/audit.py` |
| **Admin event log** | User creation, role changes, deactivation, password resets | `admin_events`, `users.py` |
| **Transient state** | Live vehicle window is soft state (30 min) and rebuilt from traffic | `backend/fusion/redis_state.py` |
| **Metadata-first** | Upstream traffic is JSON sightings, not video | architecture |
| **Secrets** | Environment only; `.env` files git-ignored | `.gitignore` |
| **Web hardening** | `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`, restrictive `Permissions-Policy` | `vercel.json` |
| **CORS allow-list** | Explicit origins (plus optional preview regex) | `TRACENET_CORS_ORIGINS` |

## 3. Planned / not yet implemented (be honest with evaluators)

| Item | Status |
| --- | --- |
| Automatic **retention and auto-purge** of observations and trajectories (DPDP storage limitation) | Design intent; **not implemented** in code |
| Mandatory **reason code** for every lookup | The audit log stores a free-text reason (defaults to "unspecified"); enforcing it is planned |
| Encryption at rest, field-level plate hashing | Planned (relies on database/disk encryption today) |
| Moving WebSocket/evidence tokens out of query strings | Planned (they appear in access logs today) |
| Multi-person approval for sensitive watchlist entries | Planned |
| Removing built-in demo credentials for production | Required before any real deployment |

## 4. DPDP Act, 2023 alignment (design view, not legal advice)

| DPDP principle | How TraceNet is designed to meet it | Status |
| --- | --- | --- |
| Lawful, specific purpose | Purpose-bound to traffic management and public safety | Documented |
| Data minimisation | Plates and vehicle class only; no faces; metadata not video | Implemented in design |
| Storage limitation | Retention windows and auto-purge | **Planned** |
| Security safeguards | RBAC, hashing, audit chain, secrets management | Implemented (see §2) |
| Accountability | Hash-chained audit of every query, admin event log | Implemented |
| State instrumentality use | Intended for authorised police / ICCC deployment with legal basis | Deployment responsibility |

A real deployment would add a data-protection impact assessment, retention schedule, and legal sign-off by the operating
authority.

## 5. Misuse resistance

- Every lookup is attributable to a user and tamper-evident.
- Deactivating a user or changing a role takes effect within seconds.
- Watchlist changes are restricted to administrators.
- Demo data is synthetic or stock footage and describes no real person.

## 6. Threats considered

| Threat | Mitigation |
| --- | --- |
| Insider browses vehicles without reason | Audit chain + attributable queries |
| Audit log edited after the fact | Any edit breaks every later hash |
| Token theft | Short TTL (default 8 h, configurable), role re-check on each request |
| Plate spoofing / cloning | Road-physics clone detection |
| Camera or network outage | Durable queues, health alerts, replay |
