---
name: mcp-connector
description: Expose a self-hosted, private web app to Claude as a custom remote MCP connector — the separate internet-facing service, OAuth 2.1, the read-only tool layer, Tailscale Funnel, and the isolation that keeps a compromised connector away from the app. Use when the user wants Claude/Cowork to read data from an app they host themselves, asks about custom connectors, remote MCP servers, MCP OAuth, or exposing a tailnet-only service to the internet safely. Also use when auditing an existing MCP connector against this design.
---

# Self-hosted MCP connector

A recipe for one specific, awkward shape: you host a small private app —
tailnet-only, behind Tailscale, no public surface — and you want Claude to read
its data. Doing that means putting *something* of yours on the public internet,
which is exactly what the app's whole security model assumed would never happen.

This is written as defence in depth rather than one lock, on a single governing
rule:

> **A fully compromised MCP container must not be able to touch the app.**

Everything below is downstream of that sentence. Where there is a choice, this
picks one option and says why.

**The fact that shapes everything:** Claude connects **from Anthropic's cloud,
not from the user's device**. A local/stdio server will not work; a tailnet-only
HTTPS endpoint will not work. It must be publicly reachable over HTTPS.

## 0. Before you start

### When this applies

- A self-hosted app with data worth querying conversationally.
- Read-only access is enough. (Write tools are a different risk conversation.)
- One or a handful of accounts, not a multi-tenant product.

### When it does not

- **If the app is already public with real auth**, you do not need most of this —
  add an MCP endpoint behind your existing auth and skip to §5 and §9.
- **If you only want Claude Code (local) to reach it**, use a stdio server on the
  machine. None of the exposure work below is needed.

### Decide these three first

| Question | Default here | Why |
|---|---|---|
| Separate service or a route in the app? | **Separate** | The app's dependency tree contains image decoders, LLM SDKs and scrapers. None of that should be in the one process facing the internet. |
| Auth | **OAuth 2.1, hardened** | It is what Claude speaks. A static bearer token means one stolen string is permanent access. |
| Tools | **Read-only** | A public endpoint that can write to your health/finance/journal data is a different risk class. Start read-only; add writes deliberately, later, never as a side effect. |

## 1. Architecture

```
Anthropic cloud ──►  app-mcp.<tailnet>.ts.net:443     [PUBLIC — Funnel]
                       /mcp  /.well-known/*  /register  /token  /revoke
you (on tailnet) ──►  app-mcp.<tailnet>.ts.net:8443    [PRIVATE — Serve]
                       /authorize   ← login + consent
                         │  tailscale-mcp sidecar   (network: mcpnet only)
                         ▼
                    mcp container  ── no app framework, no app secrets, no route to the app
                         │
                         ▼
                       db  ── the ONLY thing bridging mcpnet and the app network
                         ▲
                    app container
                         │  tailscale sidecar        (network: default)
you (on tailnet) ──►  app.<tailnet>.ts.net:443        [PRIVATE — unchanged]
```

The app keeps its own sidecar and its own hostname and is **never** Funnel'd.
The connector gets its own of everything. The database is the only shared thing,
and §7.1 makes that bridge read-mostly.

**Why `/authorize` can live on 8443:** Funnel only allows 443, 8443 and 10000,
and Serve and Funnel cannot share a port. So 443-Funnel plus 8443-Serve is
exactly the shape the constraint permits. Claude's cloud never fetches
`/authorize` — it redirects *the user's browser* there, and the user is on the
tailnet.

Ship `/authorize` on 443 first and prove the flow end to end, then flip one env
var to 8443 and re-verify. It is the best hardening available here *and* the
likeliest thing to break the connector for reasons invisible in your logs, which
is why it is a flag and not a hardcoded value.

## 2. Files this creates

```
backend/mcp_server/
  __init__.py          docstring stating the import ban (see §9.3)
  config.py            every guard; fails closed at import
  server.py            Starlette app, routes, middleware, token verifier
  main.py              entry point; reads env, installs the verifier
  tools.py             the tools, as plain functions (db, user_id, …)
  queries.py           the owned() chokepoint — the ONLY place db.query() appears
  oauth.py             DCR, /authorize, /token, /revoke, both metadata documents
  tokens.py            JWT mint/verify
  templates.py         consent + error pages (plain HTML, no framework)
  ratelimit.py         (client_ip, identifier) throttles
backend/requirements-mcp.txt    the deliberate dependency split
backend/requirements-mcp.lock   pip-compile --generate-hashes
backend/scripts/mcp_db_role.sql least-privilege DB role, self-verifying
tailscale/serve-mcp.json        the Funnel'd serve config
```

