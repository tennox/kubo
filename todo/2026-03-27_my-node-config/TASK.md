# Kubo Configuration: Private Storage + Gateway Node

## Goal
Configure a Kubo IPFS node as a **private, isolated server** that:
- Stores blocks and IPNS records via RPC API (no DHT publishing)
- Serves content via local-only gateway (`Gateway.NoFetch=true`)
- Can peer with other private servers for block exchange
- Minimizes network overhead and traffic (no public DHT participation)

## Node Type
**Private Storage + Local Gateway Server** with optional manual peering to other infrastructure nodes.

### Key Characteristics
- **Routing**: DHT client mode (`dhtclient`), isolated from public DHT (`Bootstrap: []`)
- **Storage**: Local blockstore + local DHT datastore for IPNS records
- **Content Discovery**: Manual peer connections only (no DHT bootstrap, no MDNS)
- **Gateway**: Local-only (`NoFetch=true`) — serves only locally stored content
- **Publishing**: Via RPC APIs (`/api/v0/add`, `/api/v0/routing/put`) — no DHT announcements
- **Block Exchange**: Between manually connected peers via Bitswap

## Status
✅ COMPLETE — Both server and laptop configurations finalized and documented

## Key Decisions Made
1. **`Routing.Type: dhtclient`** — DHT client mode, no server overhead
2. **`Bootstrap: []`** — Prevent auto-connection to public DHT
3. **`Provide.Strategy: "none"`** — Don't announce blocks to DHT
4. **`Ipns.UsePubsub: true`** — Enable PubSub for IPNS propagation between manually connected peers
5. **`Gateway.NoFetch: true`** — Gateway serves only local content
6. **Connection limits** — Configured for server with 2-10 peer connections max
7. **Relay disabled** — No relay client/service to reduce network burden

## Progress Notes
- ✅ Investigated DHT routing vs HTTP delegated routing
- ✅ Determined `routing/put` stores records in local DHT datastore when using DHT client
- ✅ Confirmed `Gateway.NoFetch=true` reads from local datastore via offline router
- ✅ Reviewed relay, discovery, and connection management settings
- ✅ Finalized config with all necessary isolation settings

## Related Work
- **Laptop node config** — Local-first/offline-first application node (separate from server)

## Files
- `PLAN.md` — Server configuration with reasoning
- `LAPTOP-PLAN.md` — Laptop configuration for local-first apps
- `research/` — Investigation notes on routing, storage, gateway behavior, and offline IPNS challenges
