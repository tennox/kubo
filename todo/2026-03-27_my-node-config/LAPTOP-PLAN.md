# Laptop Node Final Configuration

## Use Case
Local-first/offline-first application node that:
- Stores blocks + IPNS records locally via RPC
- Syncs with peer network (server + other machines) when online
- Auto-discovers peers on LAN via MDNS
- Announces pinned content to peers for efficient discovery
- Can fetch from peers or public delegated routing as fallback

## Configuration Rationale

### Routing: Isolated DHT Client
```bash
ipfs config --json Routing.Type '"dhtclient"'
ipfs config --json Bootstrap '[]'
```

**Why dhtclient:**
- Needs local DHT datastore for `routing/put` (store IPNS records)
- Isolated from public DHT (empty bootstrap)
- Can query peer network for content lookups
- Minimal server overhead (no incoming DHT queries)

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

## Known Limitation: Offline IPNS Updates

When storing IPNS records while offline via `routing/put`:
- Records saved to local datastore
- Not announced to peers on reconnect (PubSub wasn't running)
- Peers won't know about update until explicitly queried

**Current workaround:** Accept limitation, or use `ipfs name publish` if laptop has private keys to manage.

## Complete Laptop Config

```bash
# Routing & Bootstrap
ipfs config --json Routing.Type '"dhtclient"'
ipfs config --json Bootstrap '[]'

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

# Maintenance
ipfs config --json Datastore.GCPeriod '"24h"'

# Other settings (same as server)
ipfs config --json DNS.Resolvers '{".": "https://cloudflare-dns.com/dns-query"}'
ipfs config --json Discovery.MDNS.Enabled false  # Remove this if using MDNS
ipfs config --json Swarm.RelayClient.Enabled true
ipfs config --json Swarm.RelayService.Enabled false
ipfs config --json Swarm.Transports.Network.Relay false
```
