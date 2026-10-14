# REST API and HTTP Troubleshooting
# Methods, status codes, auth, curl, gateway vs downstream

Use this when a client, gateway, or microservice call fails.
Always capture: method, full URL, status, response body, correlation id, and time.

---

## 1. Methods

| Method | Use | Safe to retry? |
|---|---|---|
| GET | Read | Yes |
| POST | Create or action | Often no. A retry can create a second order |
| PUT | Full replace | Yes, if the server replaces the same resource |
| PATCH | Partial update | Usually yes if the patch is idempotent |
| DELETE | Remove | Yes if deleting twice returns 404 or 204 |

Idempotent means repeating the same call does not change the outcome again. GET, PUT, and DELETE usually are. POST often is not. A timeout on POST is the dangerous case: the server may have committed while the client saw a failure.

---

## 2. Status codes to memorize

| Code | Meaning | Your next step |
|---|---|---|
| 200 | OK | Read the body. Business failure can still be inside a 200 |
| 201 | Created | Note the new id in the body or Location header |
| 202 | Accepted | Work is async. Check the job or topic, not only this call |
| 204 | No content | Success with an empty body |
| 400 | Bad request | Payload, field validation, or business rule. Do not retry unchanged |
| 401 | Not authenticated | Missing, expired, or bad token |
| 403 | Authenticated but not allowed | Role, scope, or ACL |
| 404 | Not found | Wrong URL, id, or resource already removed |
| 409 | Conflict | Duplicate, version clash, state does not allow the action |
| 422 | Unprocessable | Body parsed, business validation failed |
| 429 | Too many requests | Back off. Check rate limit headers |
| 500 | Server error | Bug or a wrapped downstream error. Read the inner code |
| 502 | Bad gateway | Proxy could not get a valid response from upstream |
| 503 | Unavailable | No ready endpoints, overload, or maintenance |
| 504 | Gateway timeout | Upstream too slow. Check the downstream service, not only the gateway |

401 vs 403: 401 means "who are you?". 403 means "I know you, and you cannot do this".

A gateway 500 that contains `CPG-SUBS-04000021` and `"status":"400"` is a downstream 400 wrapped as 500. Fix the payload or the stored data. Retrying the same body repeats the 400.

---

## 3. Auth

| Scheme | What you send | Failure |
|---|---|---|
| API key | Header such as `x-api-key` | 401 if missing or revoked |
| Basic | `Authorization: Basic base64(user:password)` | 401 |
| Bearer JWT | `Authorization: Bearer <token>` | 401 if expired, 403 if scope is wrong |
| OAuth2 client credentials | Token from the token URL, then Bearer | 401 on the token call means client id/secret; 401 on the API means the token |
| mTLS | Client certificate on the TLS handshake | Handshake failure before HTTP status |

JWT parts are `header.payload.signature`. You can decode the payload (it is not encrypted) and read `exp`, `iss`, `aud`, `scope`. Do not paste live tokens into tickets or chat.

```bash
# show expiry only; do not print the token
python3 - <<'PY'
import os, json, base64, datetime
tok = os.environ["TOKEN"].split(".")[1]
tok += "=" * (-len(tok) % 4)
payload = json.loads(base64.urlsafe_b64decode(tok))
exp = payload.get("exp")
print("exp", exp, datetime.datetime.utcfromtimestamp(exp) if exp else None)
print("iss", payload.get("iss"), "aud", payload.get("aud"))
PY
```

Clock skew: a token can be rejected if the server clock is ahead of `nbf` or behind `exp`. Compare `date -u` on the caller and the server.

---

## 4. curl — copy and adapt

```bash
curl -sk -D - -o /tmp/body.json -w '\nhttp_code=%{http_code} time_total=%{time_total} time_connect=%{time_connect} time_starttransfer=%{time_starttransfer}\n' \
  -X POST 'https://<host>/chargingProvisioningGateway/v1/profile/createSubscriber' \
  -H 'Authorization: Bearer <token>' \
  -H 'Content-Type: application/json' \
  -H 'X-Correlation-Id: manual-test-001' \
  --data @payload.json
```

- `-D -` prints response headers.
- `-w` prints timings. `time_connect` high = network. `time_starttransfer` high after a fast connect = server or downstream.
- `-k` skips TLS verify. Use it only to separate "bad cert" from "bad HTTP". Then test without `-k`.
- `--max-time 30` stops a hung call.

GET:

```bash
curl -sS -D - -o /tmp/body.json -w '\nhttp_code=%{http_code}\n' \
  -H 'Authorization: Bearer <token>' \
  'https://<host>/path'
```

Verbose TLS and headers:

```bash
curl -sv -o /dev/null --max-time 20 'https://<host>/health'
```

---

## 5. How to debug one failed call

1. Method and URL. A trailing slash or wrong version (`/v1` vs `/v2`) causes 404.
2. Request body. Compare it to a known good call for the same activity.
3. Response status and body. Copy the inner `code` and `reason`.
4. Correlation id. Search gateway, app, and DB logs for that id.
5. Timing. Timeout vs fast 400 are different problems.
6. Downstream URL in the error. If it contains `svc.cluster.local`, the next hop is inside Kubernetes. Check that service's pods and logs at the same timestamp.

Layer split:

| You see | Layer |
|---|---|
| Fast 400 with a business code | Application validation or stored data |
| 401/403 | Auth |
| 404 | Routing or id |
| 504 and threads in `socketRead0` | Downstream slow or dead |
| 503 and Kubernetes endpoints empty | No Ready pod |
| Connection refused | Nothing listening, or wrong port |
| Timeout with no HTTP status | Network, firewall, or TLS hang |

---

## 6. Performance

- Slow DB query behind the API. Take the SQL from the service log and run the plan.
- N+1 calls: one request triggers hundreds of downstream calls. The access log shows a burst for one correlation id.
- Large payload: `RECORD` size or gateway body limit. Check request bytes.
- Pool exhaustion: all worker threads blocked borrowing a JDBC or HTTP connection. New calls time out in the pool, not in business code.
- Missing timeout: one hung downstream call occupies a thread until the client gives up. Set connect and read timeouts.

What to record: p50/p95 latency, error rate, downstream status mix, pool active vs max.

---

## 7. Safe retry rules

Retry GET, and retry POST only when the API documents an idempotency key or you have proved the first call did not commit.

On 400, change the payload or the data. On 429, wait. On 503/504, check the dependency before retrying in a loop. A retry storm is a second incident.