Modified: `Dockerfile` (a new stage), `docker-compose.yml`, `.env.example`, one
migration, docs.

**Untouched on purpose:** the app's `main.py`, every existing router, the app's
`serve.json`, the app's Dockerfile stage. If your change touches those, the
isolation has leaked — go back and find out why.

## 3. The dependency split

This is the part people skip, and it is the highest-value control in the whole
design.

The MCP SDK needs a recent pydantic and starlette. Your app probably pins older
ones. Do **not** upgrade the app to match — that is a framework upgrade across
every route in an app that likely has no tests, to ship a read-only side feature.

Instead: a second requirements file and a second image stage over **one shared
model layer**.

```
# requirements-mcp.txt — deliberately no web framework from the app side.
mcp==<pin>
uvicorn==<pin>
pydantic==<pin>          # pinned explicitly: the version split is the whole point
sqlalchemy==<same as app>   # lockstep with the app: both map the same tables
psycopg2-binary==<same as app>
bcrypt==<same as app>
pydantic-settings==<pin>
```

The security dividend is the point, not a side effect: **the one container
facing the internet contains no image decoders, no LLM SDK, no scrapers, no
scheduler, and none of the app's routes.** They are not "unrouted" there — they
do not exist.

### 3.1 Copy only the modules the connector imports

Derive the list mechanically rather than copying the whole app package:

```python
# walk ImportFrom nodes transitively from mcp_server/*.py, keep the app.* ones
```

Then in the Dockerfile stage, copy exactly those files. A typical result is six
or seven modules: config, database, models, security, and one or two pure
helpers. **Not** the routers, **not** the app entry point, **not** the
migrations, **not** the built frontend.

### 3.2 Make the build fail if the split ever stops holding

Put this in the stage, so it is a build gate rather than a claim in a comment:

```dockerfile
RUN SECRET_KEY=build-check MCP_TOKEN_SECRET=build-check-0123456789abcdef0123456789 \
    MCP_PUBLIC_ORIGIN=https://build.invalid DEV_MODE=false \
    python -c "import mcp_server.server, mcp_server.tools, mcp_server.oauth" && \
    ! python -c "import fastapi" 2>/dev/null && \
    ! python -c "import app.main"  2>/dev/null && \
    echo "isolation holds"
```

### 3.3 Lock the transitive closure

The packages that parse untrusted bytes on this service's behalf — starlette,
h11, `python-multipart`, jsonschema, anyio — arrive **transitively** and are
unpinned by the file above. For the one image designated internet-facing,
generate a fully pinned, hashed lock and build from it:

```bash
docker run --rm -i python:3.12-slim sh -c '
  pip install -q pip-tools; cat > /tmp/r.txt
  pip-compile -q --generate-hashes --strip-extras -o /tmp/o.lock /tmp/r.txt >/dev/null
  cat /tmp/o.lock' < backend/requirements-mcp.txt > backend/requirements-mcp.lock
```

Then `pip install --require-hashes -r requirements-mcp.lock`.

## 4. The read layer

### 4.1 One chokepoint, and a CI grep that proves it

Every user-data read goes through one function. This is the only thing enforcing
account scoping and soft-delete, and it is worth more than any amount of care at
the call sites.

```python
def owned(db: Session, model: type[T], user_id: int) -> Query[T]:
    # A non-int user_id here is a bug that would otherwise become a data leak,
    # so it raises rather than coercing.
    if type(user_id) is not int or user_id <= 0:
        raise TypeError(f"user_id must be a positive int, got {user_id!r}")
    q = db.query(model).filter(model.user_id == user_id)
    if hasattr(model, "deleted_at"):
        q = q.filter(model.deleted_at.is_(None))
    return q
```

Provide `by_id`, `children_of`, `in_range`, `clamp_limit` as sanctioned escape
hatches so nobody needs a raw query. Then enforce it — `tools.py` must contain
**zero** `db.query(`:

```bash
grep -n 'db\.query(' mcp_server/tools.py && exit 1   # CI
```

(OAuth bookkeeping in `oauth.py` legitimately queries its own tables. Scope the
grep to the tool layer, not the package.)

### 4.2 Tools are plain functions first

Write them as `(db, user_id, ...) -> dict` and exercise them from a shell against
the real database **before** any MCP, OAuth or Docker exists. Most bugs live
here and this is the cheapest place to find them.

### 4.3 Units go in the key, always

`weight_kg`, `distance_km`, `waist_cm`, `protein_g`. Never a bare `weight`
beside a separate `weight_unit` label — the model will mix representations and
sum across them. Emit **one** representation; put the user's display preference
in the profile tool, explicitly labelled as a preference, not a unit.

### 4.4 Shape for questions, not for endpoints

Eight-ish tools organised around what someone asks, not around your REST
routes. A `get_daily_summary(start, end)` that joins log + nutrition + activity
rollup in one call beats three tools the model has to correlate itself.

- Gaps are explicit `null` rows, never omitted — an absent day and a zero day
  mean different things.
- Cap list results and return `truncated: true` plus a suggested narrower range,
  rather than silently cutting.
- Bound the date range (e.g. 180 days) and make sure every default is **inside**
  the bound. A `default_days=365` against a `MAX_DAYS=180` raises on every call
  with no arguments.

### 4.5 Order any query whose result depends on order

A "change since last time" computed from an unordered query is right on SQLite
(index-driven) and wrong on Postgres (heap order), and the sign can invert. Add
the `ORDER BY` even when it looks redundant.

## 5. OAuth 2.1

Verified against Claude's connector requirements, not inferred:

- **DCR (RFC 7591) is supported** — implement `POST /register`.
- The redirect URI to accept is **`https://claude.ai/api/mcp/auth_callback`**.
- **PKCE `S256` must be advertised** in metadata or the client will not send a
  challenge.
- **`offline_access` must appear in `scopes_supported`** or no refresh token is
  requested and the connector silently dies after an hour.
- The protected-resource `resource` **must match the URL the user typed**,
  exactly, including the path.
- Only the **first** entry of `authorization_servers` is used.
- Discovery, registration and token calls have a **10 s** timeout.

Endpoints: `/.well-known/oauth-protected-resource/<path>`,
`/.well-known/oauth-authorization-server`, `POST /register`, `GET`/`POST
/authorize`, `POST /token`, `POST /revoke`, `GET /healthz`.

### 5.1 Hardening beyond the minimum

- **Redirect-URI allowlist on `/register`** — reject anything outside Claude's
  callbacks (plus loopback with the port ignored, per RFC 8252 §7.3, if Claude
  Code should also work). This alone makes a stolen auth code unredeemable.
- **Public clients only** (`token_endpoint_auth_method: "none"`). PKCE plus the
  allowlist is the correct OAuth 2.1 shape and removes a secret you would
  otherwise have to store.
- **Auth codes**: 32 random bytes, **SHA-256 hashed at rest**, 60 s TTL,
  single-use via a conditional `UPDATE ... WHERE consumed_at IS NULL` inside the
  issuing transaction.
- **Access tokens**: JWT, 1 h, `aud` = the canonical resource URI, validated on
  every call (RFC 8707). Add a `gid` claim → one indexed lookup per request,
  which buys **instant revocation** instead of waiting out the hour.
- **Refresh tokens**: hashed, **rotating** (each use kills the old), 30-day
  sliding. A dead one returns `invalid_grant`, not `invalid_request` — clients
  distinguish these.
- **Password epoch**: stamp the token with when the user's password last
  changed, and refuse a token that predates it. Changing your password then kills
  Claude's access for free. Compute it with `calendar.timegm`, not
  `.timestamp()`, if the column is naive UTC — otherwise it shifts by the host's
  offset.
- **`password_hash IS NULL` → refuse.** Never expose an account-claim flow here.
- **Throttle keyed `(client_ip, identifier)`.** If the app's own throttle is
  identifier-only because everything behind a sidecar shares one IP, that
  reasoning **inverts** on a public endpoint. Comment the divergence.
