# Trenchwatch API · v1 (api_version 2026-10)

Trenchwatch tells your product, in milliseconds, whether a new Solana token or address belongs to a known serial operator, and pushes an alert to your systems seconds after a coordinated launch. This guide covers everything you need to integrate: authentication, the sandbox, the four REST endpoints and the signed webhook.

> Alerts are probabilistic observations based on public on-chain data. They describe addresses and patterns, not people, and are not financial advice. Using alerts to trade alongside the addresses they describe is not permitted (Licence Agreement, clause 1.3).

---

## 1. Environments

| Environment | Base URL | Keys | Data |
| --- | --- | --- | --- |
| Sandbox | `https://sandbox.api.[DOMAIN]/v1` | `tw_test_…` | Synthetic, fixed, flagged `"sandbox": true`. Addresses do not exist on-chain |
| Production | `https://api.[DOMAIN]/v1` | `tw_live_…` | Live detections |

A test key never works in production and a live key never works in the sandbox (`401`). Start in the sandbox: it returns the same shapes as production, and `GET /v1/health` lists example addresses for every case.

## 2. Authentication and limits

Send your key in the `X-API-Key` header on every request except `/v1/health`.

```bash
curl -H "X-API-Key: tw_test_XXXXXXXX" https://sandbox.api.[DOMAIN]/v1/token/<mint>
```

- Keys are shown once when issued; we store only a hash. If a key leaks, ask for a new one and the old one is revoked.
- Rate limit: 600 requests per minute per key (higher on request). Over the limit: `429`.
- Every response carries `X-Trenchwatch-Api-Version: 2026-10` and a `took_ms` field.

## 3. Endpoints

### `GET /v1/health`

No key needed. Service status. In the sandbox it also returns example addresses.

```json
{"ok": true, "mode": "sandbox", "api_version": "2026-10",
 "examples": {"tokens": {"HIGH": "5WFk…NqNi", "ELEVATED": "Cwy…oq1f", "LOW": "5jsh…eP1p"},
              "operators": {"TW-9001": "HPGg…sJj8", "TW-9002": "Ubh…QUV"},
              "coordinated_wallet": "5JYE…o21p"}}
```

### `GET /v1/token/{mint}`

Risk of a token, asked before a swap, a listing or a promotion.

```json
{
  "mint": "5WFkboKPeci8vJbCBVicpqgDvFC7j8wSQ2vFxQuzNqNi",
  "risk": "HIGH",
  "created_at": 1791770343,
  "creator": "HPGgF4zjejv3ogpgXnsqM454RHgtCHuHTSoNvauZsJj8",
  "creator_operator": {"id": "TW-9001", "level": "confirmed", "first_seen": "2026-10-01 00:00:00",
                       "last_seen": "2026-10-12 02:09:03", "launches_per_hour": 12.0},
  "linked_operators_buying": [],
  "signals": ["early buyers from this creator's known operator cluster", "sells within the first seconds", "serial creator"],
  "early_buyers": 9,
  "early_sol": 4.2,
  "evidence_note": "Probabilistic, based on public on-chain data. Addresses and patterns, not people. Not financial advice.",
  "took_ms": 0.3
}
```

| Field | Meaning |
| --- | --- |
| `risk` | `HIGH`, `ELEVATED`, `LOW`, or `UNKNOWN` (token not seen by Trenchwatch) |
| `creator_operator` | The creator's entry in the operator registry, or `null` |
| `linked_operators_buying` | Other known operators whose wallets bought early |
| `signals` | Plain-language reasons, for display or logs |
| `early_buyers`, `early_sol` | Distinct buyers and SOL in the first seconds after creation |

Suggested use: `HIGH` → show a warning before the swap or hold the listing; `ELEVATED` → label the token; `LOW` / `UNKNOWN` → no action.

### `GET /v1/operator/{address}`

Is this address a known operator, or one of its coordinated wallets?

```json
{"address": "5JYE9C44QkhczUApEt4YJvoERXS37e8FMdT8nBsLo21p", "role": "coordinated_wallet",
 "operator": {"id": "TW-9001", "level": "confirmed", "first_seen": "2026-10-01 00:00:00",
              "last_seen": "2026-10-12 02:09:03", "launches_per_hour": 12.0}}
```

`role` is `operator`, `coordinated_wallet` or `unknown`.

### `GET /v1/alerts?since=<unix>&limit=<n>`

The latest alerts, newest first, for teams that prefer polling to a webhook. `since` defaults to one hour ago; `limit` defaults to 100, maximum 1000.

```json
{"alerts": [{"ts": 1791770940, "source": "watcher", "type": "operator_cluster_buy", "level": "COORDINATED",
             "token": "5WFkboKPeci8vJbCBVicpqgDvFC7j8wSQ2vFxQuzNqNi",
             "operator": {"id": "TW-9001", "level": "confirmed", "launches_per_hour": 12.0}}]}
```

