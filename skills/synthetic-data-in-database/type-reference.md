# Synthetic Data Type Reference

Complete per-engine mappings extracted from `cast_x_to_int.py` and `cast_int_to_x.py`.

`x` = column name (Type → Int) or `a.value` integer seed (Int → Type).
`N` = column_size. `p` = precision. `s` = scale. `d` = decimal_digits.

---

## MySQL

| Category | Type | Type → Integer | Integer → Type |
|----------|------|----------------|----------------|
| Integer | TINYINT | `x` | `mod(x, 256)` |
| Integer | SMALLINT | `x` | `mod(x, 32768)` |
| Integer | MEDIUMINT | `x` | `mod(x, 8388608)` |
| Integer | INT | `x` | `mod(x, 2147483648)` |
| Integer | BIGINT | `x` | `x` |
| Boolean | BIT | `cast(x as signed)` | `mod(x, 2)` |
| Float | FLOAT / REAL | `cast(x as signed)` | `(x * 1.0)` ¹ |
| Float | DOUBLE | `cast(x as signed)` | `(x * 1.0)` ¹ |
| String | CHAR / VARCHAR(N) | `cast(x as signed)` | `RIGHT(CONCAT(repeat('0',N), x), N)` |
| String | TEXT / LONGTEXT / MEDIUMTEXT / TINYTEXT | `cast(x as signed)` | `cast(RIGHT(CONCAT(repeat('0',N), x), N) as TEXT)` |
| Numeric | DECIMAL(p,s) / NUMERIC(p,s) | `cast(x as signed)` | `cast(mod(x, 10^p) / 10^s as DECIMAL(p,s))` |
| Numeric | MONEY equiv | `cast(x as signed)` | `mod(x, max_val) / 10^d` |
| UUID | UUID (BINARY(16) or CHAR(36)) | `cast(conv(x, 16, 10) as signed)` | `CAST(HEX(x) AS BINARY(32))` |
| Binary | BINARY(N) / VARBINARY(N) | `cast(conv(x, 16, 10) as signed)` | `right(concat(repeat('0',N), hex(x)), N)` |
| Binary | BLOB / TINYBLOB / MEDIUMBLOB / LONGBLOB | `cast(conv(x, 16, 10) as signed)` | `right(concat(repeat('0',N), hex(x)), N)` |
| Date/Time | DATETIME / TIMESTAMP | `TIME_TO_SEC(timediff(x,'1970-01-01 00:00:00'))` | `date_add('1970-01-01T00:00:00', interval x second)` |
| Date/Time | DATETIME WITH TZ | N/A | `date_add('1970-01-01T00:00:00+02:00', interval x second)` |
| Date/Time | YEAR | N/A | `FLOOR(RAND()*(2155-1901+1))+1901` ² |
| JSON | JSON | N/A | `JSON_OBJECT('value', x)` |

¹ MySQL 5.7 does not support `CAST AS FLOAT`; `(x * 1.0)` produces DOUBLE.  
² YEAR is not derived from `x`; uses RAND() for the 1901–2155 range.

---

## PostgreSQL

| Category | Type | Type → Integer | Integer → Type |
|----------|------|----------------|----------------|
| Integer | SMALLINT / INT2 | `x` | `cast(mod(cast(x as bigint), 32768) as smallint)` |
| Integer | INT / INT4 | `x` | `cast(mod(cast(x as bigint), 2147483648) as int)` |
| Integer | BIGINT / INT8 | `x` | `cast(x as bigint)` |
| Boolean | BIT / BOOL / BOOLEAN | `cast(x as bigint)` | `cast(mod(cast(x as int), 2) as boolean)` |
| Float | FLOAT / REAL / FLOAT4 | `cast(x as bigint)` | `cast(x as FLOAT)` |
| Float | DOUBLE PRECISION / FLOAT8 | `cast(x as bigint)` | `cast(x as DOUBLE PRECISION)` |
| String | CHAR / BPCHAR / VARCHAR / TEXT | `cast(x as bigint)` | `RIGHT(CONCAT(encode(gen_random_bytes(ceil(N/2)), 'hex'), x), N)` ³ |
| String | NCHAR / NVARCHAR / LONGVARCHAR / LONGNVARCHAR | `cast(x as bigint)` | same as VARCHAR |
| Numeric | DECIMAL(p,s) / NUMERIC(p,s) | `cast(x as bigint)` | `cast(mod(cast(x as bigint), 10^p) / 10^s as DECIMAL(p,s))` |
| Numeric | MONEY | `cast(x as bigint)` | `cast(mod(cast(x as bigint), max_val) / 10^d as MONEY)` |
| UUID | UUID | `('x'\|\|right(replace(cast(x as varchar),'-',''),16))::bit(64)::bigint` ⁴ | `CAST(LPAD(TO_HEX(cast(x as bigint)), 32, '0') AS UUID)` |
| Binary | BYTEA | `('x'\|\|right(lpad(right(cast(value as varchar),-2),16,'0'),16))::bit(64)::bigint` ⁵ | `gen_random_bytes(N)::bytea` (chunked for >1 KB) |
| Date/Time | DATE | `cast(extract(epoch from x) as bigint)` | `date '1970-01-01 00:00:00' + (x) * interval '1 second'` |
| Date/Time | TIME | `cast(extract(epoch from x) as bigint)` | `time '1970-01-01 00:00:00' + (x) * interval '1 second'` |
| Date/Time | TIMESTAMP | `cast(extract(epoch from x) as bigint)` | `timestamp '1970-01-01 00:00:00' + (x) * interval '1 second'` |
| Date/Time | TIMESTAMP WITH TIME ZONE | `cast(extract(epoch from x) as bigint)` | `timestamp with time zone '1970-01-01 00:00:00+02' + (x) * interval '1 second'` |
| XML | XML | N/A | `(SELECT xmlelement(name "root", xmlagg(xmlelement(name "item", encode(gen_random_bytes(16),'hex')))) FROM generate_series(1,N) x)` |

