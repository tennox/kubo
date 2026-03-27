# Laptop Node Final Configuration

## Use Case
Local-first/offline-first application node that:
- Stores blocks + IPNS records locally via RPC
- Syncs with peer network (server + other machines) when online
- Auto-discovers peers on LAN via MDNS
- Announces pinned content to peers for efficient discovery
- Can fetch from peers or public delegated routing as fallback

## Configuration Rationale

### Routing: Hybrid with Peer-First Lookup
```bash
ipfs config --json Routing.Type '"autoclient"'
ipfs config --json Bootstrap '[]'
ipfs config --json Routing.DelegatedRouters '["https://delegated-ipfs.dev"]'
```

**Why autoclient with delegated fallback:**
- Needs local DHT datastore for `routing/put` (store IPNS records) — autoclient includes DHT client
- Isolated from public DHT (empty bootstrap)
- **Parallel routing**: DHT and delegated routers queried simultaneously
- **In practice**: Local peer (server) responds faster than HTTP, so peer-first lookup
- Delegated routing provides fallback when content not on peer network
- Hybrid approach balances simplicity with fallback reliability

**Alternative: Custom sequential routing**
- If you want guaranteed sequential (try peer first, only then delegated)
- Requires complex routing config
- More efficient (avoids parallel queries)
- See research/autoclient-vs-custom-routing.md for details

### IPNS Synchronization
```bash
ipfs config --json Ipns.UsePubsub true
ipfs config --json Ipns.MaxCacheTTL '"1h"'
```

**Why:**
- PubSub propagates IPNS updates between peered machines
- When any peer updates an IPNS record, others learn about it
- 1h TTL balances freshness vs network queries
- Critical for keeping peer network in sync

### Peer Discovery: LAN Auto-discovery
```bash
ipfs config --json Discovery.MDNS.Enabled true
```

**Why:**
- Auto-discovers other machines on same LAN
- No manual peer connections needed for local devices
- Still requires manual connections for remote peers (servers)

### Content Announcing: Pinned Only
```bash
ipfs config --json Provide.Strategy '"pinned"'
```

**Why:**
- App pins all stored DAGs recursively
- `"pinned"` = announce only explicitly kept content
- Allows peer network to discover blocks efficiently
- Minimal network chatter (small peer network, one-time announcements)
- Zero cost when idle

### Gateway: Fetch from Peers
```bash
ipfs config --json Gateway.NoFetch false
```

**Why:**
- Unlike server, laptop should fetch from peer network
- Localhost-only gateway prevents external abuse
- Can retrieve blocks that don't exist locally

### Relay Client: NAT Traversal
```bash
ipfs config --json Swarm.RelayClient.Enabled true
ipfs config --json Swarm.RelayService.Enabled false
```

**Why:**
- Laptop may be behind restrictive NAT (hotel WiFi, etc.)
- RelayClient allows reaching peers via relay through server
- RelayService disabled: don't relay for other peers (save resources)
- Minimal overhead in small peer network

### Delegated Routing: Fallback for Content Discovery
```bash
# Default: uses cid.contact (provider lookup only)
# For IPNS support, optionally use:
ipfs config --json Routing.DelegatedRouters '["https://delegated-ipfs.dev"]'
```

**Why:**
- Provides fallback when content not found on peer network
- `cid.contact` is default, finds CID providers only
- `delegated-ipfs.dev` adds IPNS lookup support
- **Does NOT compromise isolation**: HTTP-only, peers from responses don't enter DHT
- **Safe**: `Bootstrap: []` prevents DHT exposure regardless

**How it works:**
1. Laptop queries server peer first (fastest)
2. If not found, queries delegated routing service
3. Delegated service returns peer list
4. Laptop fetches blocks from those peers directly
5. No DHT peer discovery occurs

**Important:** Delegated routing is optional and only used if content not found locally or on peers. With good peer coverage (server + LAN devices), won't be needed often.

### Peer Connection Management
```bash
ipfs config --json Swarm.ConnMgr.Type '"basic"'
ipfs config --json Swarm.ConnMgr.LowWater 2
ipfs config --json Swarm.ConnMgr.HighWater 5
ipfs config --json Swarm.ConnMgr.GracePeriod '"30s"'
```

