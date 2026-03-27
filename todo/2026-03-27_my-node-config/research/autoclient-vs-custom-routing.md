# Autoclient vs Custom Routing: Parallel vs Sequential Composition

## Question
Does `Routing.Type: "autoclient"` truly query peers first with delegated routing as fallback?

## Answer: Partially — It's Parallel, Not Sequential

### Autoclient Routing Behavior

**Code shows `autoclient` uses parallel composition:**
```
ConstructDefaultRouting(cfg, DHTClientOption)
  └─ Creates ParallelRouter with:
     - DHT client router
     - HTTP delegated routers
```

**Parallel routing means:**
- DHT and delegated routers are queried **simultaneously**
- Whichever responds first wins
- Usually DHT is faster (local peer response < HTTP latency)
- But no guarantee of peer-first

**In practice:**
- ✅ Mostly works: Local peer (server) responds faster than HTTP
- ⚠️ Race condition: If delegated service responds faster, uses that
- ⚠️ Both routers are always queried (unnecessary if peer has answer)

### Custom Sequential Routing

**For guaranteed peer-first with delegated fallback:**

```bash
ipfs config --json Routing.Type '"custom"'
ipfs config --json Routing.Routers '{
  "dht": {
    "Type": "dht",
    "Parameters": {
      "Mode": "client"
    }
  },
  "delegated": {
    "Type": "http",
    "Parameters": {
      "Endpoint": "https://delegated-ipfs.dev"
    }
  }
}'
ipfs config --json Routing.Methods '{
  "get-ipns": {"RouterName": "dht"},
  "put-ipns": {"RouterName": "dht"},
  "find-providers": {
    "SequentialRouter": [
      {"RouterName": "dht"},
      {"RouterName": "delegated"}
    ]
  },
  "find-peers": {"RouterName": "dht"},
  "provide": {"RouterName": "dht"}
}'
```

**Sequential routing means:**
- Try DHT first
- **Only if DHT fails**, try delegated
- Guaranteed peer-first
- More resource efficient (don't query both)
- But slightly slower if peer doesn't have record

### Comparison

| Aspect | Autoclient (Parallel) | Custom (Sequential) |
|--------|----------------------|-------------------|
| Config complexity | Simple | Complex |
| Query behavior | Both at once | Peer first, then fallback |
| Performance (found locally) | Slightly faster | Slightly slower |
| Performance (not found) | Saves time querying both | Same (queries both anyway) |
| Network efficiency | Queries both routers always | Saves delegated query when peer has it |
| Guaranteed peer-first | ❌ Race condition | ✅ Yes |
| Bootstrap isolation | ✅ Yes (with Bootstrap: []) | ✅ Yes (with Bootstrap: []) |

## Recommendation for Your Setup

### If prioritizing simplicity:
Use `autoclient` — parallel routing is "good enough"
- Local peer (server) almost always responds faster than HTTP
- Unlikely to have race condition in practice
- Less config to maintain

```bash
ipfs config --json Routing.Type '"autoclient"'
ipfs config --json Bootstrap '[]'
ipfs config --json Routing.DelegatedRouters '["https://delegated-ipfs.dev"]'
```

### If prioritizing reliability:
Use custom sequential routing — explicit peer-first guarantee
- No race conditions
- More efficient (avoids unnecessary delegated queries)
- But requires complex routing config

## Conclusion

**Autoclient is parallel (not sequential), but works well in practice** because:
- Local peer response is usually much faster than HTTP
- Delegated routing is effectively a fallback
- Bootstrap isolation still maintained

For your laptop config, `autoclient` is simpler and practical. Custom sequential routing only needed if you want absolute guarantee of peer-first with zero parallel queries.
