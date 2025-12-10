# BitChat Performance Assessment v1.0 (2025-12-10)

## Scope and methodology
- Reviewed published app capabilities and transport design documented in `README.md` and existing technical configuration files.
- Surveyed core runtime services (routing, deduplication, transport tuning) to identify performance-critical parameters and behaviors.
- Focused on user-facing chat flows across Bluetooth mesh and Nostr transports, as well as the messaging pipeline.

## User-visible functionality (performance-relevant)
- Dual transport architecture: Bluetooth mesh for offline reach and Nostr for internet messaging, automatically selecting the best path per message type.
- Location-based public channels via geohash precision levels, enabling scoped timelines for block, neighborhood, city, province, and regional conversations.
- Private messaging with Noise-encrypted sessions on mesh and NIP-17 encrypted payloads on Nostr, including intelligent queuing and delivery acknowledgments.
- Adaptive power and bandwidth optimizations such as LZ4 compression, battery modes, and optimized networking defaults.

## Core architecture touchpoints
- Transport tuning is centralized in `TransportConfig` with parameters covering BLE fragment sizing, TTLs, concurrency caps, adaptive duty cycles, and geohash fetch limits, providing a single source of truth for runtime knobs.
- `MessageRouter` selects the first reachable transport per peer and maintains an in-memory outbox keyed by `PeerID`, flushing queued messages when reachability updates occur.
- `MessageDeduplicationService` uses generic LRU caches for content- and Nostr-event deduplication with hashing-based normalization to suppress near-duplicate payloads.

## Findings and opportunities
1. **Adaptive transport tuning and safeguards**
   - The mesh defaults (e.g., TTL=7, max concurrent transfers=2, fragment relay jitter) and duty-cycle timers are static. Introduce runtime telemetry (success rates, hop counts, congestion) to auto-tune TTL and pacing, and to clamp aggressive parameters when failure/timeout rates rise.
   - Consider environment-aware profiles (dense vs sparse) that adjust `bleDutyOnDuration`, `bleDutyOffDuration`, and relay jitter based on observed peer density and RSSI variance.

2. **Outbox resilience and prioritization**
   - The current in-memory outbox is cleared if the process restarts. Persist queued private messages (e.g., background task + file-backed queue) with priority ordering so critical receipts and user-initiated DMs survive app restarts or transport flaps.
   - Add metrics for queue growth/age and alerts when messages exceed expected delivery windows to prevent silent drops.

3. **Deduplication efficacy and memory management**
   - LRU caches use fixed capacities and prefix-based hashing. Add time-window eviction and lightweight bloom-filter style pre-checks to reduce memory churn under bursty relay traffic while keeping false-positive dedup low.
   - Capture per-channel dedup hit/miss ratios and normalize configuration (e.g., `contentLRUCap`, processed Nostr event caps) based on real-world relay volumes.

4. **Delivery observability and retries**
   - Surface structured metrics for routing decisions (transport chosen, reachability failures, retry counts) and link them to UI indicators. Use backoff strategies for read/delivery receipts when no transport is reachable instead of silent no-ops.
   - Incorporate periodic flush scheduling for queued messages beyond event-driven notifications to reduce tail latency in sparse networks.

5. **Release engineering for performance**
   - Expand CI coverage to run Swift build/tests on all branches and add automated release notes generation so performance fixes ship with traceable changelog entries.
   - Include performance benchmarks or micro-bench targets (where available) in CI to catch regressions in message formatting, dedup hashing, and fragment handling.