- **DCR row cap** (~50, LRU-evicted). Clients register afresh on every new
  connection.

Every secret hashed at rest. SHA-256, not bcrypt — these are 256-bit random
values, so there is nothing to brute-force and per-request bcrypt is
self-inflicted DoS.

## 6. The MCP endpoint — use the SDK

Hand-rolling the protocol is ~400 lines you will re-read the spec to maintain.
The SDK owns the pedantry: POST-only with 405s, sessionless behaviour, the
header mirroring and its `-32020` mismatch code, `404` + `-32601` for unknown
methods, and JSON-Schema generation from type hints.

Two traps:

- **A mounted sub-app's lifespan never runs.** The host must enter the session
  manager itself or the first request dies with `RuntimeError: Task group is not
  initialized`.
- **`Mount("/")` matches everything**, so every real route must be listed
  *before* it.

Configure transport security explicitly. The default arms DNS-rebinding
protection against localhost only, so a fresh deploy rejects everything with a
bare `421` — which is plain text, not JSON-RPC, so the client shows a generic
transport error and the hostname appears only in your log.

The SDK's auth middleware runs **before** JSON-RPC dispatch, so an
unauthenticated request is refused at the door — nothing parsed, no tool run.

**Wrap every tool invocation.** SQLAlchemy's `DataError.__str__` embeds the full
SELECT and bound parameters; an out-of-range id is enough to dump a table's
column list to the caller. Only your own error type should cross the wire;
everything else is logged server-side as a generic failure.

## 7. Deployment

### 7.1 A dedicated database role — do this first

Without it, every other control guards a door beside an open window.

The Docker network split stops the connector reaching the app over HTTP, but the
database is a deliberate bridge — and with the app's own role it is a
**read-write** bridge. A parsing bug in the internet-facing container then means:

```sql
UPDATE users SET password_hash = '<attacker bcrypt>';   -- takeover
UPDATE users SET password_hash = NULL;                  -- re-arms any claim flow
```

followed by a normal login to the real app.

Grant: `SELECT` on the tables the tools read; `SELECT, INSERT, UPDATE, DELETE`
on the three OAuth tables **only**; `USAGE` on their sequences; and **no write
on the users table at all**.

Three things that bite:

- **Grant every table reached through a relationship**, not just the ones
  imported by name. An ORM `selectinload` to a lookup table fails only at
  runtime, only on the query that touches it.
- **Table-level SELECT on users, not column-level.** The ORM emits every mapped
  column, so a column grant breaks the service the next time the model gains a
  field. The property that matters — no write path to the password — comes from
  withholding INSERT/UPDATE/DELETE, which is not fragile.
- **No `ALTER DEFAULT PRIVILEGES`.** A table added by a future migration should
  be unreadable until someone grants it. Fail closed.

End the script with `has_table_privilege` assertions so it verifies itself, and
**prove it by attacking it**, not by reading the catalogue:

```
UPDATE users SET password_hash='x';   -- must be: permission denied
SELECT count(*) FROM daily_logs;      -- must work
```

### 7.2 Compose: profiles and defaulted variables

Both new services go behind `profiles: ["mcp"]`, and **every new variable must
use `${VAR:-}`, never `${VAR:?}`**. See §9.1 — this is not a style preference,
it is the difference between an optional feature and a broken deploy of the
*app*.

Networks: `default` carries the app and its sidecar; `mcpnet` carries the
connector and its sidecar; the database is the only member of both.

**No `env_file`.** Name every variable the connector needs explicitly. It must
never receive the app's session-signing key, the app's Tailscale auth key, or
any third-party API key.

### 7.3 Tailscale

- A **second auth key**, tagged, so revoking one does not kill both sidecars and
  only a tagged node can ever be Funnel'd.
- `serve-mcp.json` with `AllowFunnel` for the connector.
- The app's `serve.json` must have **no** `AllowFunnel`. Assert it:
  `grep -c AllowFunnel tailscale/serve.json` → `0`.

### 7.4 The tailnet ACL

Tag the connector node, grant it `funnel`, and **explicitly grant it no tailnet
destinations**. A default allow-all rule lets the internet-facing node reach the
app over the tailnet, silently defeating the Docker split.