## 4. Operators, levels and alert types

**Operator IDs** (`TW-0001`…) are stable: the same operator keeps its ID across wallets and launchpads. Operator `level`: `confirmed`, `probable`, `serial` (many launches, no coordinated wallets found yet), `repeat`.

**Evidence levels** are categories, not scores:

| Level | Meaning |
| --- | --- |
| `RELATED` | A transactional relationship was observed. On its own it does not prove common control |
| `COORDINATED` | Coordinated technical or timing behaviour between linked wallets |
| `SUSPECTED` | Several independent signals point to artificial demand |
| `CONFIRMED` | Assigned only after external verification, never automatically |

**Alert types** (stable codes; new ones may be added, so ignore codes you do not know):

| `type` | What happened | Typical action |
| --- | --- | --- |
| `operator_cluster_buy` | Several wallets of one known operator bought a token in its first seconds | Warn users before they buy; hold promotion |
| `coordinated_support` | Wallets of an operator's cluster bought its new token | Warn users; review the listing |
| `operator_launch` | A known operator created a token | Label the token |
| `linked_wallet_buy` | A wallet linked to an operator bought | Context for other alerts |
| `funding_batch` | An operator funded a set of fresh wallets at once | Add them to your watchlist |
| `linked_wallet` | A new wallet was linked to an operator | Treat it as operator-linked |
| `profit_sweep` | Proceeds consolidated to the operator's treasury | Review its recent launches |
| `exit_route` | Proceeds converged on a common destination | Context |

## 5. Webhook

With an annual licence we push every alert to an HTTPS endpoint you choose. Your endpoint must answer `2xx` within 5 seconds; do the heavy work afterwards.

**Request**

```
POST /your/endpoint
Content-Type: application/json
User-Agent: Trenchwatch-Webhook/1
X-Trenchwatch-Timestamp: 1791770943
X-Trenchwatch-Signature: 6f1c…(hex)
```

```json
{
  "api_version": "2026-10",
  "id": "A-20261012-000145",
  "type": "operator_cluster_buy",
  "level": "COORDINATED",
  "detected_at_utc": "2026-10-12 02:00:00",
  "token": "<mint>",
  "operator": {"id": "TW-0145", "address": "<operator address>"},
  "summary": "5 wallets linked to tracked operator TW-0145 bought «Sealcoin» shortly after launch. The operator created this token itself: the early demand is likely artificial.",
  "evidence": [{"step": "buy", "wallet": "<address>", "s after launch": 2, "transaction": "<signature>"}],
  "transaction": "<signature>",
  "sandbox": false
}
```

**Verify every request.** The signature is the hex HMAC-SHA256 of `"<timestamp>.<raw body>"` with your webhook secret. Reject requests whose timestamp is more than 5 minutes old, and compare in constant time.

Python:

```python
import hmac, hashlib, time

def is_valid(secret: str, timestamp: str, raw_body: bytes, signature: str) -> bool:
    if abs(time.time() - int(timestamp)) > 300:
        return False
    expected = hmac.new(secret.encode(), timestamp.encode() + b"." + raw_body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, signature)
```

Node.js:

```js
const crypto = require("crypto");

function isValid(secret, timestamp, rawBody, signature) {
  if (Math.abs(Date.now() / 1000 - Number(timestamp)) > 300) return false;
  const expected = crypto.createHmac("sha256", secret).update(`${timestamp}.${rawBody}`).digest("hex");
  return expected.length === signature.length &&
         crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(signature));
}
```

**Retries and duplicates.** If your endpoint does not answer `2xx`, we retry 3 times over about 10 minutes (after 30 s, 2 min and 7.5 min). A retried alert keeps the same `id`: use it to ignore duplicates.

**Test event.** Ask us for a test event at any time: it arrives signed with your secret, `"sandbox": true` and `"type": "operator_cluster_buy"`.

## 6. Errors

| Status | Meaning |
| --- | --- |
| `401` | Missing or invalid key, or a test key in production (or the reverse) |
| `404` | Unknown path |
| `429` | Rate limit exceeded |
| `500` | Internal error; retry with backoff and tell support if it persists |

Errors return `{"error": "<message>"}`.

## 7. Versioning and changes

- The path version (`/v1`) changes only for breaking changes, announced at least 90 days ahead.
- Within v1 we may add fields, alert types and endpoints. Your integration should ignore what it does not recognise.
- Every change is listed below with its date.

| Date | Change |
| --- | --- |
| 2026-10-12 | v1 published: token, operator, alerts, health; signed webhook with timestamp; sandbox |

Support: [emerge.trenchwatch@gmail.com](mailto:emerge.trenchwatch@gmail.com)
