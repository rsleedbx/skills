# Byte Metrics — MySQL Source Side

## Static proxy: DATA_LENGTH

```sql
SELECT TABLE_NAME,
  TABLE_ROWS,
  round(DATA_LENGTH/1024/1024, 1) AS data_mb,
  round(DATA_LENGTH/TABLE_ROWS, 1) AS bytes_per_row
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = '<db>'
  AND TABLE_NAME IN ('intpk', 'intpk_10m', 'intpk_100m');
```

`DATA_LENGTH` = InnoDB on-disk size — a proxy, not wire bytes. Wire bytes are uncompressed row data (≈ DATA_LENGTH ÷ TABLE_ROWS × rows_read), which exceed `DATA_LENGTH` because InnoDB compresses on disk but sends raw rows over JDBC.

## Assumptions underlying the byte measurement

Before using `performance_schema.status_by_account` as a wire-byte proxy, verify each assumption. Assumptions marked **[VERIFIED]** were confirmed by a live test; re-verify on new Databricks or MySQL versions.

| # | Assumption | Status | How to verify |
|---|------------|--------|---------------|
| A1 | LFC connects to MySQL without protocol compression — `Bytes_sent` = actual wire bytes | **[VERIFIED Apr 2026]** — `useCompression` is not in UC connection supported options | Probe the UC connection API: send a dummy option and read the "Supported options:" error |
| A2 | `performance_schema` is enabled on the MySQL instance | **[ASSUMED ON]** — default in MySQL 8 | `SHOW VARIABLES LIKE 'performance_schema';` — must return `ON` |
| A3 | `status_by_account` counters are cumulative since last server start and have not been flushed | **[ASSUMED]** | Detect by comparing `Bytes_sent` delta sign (negative = reset occurred) |
| A4 | `Bytes_sent` is measured before TLS encryption — independent of TLS on/off | **[VERIFIED Apr 2026]** — MySQL counts at protocol layer, above TLS | Enable/disable `trustServerCertificate`; `Bytes_sent` delta should be identical |
| A5 | Each serverless cluster run uses a distinct source IP — delta for `lfc_user` equals bytes for exactly this run | **[ASSUMED]** | Compare `COUNT(DISTINCT HOST)` in `status_by_account` before vs after |

**If any assumption is violated**, the `Bytes_sent` delta will be incorrect. The script `databricks-run-with-byte-capture.sh` guards against A3 (warns on negative delta) but cannot guard against A5 automatically.

## Exact wire bytes — pre/post snapshot

**Step 1 — pre-run snapshot**

```sql
SELECT USER, HOST, VARIABLE_NAME, CAST(VARIABLE_VALUE AS UNSIGNED) AS bytes
FROM performance_schema.status_by_account
WHERE USER IN ('lfc_user_public', 'lfc_user_png')
  AND VARIABLE_NAME IN ('Bytes_sent', 'Bytes_received')
ORDER BY USER, VARIABLE_NAME;
```

**Step 2 — trigger the run**

```bash
databricks -p $PROFILE pipelines start-update --pipeline-id <id>
watch -n 5 'databricks -p $PROFILE api get \
  "/api/2.0/pipelines/<id>/updates?max_results=1" \
  | jq ".updates[0] | {state, update_id}"'
```

**Step 3 — post-run snapshot and delta**

```sql
WITH pre AS (
  SELECT USER, VARIABLE_NAME, CAST(VARIABLE_VALUE AS UNSIGNED) AS v
  FROM performance_schema.status_by_account
  WHERE USER IN ('lfc_user_public', 'lfc_user_png')
    AND VARIABLE_NAME IN ('Bytes_sent', 'Bytes_received')
),
post AS (
  SELECT USER, VARIABLE_NAME, CAST(VARIABLE_VALUE AS UNSIGNED) AS v
  FROM performance_schema.status_by_account
  WHERE USER IN ('lfc_user_public', 'lfc_user_png')
    AND VARIABLE_NAME IN ('Bytes_sent', 'Bytes_received')
)
SELECT post.USER, post.VARIABLE_NAME,
  post.v - pre.v                          AS bytes_delta,
  ROUND((post.v - pre.v) / 1048576.0, 2) AS mb_delta
FROM post JOIN pre USING (USER, VARIABLE_NAME)
ORDER BY post.USER, post.VARIABLE_NAME;
```

`Bytes_sent` delta = exact JDBC read volume (server → LFC client).
`Bytes_received` = query text only (a few KB). Focus on `Bytes_sent`.

**Caveats:**
- If the server restarts between pre and post, the delta is invalid — detect with `SHOW GLOBAL STATUS LIKE 'Uptime'`.
- Any other operation by the same user between snapshots inflates the delta. With dedicated `lfc_user_public` / `lfc_user_png` users, this is negligible.

## Fallback: status_by_thread (shared user)

If both pipelines use the same `lfc_user`, capture the thread ID during the run:

```sql
SELECT t.THREAD_ID, t.PROCESSLIST_ID, t.PROCESSLIST_HOST,
       s_sent.VARIABLE_VALUE  AS bytes_sent_so_far,
       s_recv.VARIABLE_VALUE  AS bytes_recv_so_far
FROM performance_schema.threads t
JOIN performance_schema.status_by_thread s_sent
  ON t.THREAD_ID = s_sent.THREAD_ID AND s_sent.VARIABLE_NAME = 'Bytes_sent'
JOIN performance_schema.status_by_thread s_recv
  ON t.THREAD_ID = s_recv.THREAD_ID AND s_recv.VARIABLE_NAME = 'Bytes_received'
WHERE t.PROCESSLIST_USER = 'lfc_user'
  AND t.PROCESSLIST_COMMAND != 'Sleep'
ORDER BY CAST(s_sent.VARIABLE_VALUE AS UNSIGNED) DESC;
```

Thread counters reset when the connection closes — must be sampled while the run is live.

## Separate users vs shared user

| Concern | Separate users | Shared user |
|---------|---------------|-------------|
| Concurrent P1 + P2 runs | Clean — account-level delta per user | Ambiguous — threads interleave |
| Timing requirement | None — snapshot before/after | Must be sampled during the run |
| Counter reset risk | Only on server restart | Same, plus thread close resets to 0 |
| Setup cost | Two extra GRANT statements | None |
| Recommendation | **Preferred** | Use only if user consolidation is required |