Write user-side rules by **identity, not device** (`autogroup:member`), so new
devices are covered automatically — and so that tagged nodes, which are not
members, get nothing by default.

## 8. Verification

Run all of it. The checks most likely to be skipped are the ones that matter.

**Protocol:** `GET`/`DELETE /mcp` → 405; mismatched header → 400 + `-32020`;
unknown method → 404 + `-32601`; bad `Origin` → 403; bad `Host` → 421.

**OAuth:** `/register` **rejects** `https://evil.example/cb`;
`code_challenge_method=plain` → error; a code redeemed twice fails the second
time; wrong-`aud` token → 401; refresh rotates and the old one is dead;
`/revoke` kills access **within one request**. **And: change the account
password, then confirm the outstanding token is refused.**

**Data:** cross-check one week against the app's own UI; confirm a soft-deleted
row does not appear; confirm no response contains a bare `weight`/`distance`.

**Blast radius:**

```bash
docker compose exec mcp getent hosts <app-sidecar>   # must FAIL
docker compose exec mcp python -c "import fastapi"   # must FAIL
docker compose exec -T db psql -U <mcp_role> -d <db> \
  -c "UPDATE users SET password_hash='x';"           # must be permission denied
```

**Exposure — the one nobody does.** From a device with Tailscale **off**
(a phone on cellular):

```
https://app-mcp.<tailnet>.ts.net/healthz    → must return 200
https://app.<tailnet>.ts.net/api/version    → must FAIL TO CONNECT
```

If the app answers, stop and fix the ACL before going further. Every other
control assumes this answer.

**Add an HTTP smoke test to the deploy script.** Typically nothing in a small
repo makes an HTTP request — CI and release checks stop at `import app.main`,
and release scripts often override the container command, so the server
invocation is executed by **no gate at all**. Poll the public URL until it
reports the commit just deployed, assert the private port is refused, and print
the exact recovery on failure.

## 9. The traps

Each of these cost real debugging time. They are the reason this file exists.

### 9.1 `${VAR:?}` in compose breaks deploys of the *other* services

Compose interpolates the **entire file** before running anything, so a required
variable on an unused service makes `docker compose up` fail for the app too, on
any machine that has not configured the connector. **`profiles:` does not save
you** — it gates startup, not interpolation.

Use `${VAR:-}` and validate inside the container, where you can fail closed with
a readable reason.

### 9.2 …which means an unset secret produces a healthy-looking broken service

The flip side of 9.1: an empty password yields a syntactically valid connection
string. The container starts, `/healthz` returns 200 because it runs no queries,
and every real request 500s. **Validate every required setting at import time**,
and include the database credential.

Also refuse a connection string naming the **app's** role — reusing it is the
intuitive fix when auth fails, and it silently discards §7.1 entirely.

### 9.3 Adding a Dockerfile stage changes what a bare `docker build` produces

BuildKit builds the **last** stage when none is named. Adding the `mcp` stage at
the end means `docker compose build` hands the **app** container the MCP image —
no routes, no frontend. Pass `--target` explicitly everywhere, including
`target: app` in compose.

### 9.4 Tailscale Serve rewrites `Host` for unix-socket backends

If the connector listens on a unix socket, Serve sets `Host` to the proxy
target's host — always `localhost`. Every hostname allowlist (`TrustedHostMiddleware`,
the SDK's rebinding protection) therefore sees only `localhost` and **is inert**.

Adding `localhost` to the allowlist makes the error go away *and deletes the
check*. Instead, validate **`X-Forwarded-Host`**, which Serve sets to the true
incoming hostname and **overwrites rather than appends**, so a client cannot
forge one. Allow requests with no forwarded header (local testing); they are not
internet-reachable.

### 9.5 Over a unix socket, uvicorn reports no client — so proxy headers are silently dropped

`getpeername()` on `AF_UNIX` returns a string, so uvicorn's `get_remote_addr()`
returns `None`, and `ProxyHeadersMiddleware` evaluates `None in {"127.0.0.1"}` →
False. **Every `X-Forwarded-*` header is ignored, with no error and no log
line.** Measured: `'127.0.0.1'` gives `scheme=http, client=None`; `'*'` gives
`scheme=https` and the real IP.

