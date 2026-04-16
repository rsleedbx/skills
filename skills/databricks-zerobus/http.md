# ZeroBus — HTTP REST

No SDK install required. Uses standard HTTP POST with an OAuth bearer token.

## OAuth Token Fetch

```python
import json, requests
from requests.auth import HTTPBasicAuth

authorization_details = json.dumps([
    {"type": "unity_catalog_privileges", "privileges": ["USE CATALOG"],
     "object_type": "CATALOG", "object_full_path": CATALOG},
    {"type": "unity_catalog_privileges", "privileges": ["USE SCHEMA"],
     "object_type": "SCHEMA", "object_full_path": f"{CATALOG}.{SCHEMA}"},
    {"type": "unity_catalog_privileges", "privileges": ["SELECT", "MODIFY"],
     "object_type": "TABLE", "object_full_path": TABLE_NAME},
])

token_resp = requests.post(
    f"{DATABRICKS_WORKSPACE_URL}/oidc/v1/token",
    auth=HTTPBasicAuth(CLIENT_ID, CLIENT_SECRET),
    data={
        "grant_type": "client_credentials",
        "scope": "all-apis",
        "resource": f"api://databricks/workspaces/{DATABRICKS_WORKSPACE_ID}/zerobusDirectWriteApi",
        "authorization_details": authorization_details,
    },
    timeout=60,
)
token_resp.raise_for_status()
access_token = token_resp.json()["access_token"]
```

## Sync — `requests.Session` (connection reuse)

Always use a `Session` — reuses TCP+TLS connections. Without it, each POST pays ~200ms handshake overhead.

```python
session = requests.Session()

def insert_rows(rows: list) -> requests.Response:
    resp = session.post(
        f"{ZEROBUS_INGEST_URL}/zerobus/v1/tables/{TABLE_NAME}/insert",
        headers={"Content-Type": "application/json",
                 "Authorization": f"Bearer {access_token}"},
        data=json.dumps(rows),
        timeout=120,
    )
    resp.raise_for_status()
    return resp

# Single row
insert_rows([{"device_name": "sensor-0", "temp": 22, "humidity": 55}])

# Batch
insert_rows(batch_records)   # list of row dicts

session.close()
```

## Async — `aiohttp.ClientSession`

Async wins for **concurrent singles** (`asyncio.gather`). For sequential batch, sync is faster (lower overhead).

```python
import asyncio, aiohttp

async def insert_rows_async(session: aiohttp.ClientSession, rows: list) -> float:
    async with session.post(
        f"{ZEROBUS_INGEST_URL}/zerobus/v1/tables/{TABLE_NAME}/insert",
        data=json.dumps(rows),
        headers={"Content-Type": "application/json",
                 "Authorization": f"Bearer {access_token}"},
        timeout=aiohttp.ClientTimeout(total=120),
    ) as resp:
        resp.raise_for_status()

# Concurrent singles — wall time ≈ one RTT instead of N × RTT
async with aiohttp.ClientSession() as session:
    await asyncio.gather(*[insert_rows_async(session, [row]) for row in single_rows])

# Sequential batch — no parallelism benefit; one round-trip regardless
async with aiohttp.ClientSession() as session:
    await insert_rows_async(session, batch_records)
```

## Performance Characteristics

| Pattern | Typical wall time | Notes |
|---|---|---|
| Single row, cold (no session) | ~600–800 ms | TCP+TLS handshake per request |
| Single row, warm session | ~350–450 ms | Reusing connection |
| Async concurrent singles (10 rows, gather) | ~250–400 ms wall | All 10 in parallel |
| Batch (990 rows), warm session | ~200–600 ms | Server commit jitter; run 10× for distribution |

**Connection warmup matters most.** First request after creating `Session`/`ClientSession` pays the handshake. Runs 2+ are consistently faster.

**`asyncio.gather` is the only reason to use async for HTTP.** A `for` loop with `await` inside is sequential and slower than sync `requests` due to coroutine overhead.

## Common Pitfalls

- **Single-use `requests.post()` instead of `Session`** — pays TCP+TLS on every request (~200ms extra each).
- **`await` in a `for` loop thinking it's concurrent** — it's not; use `asyncio.gather` for concurrency.
- **Forgetting `session.close()` / `await session.close()`** — leaks connections; always close in `finally`.
- **`authorization_details` as Python dict instead of JSON string** — must be `json.dumps([...])`, not the list itself.
