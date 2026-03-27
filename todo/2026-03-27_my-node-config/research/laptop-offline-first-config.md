# Laptop Node Configuration: Local-First/Offline-First

## Use Case
Laptop running local-first application that:
- Stores blocks & IPNS records to local Kubo when offline
- Syncs with peer network (server + other machines) when online
- Discovers other nodes via MDNS on LAN
- Looks up content via direct peers or delegated routing

## Architecture Decision: `dhtclient` vs `delegated`

### Initial Confusion
Seemed like `delegated` mode (no DHT) would be sufficient since laptop only needs lookups. But **`dhtclient` is required** because:
- Laptop needs to store IPNS records via RPC (`routing/put`)
- Only DHT mode has local DHT datastore for persistence
- `delegated` mode has no local storage for records

### Chosen: `dhtclient`
```bash
ipfs config --json Routing.Type '"dhtclient"'
ipfs config --json Bootstrap '[]'
ipfs config --json Discovery.MDNS.Enabled true
ipfs config --json Ipns.UsePubsub true
```

**Why this works:**
- Local DHT datastore available for `routing/put`
- Isolated from public DHT (empty bootstrap)
- PubSub enables IPNS sync with peers
- MDNS auto-discovers other nodes on LAN

## Critical Challenge: Offline IPNS Updates

### The Problem
```
Offline scenario:
  1. Laptop stores IPNS record via routing/put (no network)
  2. Record saved in local DHT datastore
  3. Reconnect to network
  4. Peers don't automatically learn about the offline update
  5. PubSub wasn't running, so no announcement happened
```

### Why PubSub Alone Isn't Enough
- `routing/put` = raw DHT datastore operation
- Doesn't trigger IPNS republishing system
- Doesn't announce via PubSub
- Record stays local until explicitly queried

### Possible Solutions

#### Option 1: Use `ipfs name publish` for offline updates
- **Pros**: Stores in keystore, triggers republishing + PubSub automatically
- **Cons**: Requires private key on laptop, only works for own IPNS names
- **Use case**: If laptop publishes its own content

#### Option 2: Manual republish on reconnect
- **Pros**: Full control, can batch sync multiple records
- **Cons**: Requires app-level coordination, extra complexity
- **Use case**: Application explicitly handles sync on network restore

#### Option 3: Accept the limitation
- **Pros**: Simpler, no extra logic needed
- **Cons**: Offline IPNS updates don't propagate until manually synced
- **Use case**: Rare offline updates, can be repeated when online

### Unresolved Question
**How should offline IPNS updates be handled in the application?**

Depends on:
- Does laptop publish its own IPNS names, or just store/cache others?
- How critical is immediate peer visibility of offline-created records?
- Can user manually trigger sync, or must it be automatic?

## Laptop Node Config Summary

```bash
# Isolated DHT client (like server)
ipfs config --json Routing.Type '"dhtclient"'
ipfs config --json Bootstrap '[]'

# IPNS synchronization across peers
ipfs config --json Ipns.UsePubsub true

# Local network discovery
ipfs config --json Discovery.MDNS.Enabled true

# Can fetch from peers when online
ipfs config --json Gateway.NoFetch false

# Store records locally
ipfs config --json Provide.Strategy '"all"' or '"pinned"'
# (Optional - allows announcing stored blocks to peers)

# Content publishing
ipfs config --json Routing.Type '"dhtclient"'
# Can use routing/put to store IPNS records
```

## Block Exchange
Bitswap handles peer-to-peer block transfer automatically:
- Laptop asks peer for block
- Peer has it → sends it
- Block caching/syncing works without extra configuration
