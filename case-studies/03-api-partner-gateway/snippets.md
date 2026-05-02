# Illustrative Snippets — Partner Sync REST Gateway

> ⚠️ **Intentionally incomplete extracts.** Imports, the full envelope helper, and surrounding logic are omitted. Full source available privately on request.

---

## 1. Route declaration with explicit auth & type

Demonstrates: `auth="public"` + custom decorator pattern for Bearer-token auth.

```python
# Illustrative extract — one endpoint showing the pattern.
class PartnerApi(http.Controller):

    @http.route("/api/v1/partners", type="http", auth="public",
                methods=["GET"], csrf=False, save_session=False)
    @validate_token
    @rate_limit(per_minute=60)
    def list_partners(self, **qs):
        partners = request.env["res.partner"].search(
            self._build_domain(qs), limit=int(qs.get("limit", 50)))
        return self._envelope(data=[p.read(["id", "name", "email"])[0] for p in partners])
```

---

## 2. Token validation decorator

Demonstrates: how the Bearer token is extracted, validated, and bound to `request.env`.

```python
# Illustrative extract — decorator that rebinds env to the token owner.
def validate_token(func):
    def wrapper(self, *args, **kwargs):
        auth = request.httprequest.headers.get("Authorization", "")
        if not auth.startswith("Bearer "):
            return self._envelope(error=("missing_token", "Bearer token required"), status=401)
        token = request.env["api.token"].sudo().validate(auth[7:])
        if not token:
            return self._envelope(error=("invalid_token", "Token invalid or expired"), status=401)
        # Rebind env so downstream ORM runs as the token's owner.
        request.update_env(user=token.user_id)
        return func(self, *args, **kwargs)
    return wrapper
```

---

## 3. Response envelope

Demonstrates: the one-liner that enforces the JSON contract across every endpoint.

```python
# Illustrative extract — unified response shape.
def _envelope(self, data=None, error=None, status=200):
    body = {"success": error is None}
    if error is not None:
        code, message = error if isinstance(error, tuple) else ("error", str(error))
        body["error"] = {"code": code, "message": message}
    else:
        body["data"] = data
    return request.make_response(
        json.dumps(body, default=str),
        headers=[("Content-Type", "application/json")],
        status=status,
    )
```

---

## What's deliberately not shown

- The full `api.token` model (key generation, expiry, revocation, last-used tracking).
- The `rate_limit` sliding-window implementation.
- The `auth/token` endpoint that issues tokens after credential check.
- The complete test suite (token lifecycle, 401/429 paths, CRUD flows).
- The Postman collection and OpenAPI-style docs.

These are in the full module in the private repo.
