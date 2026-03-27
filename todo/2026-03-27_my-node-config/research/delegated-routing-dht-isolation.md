# Delegated Routing vs DHT Isolation

## Question
If the laptop uses delegated routing (public HTTP routers), will it accidentally connect to public DHT peers and compromise isolation?

## Answer: NO

**Delegated routing and DHT peer discovery are completely separate systems.** Using delegated routing doesn't cause DHT peer connections.

## How They're Separate

### Delegated Routing
- **What it does**: Makes HTTP GET requests to public routing service
- **Returns**: Peer information (multiaddrs) for content providers
- **Side effects**: Laptop might fetch blocks from returned peers via Bitswap
- **DHT impact**: NONE — peers are not added to DHT routing table

### DHT Peer Discovery
- **What it does**: Bootstrap to known DHT peers, gradually discover more peers
- **Requires**: Non-empty `Bootstrap` list
- **Your setup**: `Bootstrap: []` (empty)
- **Result**: DHT never starts bootstrap, stays isolated

## Routing Table Population

**DHT routing table is only populated by:**
1. Bootstrap peers (you have none)
2. Manual `ipfs swarm connect` commands (your server only)
3. Incoming peer connections (you have none from public DHT)

**NOT by:**
- Delegated routing responses ❌
- Peer information from HTTP calls ❌
- Incidental peer discovery ❌

## What Happens With Delegated Routing

```
Laptop queries delegated router:
  GET https://router.example.com/routing/v1/providers/CID
  ↓
  Returns list of peers: [Peer A, Peer B, Peer C]
  ↓
  Laptop can fetch blocks from them via Bitswap
  ↓
  But peers are NOT added to DHT routing table
  ↓
  DHT remains isolated with only server peer
```

## Safe Architecture

**Laptop can safely use delegated routing because:**
- ✅ `Bootstrap: []` prevents DHT bootstrap regardless
- ✅ Delegated routing is stateless HTTP calls
- ✅ Returned peers don't enter DHT system
- ✅ Server peer remains the only DHT neighbor
- ✅ No automatic peer discovery or DHT joining

**Server doesn't expose laptop:**
- ✅ Server also has `Bootstrap: []`
- ✅ Server has `Provide.Strategy: "none"` (doesn't announce)
- ✅ Server doesn't share its peer list via DHT

## Delegated Routing Limitations

While safe for isolation, delegated routing has limits:
- **Provider discovery only** — Can find who has content
- **No IPNS from public routers** — Some don't support IPNS lookups
- **Centralized** — Depends on external service availability
- **Privacy** — Queries visible to router operator

For your setup, delegated routing is a **fallback for content discovery**, not a threat to isolation.

## Recommended Use

```bash
# Laptop can safely use delegated routing as fallback
# (default cid.contact is fine for provider lookups)

# But for IPNS:
# - Rely on server peer (via DHT query after reconnect)
# - Or use PubSub if peers are connected
# - Delegated routers may not support IPNS
```

## Conclusion

**Using delegated routing in an isolated DHT setup is safe.** It's HTTP-only with no DHT peer discovery implications. Your network stays isolated while gaining access to content discovery as a fallback mechanism.
