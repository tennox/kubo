# Routing Modes Analysis

## Question
How do different routing modes affect IPNS record storage and retrieval for a local-only gateway?

## Key Findings

### Routing Types vs DHT Participation

| Mode | DHT Server? | Local Storage | Network Traffic | Use Case |
|------|-------------|---------------|-----------------|----------|
| `dht` | ✅ YES | ✅ YES | Higher | Full DHT node |
| `dhtclient` | ❌ NO | ✅ YES | Lower | **Chosen for this setup** |
| `delegated` | ❌ NO | ❌ NO | Minimal | HTTP-only routers |
| `auto` | ✅ YES | ✅ YES | Higher | Default public node |
| `autoclient` | ❌ NO | ✅ YES | Lower | Like dhtclient |

### `dhtclient` Details
- Runs DHT **client only** — accepts no incoming DHT queries from peers
- Stores records in **local DHT datastore** (under `Repo.Datastore()`)
- Does **not** participate in DHT bootstrapping or peer discovery
- Perfect for isolated nodes that need local storage but minimal server burden

### IPNS Record Storage Flow
```
routing/put API
    ↓
routing.PutValue(key, record)
    ↓
DHT datastore
    ↓
Local storage (accessible to offline router)
```

### Gateway + NoFetch Resolution
When `Gateway.NoFetch=true`:
- Gateway creates **offline router** pointing to local datastore
- Resolves `/ipns/k51...` by querying **local DHT datastore only**
- ✅ Records stored via `routing/put` are findable
- ❌ Records stored via HTTP Routing V1 PUT are NOT stored locally

## Decision: dhtclient
**Chosen because:**
1. Records stored via RPC are persisted locally
2. Gateway can access them via offline router
3. Minimal server burden (no incoming DHT queries)
4. Manual peering still works for Bitswap exchange
5. With `Bootstrap: []`, prevents auto-join of public DHT