³ For large columns (>2 KB), uses `string_agg(encode(gen_random_bytes(1000),'hex'),'')` over `generate_series`.  
⁴ Takes rightmost 16 hex chars of UUID to fit in bigint; may lose upper bits.  
⁵ Strips `\x` prefix (`right(cast(v as varchar),-2)`), left-pads to 16 hex chars, interprets as bigint.

---

## SQL Server

| Category | Type | Type → Integer | Integer → Type |
|----------|------|----------------|----------------|
| Integer | TINYINT | `x` | `cast(cast(x as bigint) % 256 as tinyint)` |
| Integer | SMALLINT | `x` | `cast(cast(x as bigint) % 32768 as smallint)` |
| Integer | INT | `x` | `cast(cast(x as bigint) % 2147483648 as int)` |
| Integer | BIGINT | `x` | `cast(x as bigint)` |
| Boolean | BIT | `cast(x as bigint)` | `cast(cast(x as bigint) % 2 as bit)` |
| Float | FLOAT / REAL / FLOAT4 | `cast(x as bigint)` | `cast(x as FLOAT)` |
| Float | REAL (double) | `cast(x as bigint)` | `cast(x as REAL)` |
| String | CHAR / NCHAR / VARCHAR / NVARCHAR(N) | `cast(x as bigint)` | `cast(RIGHT(CONCAT(replicate(cast('0' as varchar(max)),N), x), N) as VARCHAR(N))` |
| String | TEXT / NTEXT | `cast(x as bigint)` | `cast(RIGHT(CONCAT(replicate(cast('0' as varchar(max)),N), x), N) as TEXT)` |
| String | SYSNAME | `cast(x as bigint)` | same as VARCHAR |
| Numeric | DECIMAL(p,s) / NUMERIC(p,s) | `cast(x as bigint)` | `cast((cast(x as bigint) % 10^p) / 10^s as DECIMAL(p,s))` |
| Numeric | MONEY | `cast(x as bigint)` | `cast((cast(x as bigint) % max_val) / 10^d as MONEY)` |
| Numeric | SMALLMONEY | `cast(x as bigint)` | `cast((cast(x as bigint) % max_val) / 10^d as SMALLMONEY)` |
| UUID | UNIQUEIDENTIFIER | `convert(bigint, convert(varbinary, x, 2))` | `convert(uniqueidentifier, convert(binary, FORMAT(cast(x as bigint),'X32'), 2))` |
| Binary | BINARY(N) / VARBINARY(N) | `convert(bigint, x, 2)` | `convert(VARBINARY(N), right(CONCAT(replicate('0',2N), convert(varchar(16),convert(varbinary(8),cast(x as bigint)),2)), 2N), 2)` |
| Binary | IMAGE | `convert(bigint, x, 2)` | cast varbinary expr to IMAGE |
| Date/Time | DATETIME / DATETIME2 / SMALLDATETIME | `cast(DATEDIFF(s,'1970-01-01 00:00:00.000', x) as bigint)` | `DATEADD(ss, x, '01 JAN 1970')` |
| Date/Time | DATETIMEOFFSET | same as DATETIME | `DATEADD(ss, x, '01 JAN 1970') at time zone 'Central European Standard Time'` |
| Spatial | GEOMETRY | N/A | `geometry::STGeomFromText(concat_ws('','multipoint(',...,')'),0)` using `x % 90` for coords |
| Spatial | GEOGRAPHY | N/A | `geography::STGeomFromText(concat_ws('','multipoint(',...,')'),4326)` |
| Special | XML | N/A | `cast(replicate(concat('<foo>',x,'</foo>'), N) as XML)` |
| Special | HIERARCHYID | N/A | `concat(replicate(concat('/',x), N), '/')` |
| Special | SQL_VARIANT | `cast(x as bigint)` | `cast(cast(x as bigint) % 2147483648 as int)` |

---

## Oracle

