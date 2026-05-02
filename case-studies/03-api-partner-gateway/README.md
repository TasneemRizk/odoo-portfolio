# Case Study 03 — Partner Sync REST Gateway

> **Stack:** Odoo 18 · Python · `http.Controller` · JSON
> **Fictional client:** _FlowBridge Logistics_ — a fictional 3PL that needs to expose a subset of Odoo partners to an external dispatch system.
> **Type:** Backend controller from scratch + token auth model

---

## The Problem

FlowBridge's dispatch software is a Node.js application that assigns shipments to carriers. It needs to **pull and push partner data** (carriers, drop-off contacts) from the Odoo backend — in **real time, over HTTPS, with proper authentication**.

Odoo's built-in JSON-RPC endpoint technically works for this, but it has three operational problems for external consumers:

1. **Session cookies, not tokens.** Non-browser clients end up doing login handshakes on every restart and storing passwords — a security smell.
2. **No API versioning.** `web/dataset/call_kw` is Odoo's internal RPC; any refactor of the method signature silently breaks clients.
3. **No rate limiting.** A misconfigured cron on the client side can hammer Odoo with thousands of requests per second.

The deliverable: a **dedicated `/api/v1/*` REST surface** with token authentication, explicit response envelopes, and in-process rate limiting, designed to be consumed by external services independently of the Odoo web UI.

---

## Constraints & Considerations

- **No external dependencies.** Keep the module pip-free — use only `werkzeug` (already an Odoo dep) and the Python stdlib.
- **Tokens must be auditable.** Every token is a record; rotation, revocation, and last-used tracking all happen through the ORM.
- **Errors must be structured.** Every 4xx/5xx returns a JSON envelope with a stable `error.code` — not raw HTML from werkzeug.
- **Stateless controllers.** No reliance on session cookies; every request carries its own auth. This matches how modern API consumers (curl, Postman, SDKs) work.
- **Rate limit in-process.** A real deployment would use Redis; here, an in-memory sliding window demonstrates the pattern without adding a dependency.

---

## API Surface

```
POST   /api/v1/auth/token           Issue a token for (login, password).
GET    /api/v1/partners             List partners (paginated, filterable).
POST   /api/v1/partners             Create a partner.
GET    /api/v1/partners/<id>        Retrieve one partner.
```

All endpoints (except `auth/token`) require:

```
Authorization: Bearer <token>
```

Response envelope:

```json
{ "success": true,  "data": { ... }                        }
{ "success": false, "error": { "code": "invalid_token", "message": "…" } }
```

Full OpenAPI-style request/response contracts live in the private repo's `docs/api.md`.

---

## Architecture

See [`architecture.md`](./architecture.md) for the full diagram.

At a glance:

```
Client  ──HTTPS──►  /api/v1/partners
                        │
                        ▼
          @validate_token  (decorator)
                        │
                        ▼
          @rate_limit     (sliding window)
                        │
                        ▼
               Controller method
                        │
                        ▼
          env['res.partner'] ORM ops
                        │
                        ▼
               JSON envelope out
```

---

## Key Technical Decisions

- **`http.Controller` with `type="json"` where possible, `type="http"` where JSON parsing must be explicit.** Keeps Odoo's serialization helpful where it fits and gets out of the way where it doesn't.
- **`auth="public"` + custom decorator.** Odoo's built-in `auth="user"` assumes session cookies; we set `auth="public"` and enforce the Bearer token in our own `@validate_token` decorator that does `self.env = self.env(user=token.user_id)` to sudo into the right identity.
- **Tokens are a first-class model.** `api.token` has `user_id`, `expires_at`, `last_used_at`, `is_active`. Revocation = write.
- **Response envelope is a single helper.** `_envelope(data=None, error=None, status=200)` — one place to enforce the contract. No ad-hoc JSON dicts scattered across endpoints.
- **Rate limiter lives in `api.token.check_rate_limit(ip, endpoint)`** — portable to Redis later by swapping the storage class.

---

## Illustrative Snippets

See [`snippets.md`](./snippets.md) for the route + decorator + envelope shape.

---

## Screenshots & Demo

| Screenshot | What it shows |
|---|---|
| ![Tokens](./screenshots/01-api-tokens.png) | Backend list of issued tokens with expiry and usage stats |
| ![Postman](./screenshots/02-postman-call.png) | Postman call to `GET /api/v1/partners` with Bearer token |
| ![Error](./screenshots/03-error-envelope.png) | Structured 401 response when the token is expired |

🎬 [`demo.gif`](./demo.gif) — 20 sec: issue a token in the Odoo UI → curl the endpoint → receive the JSON envelope → revoke the token → curl again → 401.

---

## What I Learned

> Writing controllers from scratch made me appreciate how much **Odoo's default `auth="user"` hides**. The moment you step outside the web UI, you need to think about token lifecycle, request body parsing, CORS, and error envelopes — none of which Odoo gives you for free.

Two takeaways:

1. **Always write a decorator before the second endpoint.** The first endpoint tempts you to inline the auth check; the second endpoint forces you to realize you need a decorator. Starting with the decorator saves you the rewrite.
2. **A structured error envelope is the most valuable thing you can give a client integrator.** `{"error": {"code": "...", "message": "..."}}` with stable codes lets the client build a proper error-handling matrix. Returning raw werkzeug HTML is a hostile API.

---

## Want the Full Code?

Complete module — including the full test suite, Postman collection, and OpenAPI-style docs — is in the private companion repo. **Available on request for interview context.**
