# DHT Queries in Isolated Network (dhtclient + Bootstrap: [])

## Question
Can DHT GetValue queries find IPNS records stored on manually connected peers when the node is isolated from public DHT?

## Answer: YES

**DHT queries work across manually connected peers** even with `Bootstrap: []` and isolated setup.

## How It Works

### DHT Query Process
1. Node calls `ipfs routing get /ipns/k51...`
2. DHT uses its **routing table** to locate the record
3. With `Bootstrap: []`, routing table contains only **manually connected peers**
4. DHT queries those peers recursively to find the value
5. If any peer has the record, query succeeds

### Routing Table Population
- `Bootstrap: []` prevents auto-discovery
- But manually connected peers are **explicitly added to routing table**
- When you do `ipfs swarm connect /ip4/.../p2p/...`, that peer joins the DHT routing table
- DHT can then traverse through them to find values

## Practical Use Case: Offline Sync

When peers reconnect after offline period:

```bash
# Laptop connects to server
ipfs swarm connect /ip4/SERVER_IP/tcp/4001/p2p/SERVER_PEER_ID

# Laptop queries for IPNS records that were stored while offline
ipfs routing get /ipns/k51...
# DHT finds it on server via routing table, returns it
```

**This works because:** Server is now in laptop's routing table, DHT can query it directly.

## Limitations

- **Scope is limited to connected peers** — Can't reach beyond your manual peer network
- **Requires active peer connections** — If peers aren't connected, query fails
- **No distributed discovery** — Unlike public DHT, can't find peers via network hops
- **Requires peer to have the record** — One of your peers must be storing the value

## Difference from Public DHT

| Aspect | Public DHT | Isolated DHT |
|--------|-----------|-------------|
| Bootstrap | Public peers | Empty |
| Routing table | Large (hundreds of peers) | Small (your peers only) |
| Query scope | Network-wide | Limited to connected peers |
| GetValue reach | Can find any published record | Only finds on your peers |

## Application for Offline-First Sync

**Solves the offline IPNS problem partially:**
- Peers can query each other for stored IPNS records
- Not automatic (like PubSub), but explicit/manual
- Requires application to call `ipfs routing get` after reconnecting

**Example workflow:**
1. Laptop offline, stores IPNS record locally
2. Reconnects to network, connects to server
3. Server calls `ipfs routing get` to discover laptop's new record
4. Or laptop calls `ipfs routing get` to check if server has newer version
5. Explicit sync replaces missing PubSub history

## Conclusion

**DHT queries between isolated peers work and can be used for post-reconnect sync.** This is a manual alternative to PubSub when peers come back online.
