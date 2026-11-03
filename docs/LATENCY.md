# Latency Analysis

The mirror path was profiled hop-by-hop and optimised until the only remaining
cost is the broker's own order-placement API — the physical floor.

## Critical path

```
Broker fill
  │  real-time (order-update WebSocket)
  ▼
Master: parse → update net → sign → forward
  │  ~15 ms   (pre-warmed keep-alive TLS, TCP_NODELAY)
  ▼
Network: master → follower
  │  ~7 ms    (persistent connection, private tunnel)
  ▼
Follower: verify sig → convergence
  │  microseconds   (in-memory net cache; exits skip margin)
  ▼
Broker: place_order
  │  ~50–150 ms   ◀── IRREDUCIBLE: the broker accepting the order
  ▼
Operator order live
```

**End-to-end: ~75–175 ms**, dominated by `place_order`.

## What each optimisation removed

| Optimisation | Latency removed |
|--------------|-----------------|
| Fix dead order-update WebSocket | Removed the fallback to ~1 s polling — fills now arrive in real time |
| Pre-warmed keep-alive TLS + `TCP_NODELAY` | Removed per-event TLS handshakes from the forward |
| **In-memory net cache** | Removed a 100–2500 ms REST position read from **every** event |
| **Margin bypass on exits** | Removed a ~100–200 ms REST margin call from every exit |
| Tightened retry backoff | A transient retry costs ~50 ms instead of 300 ms+ (success path never sleeps) |

## Why `place_order` is the floor

The operator order cannot exist until the broker's matching/risk systems accept
it and return an order id. Nothing in the mirror can place the order before that
round-trip completes. The only way below it is a broker-side low-latency/colocated
order channel — infrastructure outside this codebase.

## Redundancy vs. latency

The three reliability layers run at different cadences so that speed and safety
don't trade off:

| Layer | Cadence | Role |
|-------|---------|------|
| WebSocket fast path | real-time | primary — milliseconds |
| REST poll | 0.3 s | catches a WebSocket blip |
| Reconciler | ~3 s | guarantees eventual convergence |

In the common case the fast path wins and the slower layers simply observe
"in sync." They only place an order when the fast path has genuinely missed one.
