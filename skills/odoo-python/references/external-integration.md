# External Integration Reference

Use when calling an external system from Odoo — an API, a device, or a payment provider (biometric terminals, payment gateways, customer ERPs). The same integration mistakes recur; these patterns keep the call RPC-safe, resilient, and reviewable.

## Keep external-call logic out of the model

A method that doesn't use `self` and talks to an outside system belongs in a module-level `utils.py`, **not** as a model method — every public model method is RPC-callable and reachable from server actions. Module-level functions aren't.

```python
# models/utils.py — plain functions, not on a model
def fetch_transactions(session, server_url, since):
    ...

# models/zkteco_terminal.py
from odoo.addons.hr_attendance_zkteco.models.utils import fetch_transactions
```

Reference pattern: `enterprise/l10n_be_hr_payroll/models/utils.py`.

## Use a configured `requests.Session`

Build one `Session`, set constant headers and auth on it once, pass it down. Don't pass tokens around and re-attach headers at each call site.

```python
import requests

def make_session(credentials):
    s = requests.Session()
    s.headers["Authorization"] = f"Bearer {_get_jwt_token(credentials)}"
    return s   # constant headers live on the session, not per call

# json= sets Content-Type: application/json for you — don't set it manually.
resp = session.post(url, json=payload, timeout=30)
```

Resolve auth in the call that uses it, not three lines earlier in every caller. If a token is needed for a request, the request (or the session) acquires it.

## Take one URL, not `server_url` + `endpoint`

If a function joins `server_url` and `endpoint` and trusts both, just take the full URL — same effect, fewer trust boundaries and fewer params.

## Follow pagination links — don't just check they exist

If the API returns a `next` link, *use* it to fetch the next page. Checking that it's present and then re-deriving the next URL yourself is a bug waiting to happen.

## Error handling: know the `requests` exception hierarchy

`Timeout` and `ConnectionError` are subclasses of `RequestException`; `raise_for_status()` and `Response.json()` (since requests 2.27) also raise `RequestException` subclasses. So:

- **Don't catch redundant subclasses** alongside `RequestException` — it's dead code.
- **Don't write several `except` branches that all `return None`** — that misclassifies errors for no benefit. Catch `RequestException` once.
- **Log the actual error.** `_logger.warning("zkteco sync failed: %s", e)` — not `_logger.warning("error")`, which tells the support tech nothing.

```python
try:
    resp = session.get(url, timeout=30)
    resp.raise_for_status()
    return resp.json()
except requests.RequestException as e:
    _logger.warning("zkteco fetch failed for %s: %s", url, e)
    return None
```

## No unused parameters

If callers never pass `timeout` and it isn't part of the credentials, don't make it a parameter. A keyword-only API should declare itself keyword-only — or keep the `credentials` dict as one bag instead of splatting it into N params that every call site re-assembles.