**Why these limits:**
- Laptop will have ~3-5 peers (server + maybe other devices)
- `HighWater: 5` is sufficient for small peer network
- Lower limits than server = less memory/battery drain
- Graceful disconnection prevents connection churn

### Maintenance
```bash
ipfs config --json Datastore.GCPeriod '"24h"'
```

**Why:**
- Same as server: reasonable balance
- Laptop storage is managed by app (pins recursively)
- 24h cycle prevents excessive disk I/O

## Differences from Server

| Aspect | Server | Laptop |
|--------|--------|--------|
| Role | Primary storage | Client + local cache |
| `Provide.Strategy` | `"none"` | `"pinned"` |
| `Gateway.NoFetch` | `true` | `false` |
| `ConnMgr.HighWater` | 10 | 5 |
| Purpose | Store & serve | Fetch & announce |

Server doesn't announce (it's authoritative storage). Laptop announces so peers can find cached blocks.

## Content Lookup Workflow

When laptop needs to find content (CID or IPNS):

### Default Behavior (Peer Network First)
1. **Query server peer** via DHT (fastest, no internet needed)
   - `ipfs routing get /ipns/k51...`
   - Works even offline if server is nearby
2. **MDNS-discovered LAN peers** (if available)
   - Auto-discovered, shares content via Bitswap
3. **PubSub cache** (if peers are connected)
   - IPNS updates via PubSub keep records fresh
4. **Cache TTL** fallback (1h)
   - If PubSub fails, stale record eventually refreshes

### Fallback to Public Routers (Optional)
If content not found on peers:
1. Delegated routing service (if configured)
   - Default: `cid.contact` (CID providers only)
   - Optional: `delegated-ipfs.dev` (adds IPNS support)
2. Service returns peer list with content
3. Laptop fetches blocks from returned peers

**Example sequence when offline from server but online to internet:**
```
Need CID → Query server (fails, offline) → Query delegated router → Get peers → Fetch blocks
Need IPNS → Query server (fails) → Query delegated router → Get IPNS record → Cache it
```

### Important: No DHT Exposure
- Delegated routing returns peers, but they're NOT added to DHT routing table
- Only your server is in DHT routing table (no bootstrap peers)
- No public DHT connection occurs, regardless of delegated routing use

## Known Limitation: Offline IPNS Updates

When storing IPNS records while offline via `routing/put`:
- Records saved to local datastore
- Not announced to peers on reconnect (PubSub wasn't running)
- Peers won't know about update until explicitly queried via DHT

**Workarounds:**
1. **Explicit DHT sync** — After reconnecting: `ipfs routing get /ipns/k51...` to check peers
2. **Use `ipfs name publish`** — If laptop has private keys to manage own IPNS names
3. **Accept limitation** — Offline updates sync when peer is queried for the record

## Complete Laptop Config

```bash
# Routing & Bootstrap - Hybrid with delegated fallback
ipfs config --json Routing.Type '"autoclient"'
ipfs config --json Bootstrap '[]'
ipfs config --json Routing.DelegatedRouters '["https://delegated-ipfs.dev"]'

# IPNS
ipfs config --json Ipns.UsePubsub true
ipfs config --json Ipns.MaxCacheTTL '"1h"'
ipfs config --json Ipns.ResolveCacheSize 256

# Content announcing
ipfs config --json Provide.Strategy '"pinned"'

# Peer discovery
ipfs config --json Discovery.MDNS.Enabled true

# Connection management
ipfs config --json Swarm.ConnMgr.Type '"basic"'
ipfs config --json Swarm.ConnMgr.LowWater 2
ipfs config --json Swarm.ConnMgr.HighWater 5
ipfs config --json Swarm.ConnMgr.GracePeriod '"30s"'

# Gateway
ipfs config --json Gateway.NoFetch false

# Relay for NAT traversal
ipfs config --json Swarm.RelayClient.Enabled true
ipfs config --json Swarm.RelayService.Enabled false

# Maintenance
ipfs config --json Datastore.GCPeriod '"24h"'

# DNS
ipfs config --json DNS.Resolvers '{".": "https://cloudflare-dns.com/dns-query"}'
```
