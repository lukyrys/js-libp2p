# fix(kad-dht): prevent Multiaddr retention via pEvent listener leak

## Summary
Heap-verified memory leak in `@libp2p/kad-dht` where Multiaddr/PeerInfo instances accumulate unbounded during normal DHT operation, especially visible in Bun/JSC runtime.

## Observed impact (LiberShare production, Bun 1.3)

5 nodes running for ~3h on 16.1.6 (pre-fix): RSS 900-1250 MB each, 5000+ MB total. Heap profile showed 130k+ Multiaddr instances retained, growing at ~2300/min.

After fix: 132-292 MB per node, stable over 20+ min. No new Multiaddr accumulation.

## Root cause (heap retention path, 99.9% of samples)

```
GC root (synthetic)
  <- closure:Function (abort listener)
  <- JSLexicalEnvironment (pEventMultiple scope: items[], rejectHandler, onItem, cancel)
  <- Promise (fulfilled: Array(1))        via cancel edge
  <- Array(1)
  <- Object { id, multiaddrs }             (PeerInfo from PEER_RESPONSE)
  <- array                                  (multiaddrs field)
  <- Multiaddr x N
```

`QueryManager.run()` in `packages/kad-dht/src/query/manager.ts:148` calls:
```ts
await pEvent(this.routingTable, 'peer:add', { signal, filter })
```

Inside `p-event`, `pEventMultiple` registers:
```js
signal.addEventListener('abort', () => { rejectHandler(signal.reason) }, { once: true })
```

In some runtimes (observed in Bun/JSC), the `{ once: true }` abort listener is NOT automatically detached from the native `AbortSignal` after `resolve()` — the listener closure, together with its entire `pEventMultiple` lexical scope (containing `items[]` which references a `PeerInfo` object with its `multiaddrs` array), remains attached to the output signal of `anySignal()` for the lifetime of the signal.

`signal.clear()` in the `finally` block at line 186 only detaches listeners from **input** signals (shutDownController, queryEarlyExitController, options.signal), not from the output signal that `pEventMultiple` attached its listener to.

Every DHT query while the routing table is empty (which happens frequently during early bootstrap or when peers churn) leaks a `PeerInfo` + `Multiaddr[]` set. Over minutes, this grows to tens or hundreds of thousands of instances.

## Fix

Replace `pEvent` with explicit `addEventListener` + `removeEventListener` in a `finally`-equivalent cleanup path (idempotent, no reliance on runtime `{ once: true }` semantics). Also unwrap `pbStream` in `network.ts` to detach its stream listeners after write/read.

## Commits

1. **Replace pEvent with explicit listener management** (`manager.ts`)
2. **Unwrap pbStream after write/read** (`network.ts`)
3. **Check signal.aborted after listener registration to avoid race**

## Verification

- Heap profile sampling 5,000 Multiaddr instances: 99.9% retained via the pEvent closure path
- Deployed fix on 5 nodes (1 Docker, 1 systemd, 1 LXC, 1 Windows, 1 ARM64 Radxa)
- RSS stable within 132-292 MB per node, no unbounded growth over 20+ min
- All existing tests continue to pass

## Notes

- The fix intentionally does not change `p-event` — the retention is a runtime-specific bug in how `{ once: true }` listeners interact with native `AbortSignal`. Using explicit listener management in the DHT query path avoids the issue across runtimes.
- Alternative config-only workaround: `kadDHT({ allowQueryWithZeroPeers: true })` bypasses the leaky `pEvent` call entirely. This is what LiberShare ships today, but the upstream fix makes the workaround unnecessary.
