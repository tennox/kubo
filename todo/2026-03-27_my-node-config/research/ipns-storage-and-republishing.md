# IPNS Storage and Republishing

## Question
Which RPC APIs store IPNS records locally for automatic republishing?

## Findings

### Three Ways to Store IPNS Records

#### 1. `ipfs name publish` (CLI/RPC)
- **Stores in**: Local keystore
- **Republishing**: ✅ YES — IPNS republisher tracks and re-announces automatically
- **Use case**: Publishing your own IPNS names
- **Reprovision interval**: `Ipns.RepublishPeriod` (default: 24h)

#### 2. `routing/put` RPC API
- **Stores in**: DHT datastore (local only if using dhtclient)
- **Republishing**: ❌ NO — one-time store only
- **Use case**: Storing raw pre-signed IPNS records manually
- **Note**: `ipfs name put` CLI wraps this with additional validation

#### 3. HTTP Routing V1 PUT Endpoint
- **Stores in**: Remote HTTP server only
- **Republishing**: ❌ NO — delegated to HTTP routing server
- **Use case**: Publishing to external routing infrastructure
- **Note**: Records NOT stored locally, gateway can't access

### For This Setup
- Use **`routing/put`** for storing blocks and IPNS records via RPC
- Records persist in local DHT datastore
- Gateway can access them with `Gateway.NoFetch=true`
- If auto-republishing is needed between servers, rely on **PubSub** (`Ipns.UsePubsub=true`)

### PubSub for Peer Synchronization
When two manually-peered servers have `Ipns.UsePubsub: true`:
- Server A updates IPNS → announces via PubSub
- Server B receives update → invalidates cache automatically
- Without PubSub: Server B serves stale data until TTL expires

**For your cluster: Keep PubSub enabled** so peers stay synchronized on IPNS changes.
