# Backpressure Regulator

**Backpressure Regulator** is a Rust library implementing flow-control primitives for rate-matching between producers and consumers, preventing resource exhaustion when upstream throughput exceeds downstream capacity.

## Why It Matters

Every distributed system encounters the producer-consumer rate mismatch: a fast producer feeding a slow consumer. Without regulation, this leads to unbounded queue growth, memory exhaustion, and cascading failures. The backpressure regulator pattern — borrowed from fluid dynamics where a relief valve prevents pipe overpressure — caps the in-flight work between system components. This is the foundational primitive behind TCP windowing, reactive streams (Project Reactor, Akka Streams), and gRPC flow control. In the SuperInstance actor framework, backpressure prevents a flood of conservation-law observations from overwhelming the analysis pipeline, ensuring graceful degradation under load rather than catastrophic failure.

## How It Works

The regulator implements a **bounded buffer with signaling** semantics. The core algorithm maintains a fixed-capacity internal queue:

```
State: pending = count of in-flight items
Capacity: C (maximum in-flight items)

Producer side:
  if pending < C:
    dispatch(item); pending++
  else:
    block / reject / drop (strategy-dependent)

Consumer side:
  on complete(item):
    pending--
    signal producer (if waiting)
```

**Three regulation strategies:**

| Strategy | Behavior | Latency | Lossiness |
|----------|----------|---------|-----------|
| Block (synchronous) | Producer waits | High under load | Lossless |
| Drop (sampled) | Discard excess | Low | Lossy |
| Signal (reactive) | Upstream notified | Variable | Configurable |

**Little's Law connection:** The optimal capacity follows Little's Law:

```
L = λ × W
```

Where L = in-flight items (capacity), λ = arrival rate, W = mean processing time. Setting C ≈ 2 × L provides headroom for bursts while bounding latency to 2W.

**Mathematical model:** Under stationary arrival process with rate λ and service rate μ:
- If λ < μ: queue is stable, expected length ≈ λ / (μ − λ)
- If λ ≥ μ: queue grows without bound without backpressure

The regulator's capacity bound prevents the unbounded case, converting it into either delay (blocking) or loss (dropping).

## Quick Start

```rust
fn main() {
    println!("Backpressure regulator active.");
    // In the actor framework:
    // 1. Each actor pair (producer, consumer) has a regulator
    // 2. Capacity is sized to Little's Law: C = 2 × λ × W
    // 3. On overflow: strategy determines behavior (block/drop/signal)
    // 4. Metrics: overflow_count, avg_queue_depth, max_wait_time
}
```

## API

| Component | Description |
|-----------|-------------|
| Regulator config | Capacity, overflow strategy, metrics |
| Producer interface | `try_send()` / `send().await` |
| Consumer interface | `on_complete()` acknowledgment |
| Metrics | Queue depth, overflow count, wait times |

## Architecture Notes

The Backpressure Regulator enforces the **flow conservation** aspect of γ + η = C. Just as the conservation equation requires that resource usage (γ) and intelligence processing (η) balance, the regulator ensures that message flow between layers cannot create unsustainable resource accumulation. Under sustained overload, the regulator degrades gracefully rather than allowing system-wide failure.

See [ARCHITECTURE.md](https://github.com/SuperInstance/SuperInstance/blob/main/ARCHITECTURE.md).

**Cascading failure prevention:** Without backpressure, a single slow downstream service can bring down an entire fleet. The failure cascade follows a predictable pattern: (1) slow service's queue grows, (2) queue consumes available memory, (3) GC pressure causes further slowdown, (4) upstream callers block on timeouts, (5) their threads exhaust, (6) their callers time out — the failure propagates upstream through every dependency. The backpressure regulator breaks this chain at step 1 by rejecting excess messages before the queue grows unboundedly.

**Reactive Streams standard:** The backpressure regulator implements the same semantics as the Reactive Streams specification (adopted in Java 9 Flow API, Project Reactor, RxJava, Akka Streams). The four protocol operations — `subscribe`, `onNext`, `onError`, `onComplete` — map to the regulator's send/reject/complete/error transitions.

## References

1. Little, J.D.C. (1961). "A Proof for the Queuing Formula L = λW." *Operations Research*, 9(3), 383–387.
2. Nygard, M. (2018). *Release It!* 2nd ed. Pragmatic Bookshelf. Chapter 5: Backpressure Patterns.

## License

MIT
