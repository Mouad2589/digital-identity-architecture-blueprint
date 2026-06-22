# ADR-0006 — CRDT-Based Sync Conflict Resolution

## Status

Accepted

## Context

In an offline-first system, multiple field devices capture events independently without connectivity. When they synchronise with the central system, conflicts can arise in two forms:

**Type 1 — Duplicate registration:** two devices independently register the same birth (e.g., the hospital maternity ward and a village health worker both register the same child). Result: two separate person records for the same individual in the Population Registry.

**Type 2 — Concurrent modification:** two devices update the same person record (e.g., address correction submitted from two different district offices simultaneously). Result: two versions of the same field with no causal relationship.

Several conflict resolution strategies exist:

**Option A — Last server arrival wins:** the server processes events in arrival order; whichever batch arrives last overwrites the previous state. Simple to implement.

**Option B — Last-write-wins with wall-clock timestamp:** events carry device wall-clock timestamps; the event with the highest timestamp wins. Robust to arrival order.

**Option C — CRDT + HLC + business rules + human review for ambiguous cases:** a layered strategy using data-type-appropriate merge semantics, Hybrid Logical Clocks for ordering, semantic business rules for known invariants, and human review as the final arbiter for genuinely ambiguous conflicts.

## Decision

**Option C is adopted.** The conflict resolution strategy uses four layers, applied in order:

### Layer 1 — Event log: Grow-Only Set (G-Set CRDT)

The event log is a **grow-only set**. Every captured event is appended permanently and propagated to all nodes. Events are never deleted. **No conflict is possible at the event level** — the event log always converges by union.

This layer is the source of truth. All other state is derived from the event log.

### Layer 2 — Mutable person attributes: Last-Write-Wins Register (LWW-Register) with HLC

Mutable person record attributes (address, contact, spelling corrections) use **Last-Write-Wins semantics** with Hybrid Logical Clock (HLC) timestamps as the comparison key.

**Why HLC over wall-clock timestamp (Option B):**

- Wall-clock timestamps are unreliable: NTP is unavailable in Tier 0; device clocks drift; two events on different devices can share the same millisecond.
- HLC timestamps combine wall-time proximity with monotonic causal ordering. An event with a higher HLC either happened after, or was concurrent but on a device with a higher UUID (deterministic tiebreaker).
- HLC provides a total order that is: (a) monotonically increasing per device, (b) causally correct across hops, and (c) close to physical time.

**LWW rule:** for any field F updated by two concurrent events E1 and E2, the value from the event with `max(HLC(E1), HLC(E2))` is applied. If `HLC(E1) == HLC(E2)` (identical tuple), see Layer 4.

### Layer 3 — Life status: Monotonic state machine

Life status transitions are **monotonic and irreversible**:

```
living → deceased   (irreversible)
unknown → living    (irreversible once confirmed)
unknown → deceased  (irreversible)
```

**Rule:** the `deceased` status, once set by any event in the log, is the final status, regardless of HLC ordering or arrival order. A `living` update that arrives after a `deceased` event (due to sync delay) does not revert the status.

This models reality: death is a fact, not a preference. No conflict resolution is needed — the monotonic rule applies without human input.

Correction events (spelling corrections, date corrections) always append a new `VitalEvent` with type `correction`. The corrected value is applied as the current value; the original event is retained permanently in the log. There is no overwrite.

### Layer 4 — Ambiguous concurrent edits: Multi-Value Register + supervisor review

If two events have identical HLC tuples `(wall_ms, logical, node_id)` — which is theoretically impossible given the node_id tiebreaker but defensively handled — or if the business rule engine determines that an automated merge would be semantically incorrect (e.g., two conflicting name corrections), the system uses a **Multi-Value Register**: both values are preserved, and the conflict is placed in the **district supervisor review queue**.

The supervisor:
- Reviews the conflicting values with full event context (who captured what, when, on which device).
- Selects the correct value or enters a new one.
- The decision is recorded as a new `correction` event with `resolvedBy: supervisor_id`.

**No silent merge is ever performed.** The system never automatically chooses between two plausible values without human confirmation.

### Duplicate detection (separate from conflict resolution)

Duplicate person records — created when two registrations refer to the same individual — are detected by the Population Deduplication Service, which runs asynchronously after sync. This is distinct from field-level conflict resolution.

Suspected duplicates are routed to the supervisor review queue for merge or dismissal. A merge:
- Selects the canonical record (typically the oldest, most complete, or eCVRS-assigned personId).
- Links the duplicate as an alias with a reference to the canonical record.
- Publishes a `PersonMerged` event to downstream systems so they can update their references.
- Retains both original registration records in the event log permanently.

### Conflict resolution summary table

| Scenario | Layer | Resolution |
|---|---|---|
| Same event re-delivered (same `event_id`) | Idempotency | No-op at server |
| Two events update the same field (different HLC) | Layer 2 — LWW-Register | Higher HLC wins |
| Two events update the same field (identical HLC) | Layer 4 — Multi-Value Register | Supervisor review queue |
| Life status: `deceased` event received after `living` update | Layer 3 — Monotonic | `deceased` always wins |
| Two independent registrations of the same individual | Deduplication | Supervisor merge |
| Correction vs. original value | Layer 1 — G-Set | Both retained; correction applied as current |

## Consequences

- **No silent data loss:** every event is stored permanently in the grow-only log. No event is ever discarded due to conflict resolution. Merge decisions are themselves auditable events.
- **Supervisor queue is a real operational cost:** the review queue requires staffed supervisors at district level. High registration volumes — particularly during campaigns — generate proportional review workload. Staffing and SLA for review must be planned.
- **HLC synchronisation is required:** devices must exchange HLC values on every sync. The sync protocol includes the sender's `max_seen_hlc` in the batch envelope header. The server updates its HLC and returns the server's `max_seen_hlc` in the response, ensuring all devices converge toward the same logical time.
- **Downstream monotonic consistency:** downstream systems (Identity Platform, Health, Social Protection) must be designed to accept monotonic state updates that arrive out of wall-clock order. They must not reject a `DeathRegistered` event because it arrives after a `PersonUpdated` event with a more recent wall-clock time.
- **Testing requirement:** CRDT merge logic must be exhaustively unit-tested with concurrent event scenarios. Property-based testing is recommended to explore edge cases: what is the merge result for every possible pair of event types on the same person record? Determinism must be verified — the same input events must always produce the same merged state regardless of processing order.
- **Algorithm replaceability:** the conflict resolution engine is behind an internal interface (`IConflictResolutionStrategy`). The LWW and monotonic rules can be adjusted per event type and per country-specific business rules without changing the event log or sync protocol.
