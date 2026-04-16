# ZeroBus — gRPC SDK

## Raw SDK (direct, notebook-style)

```python
from databricks.zerobus import ZerobusSdk, ZerobusSdkAsync, AckCallback

sdk = ZerobusSdk(server_endpoint=ENDPOINT, headers_provider=...)
stream = sdk.create_stream(table_name=TABLE_NAME, ...)

# Single row
offset = stream.ingest_record_offset(record_bytes)

# Batch (list of serialized records)
offset = stream.ingest_records_offset(batch_bytes_list)

# Wait for ack
stream.wait_for_offset(offset)

stream.close()
```

### Async variant

```python
sdk = ZerobusSdkAsync(...)
stream = await sdk.create_stream(...)
offset = await stream.ingest_records_offset(batch)
await stream.wait_for_offset(offset)
await stream.close()
```

### AckCallback (optional)

Receives server-side ack notifications without blocking the send path:

```python
class DemoAckCallback(AckCallback):
    def on_ack(self, offset: int) -> None:
        print(f"ack offset={offset}")
    def on_error(self, error: Exception) -> None:
        print(f"error: {error}")
```

Pass `ack_callback=DemoAckCallback()` to `create_stream`.

---

## zbhelper Wrapper (library / production)

`src/zbhelper/zerobus_ingest.py` — wraps raw SDK for DataFrame ingestion via Protobuf.

```python
from src.zbhelper.zerobus_ingest import IngestConfig, ingest_dataframe, build_qualified_table_name
from src.statschema.model import CanonicalTableSchema, CanonicalColumn

table = CanonicalTableSchema(
    name="orders",
    columns=[
        CanonicalColumn(name="order_id", type="integer"),
        CanonicalColumn(name="amount", type="double"),
    ],
)

config = IngestConfig.from_workspace_client(build_qualified_table_name("orders"))
result = ingest_dataframe(df, table, config)
print(f"Ingested {result.rows_sent} rows in {result.batch_count} batches")
```

`ingest_dataframe` pipeline:
1. `schema_to_proto_str(table)` → `.proto` string
2. `compile_proto(proto_str)` → compiled message class
3. `serialize_row(row, msg_class)` → bytes per row
4. `stream.ingest_records_offset(batch)` per batch (default `batch_size=500`)
5. `stream.flush()` then `stream.close()` in `finally`

`build_qualified_table_name("orders")` → `main.robert_lee.orders` (derives schema from user email).

---

## Testing

### Unit tests (mock SDK)

```python
from unittest.mock import MagicMock, patch

mock_stream = MagicMock()
mock_stream.ingest_records_offset.side_effect = iter([1, 2, 3])
mock_sdk = MagicMock()
mock_sdk.create_stream_with_headers_provider.return_value = mock_stream

with patch.multiple("src.zbhelper.zerobus_ingest",
        ZerobusSdk=MagicMock(return_value=mock_sdk),
        TableProperties=MagicMock(),
        StreamConfigurationOptions=MagicMock()):
    config = IngestConfig.from_workspace_client("main.default.t")
    result = ingest_dataframe(fake_df, table, config)
```

### Integration tests

```bash
export ZEROBUS_TABLE_NAME=main.default.my_table
.venv_3_11/bin/python -m pytest tests/test_zerobus_ingest.py -k integration -v
```

---

## Common Pitfalls

- **`HeadersProvider.__new__() takes 0 positional arguments`** — Pyo3 `__new__` rejects forwarded args. Fix: `def __new__(cls, *a, **kw): return HeadersProvider.__new__(cls)`.
- **Rows ingested but not visible** — call `flush()` before `close()`. `ingest_dataframe` does this in `finally`.
- **Duplicate proto symbol in pytest** — multiple `compile_proto` calls collide. Fix: unique suffix per compilation (`Table_0`, `Table_1`, …).
- **`AttributeError: module has no attribute 'ZerobusSdk'` in mock** — ZeroBus imports must be at module level, not inside functions.
