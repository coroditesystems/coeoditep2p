---
name: Add explicit Shutdown event and transition
about: Make NodeState::Shutdown reachable via explicit event
title: "Add explicit Shutdown event and transition"
labels: ["machine-model", "state-semantics", "event-driven"]
assignees: []

---

## Problem

`NodeState::Shutdown` exists in the state enum (line 407-412 of coroditep2p.rs) but there is no `Event::Shutdown` to trigger this transition. The state exists as a sink, but no explicit causal path reaches it.

Current state flow:
- `Init` → (Tick) → `Running`
- `Running` → (error) → `Trapped(trap)`
- `Trapped(trap)` → (no recovery path defined)
- `Shutdown` → (unreachable via event)

## Why this matters

1. **Causal completeness**: EVENT → TRANSITION → STATE requires explicit events for each reachable state.
2. **Machine observability**: Without an explicit shutdown event, observers cannot distinguish between "machine still running" and "machine has shut down."
3. **Determinism**: A hidden or implicit shutdown violates the deterministic state principle.
4. **Trap recovery**: Unclear whether Trapped state should transition to Shutdown, remain indefinitely, or recover to Running.

## Expected direction

1. Add `Event::Shutdown { reason: u8 }` to the Event enum
2. Add case in `handle_trapped()` and/or `handle_running()` to accept this event
3. Document whether:
   - Only certain traps allow shutdown (e.g., after N recovery attempts fail)
   - Shutdown is immediate or deferred
   - Shutdown effect must be signaled to the application layer
4. Clarify final effect after transition: what cleanup/logging occurs?

## Acceptance criteria

- [ ] `Event::Shutdown` defined and reachable
- [ ] Transition rule documented: which state(s) accept Shutdown?
- [ ] `handle_trapped()` processes Shutdown (recovery condition defined)
- [ ] Test case: Trapped state → Shutdown event → Shutdown state
- [ ] No dead code: Shutdown state is reachable via at least one event path