Use `--forwarded-allow-ips='*'` on the socket path. It is *tighter* than a
numeric list there, because only a process with filesystem access to the socket
can connect at all.

### 9.6 The two discovery documents can disagree by one trailing slash

If the SDK builds `authorization_servers` from a pydantic `AnyHttpUrl`, pydantic
normalises a bare origin by **appending a slash** — so the protected-resource
document advertises `https://host/` while your RFC 8414 document reports
`issuer` as `https://host`. A client validating that those match rejects you; one
that concatenates requests `https://host//.well-known/...`, a 404. Both surface
as a generic connection error with nothing in any log.

Serve your **own** protected-resource document, listed before `Mount("/")`, so
both come from one module and cannot drift.

### 9.7 A bind address cannot close a port behind a userspace Tailscale sidecar

If the app shares a sidecar's network namespace, tailscaled's netstack rewrites
**every** inbound tailnet connection to `127.0.0.1` with the port unchanged and
no allowlist. Binding loopback does not help — loopback is exactly where it
delivers.

The only way to close a port is to have **nothing listening on TCP**: serve on a
unix socket and point Serve at `unix:/path/to.sock`. Then netstack's dial fails
and the peer gets a RST. (Confirm the socket is on a mount shared by both
containers — sharing a *network* namespace does not share a filesystem.)

### 9.8 psql does not substitute `:vars` inside dollar-quoted bodies

The obvious `DO $$ ... :password ... $$` fails with `syntax error at or near
":"`. Use `SELECT format('...%L', :'password') \gexec`.

### 9.9 A blanket `*.sql` in `.gitignore` will eat your role script

Repos that ignore `*.sql` to keep database dumps uncommittable will silently
exclude the operator script too — and it goes missing on the server at exactly
the step your documentation tells the user to run. Add a narrow negation
(`!backend/scripts/*.sql`).

### 9.10 An SPA fallback answers 200, which breaks naive health checks

A probe built by string concatenation can produce a double slash; an SPA
catch-all route serves `index.html` with **200** rather than 404. Any check that
treats "200" as success will pass while testing nothing. Assert on **content**
(the deployed commit), not on the status code, and normalise the URL first.

### 9.11 `--virtual-time-budget` cannot screenshot an IndexedDB app

Headless Chrome's virtual clock fast-forwards timers, but IndexedDB callbacks
never fire, so every screenshot is the loading spinner. Drive CDP and
`sleep` in real time instead.

## 10. Staging

| Stage | Work | Proven by |
|---|---|---|
| **0** | Dependency spike — a venv with the MCP requirement set imports your models *and* the SDK | go/no-go on the split; pins the requirements file |
| **1** | The chokepoint and the tools as **plain functions** | called from a shell against the real DB — no MCP, no OAuth, no Docker |
| **2** | Wire into the SDK server, stub verifier, local port | MCP Inspector + raw curl for every error code |
| **3** | Migration, OAuth endpoints, consent page, real verifier, **rate limits and host allowlist** | the whole dance scripted with curl |
| **4** | Image stage, compose, sidecar, **DB role**, ACL | the blast-radius and off-tailnet checks in §8 |
| **5** | Move `/authorize` to the private port, audit logging | full re-verify of the connect flow |
| **6** | Docs + release | your normal release gates |

**Stages 0–3 need no Claude in the loop**, and most of 4 does not either. Do not
skip 1 — tools written directly against the MCP server are tools you cannot
debug.

## 11. Risks worth stating out loud

1. **The dependency split is the whole design.** Enforce it in CI, do not assume it.
2. **Moving `/authorize` to the private port** is the best hardening and the
   likeliest silent breakage. Flag it; flip it after the flow works.
3. **A wrong `resource` value** surfaces as a generic connection error. Document
   the exact string; log received-vs-expected at WARNING.
4. **Soft-delete leakage is silent and permanent.** The chokepoint plus the CI
   grep is the entire defence.
5. **Whether Funnel forwards a real client IP is unverified.** If it does not,
   every IP-keyed limiter degrades to one global bucket with no error — which is
   why the login limiter must also key on the identifier.
6. **The consent screen is the only place the user sees what they are sharing.**
   Free-text notes *are* included. Say so in plain words on the page, not buried
   in a scope string.