| Category | Type | Type → Integer | Integer → Type |
|----------|------|----------------|----------------|
| Integer | NUMBER (tinyint) | `cast(x as number(19))` | `cast(mod(x, 256) as number(3))` |
| Integer | NUMBER (smallint) | `cast(x as number(19))` | `cast(mod(x, 32768) as number(5))` |
| Integer | NUMBER (mediumint) | `cast(x as number(19))` | `cast(mod(x, 8388608) as number(7))` |
| Integer | NUMBER (int) | `cast(x as number(19))` | `cast(mod(x, 2147483648) as number(10))` |
| Integer | NUMBER (bigint) | `cast(x as number(19))` | `cast(x as number(19))` |
| Boolean | NUMBER(1) / BIT | `cast(x as number(19))` | `cast(mod(cast(x as int), 2) as number(1))` |
| Float | BINARY_FLOAT | N/A | `cast(x as BINARY_FLOAT)` |
| Float | BINARY_DOUBLE | N/A | `cast(x as BINARY_DOUBLE)` |
| String | VARCHAR2(N) / NVARCHAR2(N) / CHAR / NCHAR | `cast(x as number(19))` | `substr(lpad(x, N, '0'), -N)` |
| Numeric | NUMBER(p,s) | `cast(x as number(19))` | `cast(mod(x, 10^p) / 10^s as NUMBER(p,s))` |
| Numeric | MONEY equiv NUMBER(19,d) | `cast(x as number(19))` | `cast(mod(x, max_val) / 10^d as NUMBER(19,d))` |
| UUID | RAW(16) UUID-style | `TO_NUMBER(RAWTOHEX(x),'XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX')` | `LOWER(REGEXP_REPLACE(RAWTOHEX(SYS_GUID()),...))` ⁶ |
| Binary | RAW(N) | `utl_raw.cast_to_number(x)` ⁷ | `hextoraw(dbms_crypto.randombytes(N))` |
| Binary | BLOB | N/A | `to_blob(dbms_crypto.randombytes(N))` |
| Binary | CLOB / NCLOB | N/A | `to_clob(dbms_crypto.randombytes(N))` / `to_nclob(...)` |
| Date/Time | DATE | `(x - date '1970-01-01') * 86400` | `DATE '1970-01-01' + NUMTODSINTERVAL(x, 'SECOND')` |
| Date/Time | TIMESTAMP / TIMESTAMP(n) | `(cast(x at time zone 'UTC' as date) - date '1970-01-01') * 86400` | `TIMESTAMP '1970-01-01 00:00:00' + NUMTODSINTERVAL(x, 'SECOND')` |
| Date/Time | TIMESTAMP WITH TIME ZONE / LOCAL | same as TIMESTAMP | `TIMESTAMP '1970-01-01 00:00:00 +02:00' + NUMTODSINTERVAL(x, 'SECOND')` |
| Interval | INTERVAL YEAR(n) TO MONTH | N/A | `NUMTOYMINTERVAL(random_val, 'MONTH')` ⁸ |
| Interval | INTERVAL DAY(n) TO SECOND(n) | N/A | `NUMTODSINTERVAL(random_val, 'SECOND')` ⁸ |
| Special | ROWID / UROWID | N/A | `DBMS_ROWID.ROWID_CREATE(1, x, x, x, x)` |
| Special | SDO_GEOMETRY | N/A | `SDO_GEOMETRY(2001, 8307, SDO_POINT_TYPE(mod(x,90), mod(x,90), NULL), NULL, NULL)` |
| Special | XMLTYPE | N/A | `XMLTYPE('<foo>' \|\| x \|\| '</foo>')` |
| Special | JSON | N/A | `JSON_OBJECT('value', x returning clob)` |

⁶ Oracle UUID uses `SYS_GUID()` — the result is **not** deterministic from `x`. Cannot reconstruct input integer.  
⁷ Assumes the RAW value was created with `utl_raw.cast_from_number(x)`.  
⁸ INTERVAL types use `random.randint(1,100)` in Python, not `x`. Not deterministic.

---

## Gaps: JavaSqlTypes with no castFuncs entry

| Type | Priority | Notes |
|------|----------|-------|
| `SERIAL4` / `SERIAL8` | Low | PostgreSQL alias for INT/BIGINT with sequence. Delegate to integer cast. |
| `ROWVERSION` | Medium | SQL Server 8-byte auto-increment binary. Skip in INSERT (DB manages it). |
| `LONG_RAW` | Medium | Oracle deprecated binary. Delegate to `castIntToBinary`. |
| `BFILE` | Low | Oracle file locator. `BFILENAME('MY_DIR', lpad(to_char(x),20,'0')\|\|'.bin')` |
| `SQLXML` | Low | Java synonym for XML. Add `SQLXML → castIntToXml`. |
| `NULL` | Low | Return `NULL`. |
| `ARRAY` | Medium | PostgreSQL `ARRAY[...]`. Requires element type introspection. |
| `STRUCT` | Low | Engine-specific composite type. |
| `REF_CURSOR` | N/A | Oracle OUT cursor. Not a column type. Skip. |
| `DATALINK` | Low | IBM DB2 only. `DATALINK(CAST('http://...' \|\| x AS VARCHAR(200)))` |
| `OTHER` / `DISTINCT` / `REF` | Low | Generic fallback to integer. |
