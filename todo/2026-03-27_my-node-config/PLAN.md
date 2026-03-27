# Kubo Configuration Plan: Private Storage + Local Gateway Node

## Node Architecture
**Private isolated IPFS node** with:
- RPC-driven content storage (blocks + IPNS records)
- Local-only gateway serving
- Optional manual peering to infrastructure servers
- Minimal DHT/network overhead

## Configuration Strategy

### 1. Routing: Isolated DHT Client Mode
```bash
ipfs config --json Routing.Type '"dhtclient"'
ipfs config --json Bootstrap '[]'
```

**Why:**
- `dhtclient`: Stores records locally, no incoming DHT server burden
- `Bootstrap: []`: Prevents auto-discovery and connection to public DHT nodes
- Records stored via `routing/put` persist in local DHT datastore
- Gateway's offline router can access them

### 2. DNS & IPNS Caching
```bash
ipfs config --json DNS.Resolvers '{".": "https://cloudflare-dns.com/dns-query"}'
ipfs config --json Ipns.UsePubsub true
ipfs config --json Ipns.MaxCacheTTL '"1h"'
ipfs config --json Ipns.ResolveCacheSize 256
```

**Why:**
- Custom DNS resolver: Explicit control over DNS queries
- `Ipns.UsePubsub: true`: Enables PubSub for peer synchronization when manually connected to other servers
- Cache TTL: 1h balances freshness vs. network queries
- Cache size: 256 entries reasonable for a server

### 3. Gateway: Local-Only Content
```bash
ipfs config --json Gateway.NoFetch true
ipfs config --json Gateway.HTTPHeaders.Cache-Control '["public, max-age=3600"]'
```

**Why:**
- `NoFetch: true`: Gateway queries offline router (local datastore only)
- Records from `routing/put` are discoverable
- HTTP cache headers: Help clients and CDNs cache responses
- No network fetch on gateway requests

### 4. Content Publishing: No DHT Announcements
```bash
ipfs config --json Provide.Strategy '"none"'
ipfs config --json Experimental.StrategicProviding false
```

**Why:**
- Don't announce blocks to DHT (they won't reach public peers anyway)
- Reduces network traffic and CPU overhead
- Blocks still served to directly connected peers via Bitswap

### 5. Peer Discovery: Manual Only
```bash
ipfs config --json Discovery.MDNS.Enabled false
ipfs config --json Swarm.DisableNatPortMap true
ipfs config --json Swarm.RelayClient.Enabled true
ipfs config --json Swarm.RelayService.Enabled false
```

**Why:**
- MDNS disabled: No local network peer discovery (manual only)
- NAT port mapping disabled: Explicit control over port forwarding
- RelayClient enabled: Allows reaching peers behind NAT (e.g., laptop in hotel WiFi reaching office workstation)
- RelayService disabled: Don't relay traffic for other peers
- Network isolation: Only manually configured peers

### 6. Connection Management
```bash
ipfs config --json Swarm.ConnMgr.Type '"basic"'
ipfs config --json Swarm.ConnMgr.LowWater 2
ipfs config --json Swarm.ConnMgr.HighWater 10
ipfs config --json Swarm.ConnMgr.GracePeriod '"30s"'
```

**Why:**
- Basic connection manager: Simple, predictable resource usage
- LowWater=2, HighWater=10: Allows manual peer connections without opening more
- GracePeriod=30s: Reasonable timeout for graceful disconnection

### 7. Resource Limits
```bash
ipfs config --json Swarm.ResourceMgr.Limits.System '{
  "Conns": 100,
  "ConnsInbound": 30,
  "ConnsOutbound": 70,
  "Streams": 300
}'
```

Or as a single line:
```bash
ipfs config --json Swarm.ResourceMgr.Limits.System '{"Conns":100,"ConnsInbound":30,"ConnsOutbound":70,"Streams":300}'
```

**Why:**
- Conservative limits protect server from runaway connections
- InboundConns=30: Prevents abuse from random peers (though none should connect)
- OutboundConns=70: Room for multiple manual peer connections
- Streams=300: Reasonable for a few peers

### 8. Maintenance
```bash
ipfs config --json Datastore.GCPeriod '"24h"'
```

**Why:**
- 24h GC cycle: Reasonable balance between freeing space and disk I/O
- Not aggressive (which would be wasteful on a server)

### 9. API Security
```bash
ipfs config --json API.HTTPHeaders.Access-Control-Allow-Origin '["http://localhost:3000", "http://localhost:5173", "http://localhost:5001", "http://127.0.0.1:5001", "https://webui.ipfs.io"]'
```

**Why:**
- CORS limited to trusted local ports + webui.ipfs.io
- Prevents accidental XSS attacks from untrusted websites

## Manual Peering Setup

To peer with other infrastructure servers:

```bash
# On remote server, get peer info
ipfs id

# On this server, connect to remote
ipfs swarm connect /ip4/REMOTE_IP/tcp/4001/p2p/PEER_ID
```

Once connected:
- Bitswap enables block exchange
- PubSub enables IPNS updates propagation
- No DHT queries needed

## Storage & Retrieval Flow

### Adding content
```bash
curl -F file=@myfile.txt http://localhost:5001/api/v0/add
# Blocks stored in local blockstore
```

### Publishing IPNS record
```bash
curl -X POST -F data=@record.bin \
  http://localhost:5001/api/v0/routing/put?arg=/ipns/k51... \
  &allow-offline=true
# Record stored in local DHT datastore
```

### Serving via gateway
```bash
curl http://localhost:8080/ipns/k51...
# Gateway queries offline router → finds record in local datastore
# Serves content from local blockstore
```

## Implementation Notes

### What's NOT in this config
- No `Addresses.Swarm` restriction to localhost (allows manual remote peering)
- Bitswap remains enabled (needed for peer block exchange)
- No custom routing (uses default dhtclient setup)

### What changes from defaults
- Routing isolated from public DHT
- Gateway offline-only
- Content not published to DHT
- Discovery mechanisms disabled
- Relay services disabled

## Verification

After applying config, verify:
```bash
ipfs config show | jq '.Routing, .Gateway, .Bootstrap, .Provide'
```

Should show:
- `Routing.Type: "dhtclient"`
- `Bootstrap: []`
- `Gateway.NoFetch: true`
- `Provide.Strategy: "none"`
