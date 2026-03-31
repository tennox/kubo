# Pinning Incomplete DAGs in Kubo

## Question
After importing CAR files incrementally, how can we safely pin a DAG without hanging/timing out when blocks are missing? Is there a way to verify DAG completeness before pinning?

## Short Answer
Kubo has **no built-in partial/best-effort recursive pin**. Pinning is all-or-nothing. However, `ipfs --offline refs -r` can verify completeness before pinning, and the RPC API supports this via `?offline=true`.

## Key Findings

### `pin add` hangs on missing blocks

`pin add` calls `FetchGraph()` which walks the entire DAG. The pinner is initialized with the node's **online** DAG service (Bitswap-backed):

- **Online (default):** hangs waiting for Bitswap to find missing blocks — can stall indefinitely
- **`ipfs --offline pin add`:** uses CoreAPI's offline DAG service → fails immediately on missing blocks, no pin created

Code path: `pin add` → `cmdenv.GetApi()` (respects `--offline`) → `api.Pin().Add()` → `api.pinning.Pin()` — the CoreAPI's pinner uses `api.dag` which **is** swapped to offline when `--offline` is set.

Source: `core/coreapi/pin.go:42`, `core/coreapi/coreapi.go:244-249`

### `dag import --pin-roots` also hangs (despite comments suggesting otherwise)

The import code at `core/commands/dag/import.go:46` sets `api` to offline mode:
```go
api, err = api.WithOptions(options.Api.Offline(true))
```

But the pinning step at line 177 bypasses the offline API entirely:
```go
node.Pinning.Pin(req.Context, nd, true, "")
```

`node.Pinning` holds a reference to `node.DAG` — the node's **original online DAG service** — not the offline API created above. The offline mode only applies to the block import batch (`api.Dag()`), not to the pinner's internal `FetchGraph()` walk.

**The comment "on import ensure we do not reach out to the network for any reason" at line 43 is misleading** — it only holds for the block import phase, not the pinning phase.

The error handling is per-root and non-fatal (lines 166-184, "opportunistic pinning: try whatever sticks"), but if blocks are missing, `FetchGraph()` will still attempt Bitswap resolution and potentially hang before the error is captured.

### `refs` correctly respects `--offline`

`refs` at `core/commands/refs.go:80` calls `cmdenv.GetApi(env, req)` which checks the global `--offline` flag. It then builds its DAG walker from the API's DAG service (line 116):
```go
DAG: merkledag.NewSession(ctx, api.Dag())
```

With `--offline`, this DAG service uses `offlinexch.Exchange` → fails immediately on missing blocks.

**RPC endpoint:** `POST /api/v0/refs?arg=<CID>&recursive=true&offline=true`

### `pin verify` is always offline (but only for existing pins)

`PinAPI.Verify()` at `core/coreapi/pin.go:191-197` hardcodes its own offline DAG service:
```go
DAG := merkledag.NewDAGService(bserv.New(bs, offline.Exchange(bs)))
```

This never goes to the network regardless of flags. But it only checks **already recursively-pinned** CIDs, so it can't be used for pre-pin verification.

### No `--sparse` or partial pin support

[Issue #5188](https://github.com/ipfs/kubo/issues/5188) (filed 2018, status: deferred) requests `--sparse` pinning for incomplete DAGs. Unimplemented.

## Verification Approaches (fastest to slowest)

### 1. `refs local` + CAR link extraction (fastest, requires external tooling)
```bash
# Sequential blockstore scan — no DAG walk
ipfs refs local > local-blocks.txt
# Extract links from CAR files (sequential file IO)
car list --links *.car > all-links.txt
# Diff (requires CID version normalization)
comm -23 <(sort -u all-links.txt) <(sort -u local-blocks.txt)
```
**Caveat:** `refs local` returns CIDv1-Raw for all blocks. CID version normalization (e.g. comparing by multihash) may be needed.

### 2. `ipfs --offline refs -r <CID>` (pure kubo, slower)
```bash
ipfs --offline refs -r <root-CID> > /dev/null
# RPC: POST /api/v0/refs?arg=<CID>&recursive=true&offline=true
```
Walks the entire DAG via random blockstore reads. Fails immediately on first missing block. Slow for large DAGs (hundreds of thousands of blocks) due to random I/O, but guaranteed not to hang.

### 3. `pin verify` (only for already-pinned roots)
```bash
ipfs pin verify
```
Always offline (hardcoded). Reports `BadNodes` with CIDs and errors. Only useful for post-hoc verification of existing pins.

## Recommended Workflow

```bash
# 1. Import CAR files without pinning (always succeeds)
ipfs dag import --pin-roots=false chunk1.car chunk2.car chunk3.car

# 2. Verify DAG completeness offline (fail-fast, no hang)
ipfs --offline refs -r <root-CID> > /dev/null 2>&1

# 3. Only pin if complete (offline ensures no network even if something changed)
if [ $? -eq 0 ]; then
    ipfs --offline pin add <root-CID>
fi
```

## Summary Table

| Method | Network? | Hang risk | Pre-pin check | Speed |
|---|---|---|---|---|
| `pin add` (online) | Bitswap | **YES** | N/A | Slow (hangs) |
| `--offline pin add` | No | No (fail-fast) | N/A | Slow (DAG walk) |
| `dag import --pin-roots` | **YES** (pinner bypasses offline API) | **YES** | N/A | Slow |
| `--offline refs -r` | No | No (fail-fast) | **Yes** | Slow (random reads) |
| `refs local` + link diff | No | No | **Yes** | Fast (sequential) |
| `pin verify` | No (hardcoded offline) | No | No (existing pins only) | Slow (DAG walk) |

## Potential Kubo Improvements

1. **Fix `dag import --pin-roots`** to use the offline API's pinner (or create an offline DAG service for the pin step), matching the comment's intent
2. **Implement `--sparse` pinning** ([#5188](https://github.com/ipfs/kubo/issues/5188)) for best-effort recursive pins on incomplete DAGs
3. **Add `--offline` to `pin add` as a first-class flag** rather than relying on the global flag
