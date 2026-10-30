---
title: Eating and bites
status: planned
code: []
decisions: [D7]
related: [fish-simulation, networking, shark-movement, persistence]
last_verified: n/a (no code yet)
---

# Eating and bites

## Summary

Clients detect bites and show them immediately; the server decides whether they happened. A bite is a claim, "I bit target X at server time T", which the server checks against where X was at T, using a rewind window clamped per player and per target tier. This is the one place the game needs exact authority, and it works the same whether movement is client-owned or, later, server-authoritative.

## Spec

### Flow [decided]

1. **Client:** the shark's mouth overlaps a target. Play the chomp and hide the target straight away.
2. **Client → server:** send `BiteClaim {targetId, renderServerTime}` through the rate-limited Blink event. `renderServerTime` is the server time at which the target was drawn: the client's estimated server time for tier A, or the interpolation time for tiers B and C.
3. **Server, cheap checks first, in this order:**
   1. The target is alive.
   2. The shark is big enough to eat it.
   3. The bite cooldown has passed.
   4. The eats-per-second leaky bucket has room. This bucket also caps how fast anyone can grow.

   The first valid claim on a target wins.
4. **Clamp** the claimed time (next section).
5. **Find the target's position** at the clamped time: the closed-form function for tier A, or the 10–20 Hz position ring buffer, interpolated between samples, for tiers B and C.
6. **Accept** if that position is within bite range plus a tolerance and inside the mouth cone, measured from the biting shark's latest replicated position and orientation on the server [proposed].
7. **Accepted:** apply the eat on the server (growth and so on), remove the target, and send `Eaten` to every client that can see it. **Rejected:** send `BiteRejected` with a reason to the biter, whose client shows the target again.

### Rewind clamps

| Target | When the biter saw it | Maximum rewind |
|---|---|---|
| Tier-A fish | At estimated server time | About the client-to-server half of the round trip: 75–150 ms at 150–300 ms RTT [target] |
| Tier-B fish, tier-C predators | About one round trip plus the interpolation buffer in the past | This player's measured RTT + the tier's buffer + a margin, with a ceiling near 500 ms [target] |
| Other players (PvP) | — | A ceiling of about 150–250 ms that favours the victim, plus a check that the victim is still near the attacker at the current server time [target] |

Why the interpolated case needs about 450–500 ms: a snapshot is about RTT/2 old when it arrives, the client draws it one buffer (B) later, and the claim takes another RTT/2 to reach the server. So the rewind is about RTT + B. At 300 ms RTT with a 150 ms buffer (20 Hz), that's about 450 ms, plus a margin.

The clamp is per player so that low-ping players can't claim stale bites. Beyond the PvP ceiling, high-ping attackers have to lead their target. Generous mouth radii and a short, telegraphed lunge do more for PvP fairness than a longer rewind.

### Logging [decided]

For every bite, record the rewind used, the player's RTT, the target tier, the result and any rejection reason, and tune the clamps from real mobile traffic. Aggregate before sending analytics; never send one event per fish ([persistence](persistence.md)).

## Invariants

- The server never applies an eat it hasn't validated.
- A target is eaten at most once: the first valid claim wins.
- No rewind exceeds the tier's ceiling or the player's own clamp.

## Tests

None yet. Planned Lune specs:

- Claims at a range of RTTs against closed-form and ring-buffer targets are accepted inside the clamp and range, and rejected outside them.
- Two claims on the same target: exactly one is accepted.
- The eats-per-second bucket rejects a burst and recovers at its refill rate.

## Open questions

- Bite range, tolerance, mouth cone angle, cooldown and the eats-per-second rate. These depend on shark sizes, which aren't designed.
- What "big enough to eat it" means (the food chain isn't designed).
- How each player's RTT is measured.
- Whether the claim should also carry the attacker's own position, checked against the replicated one within a tolerance.
- The PvP outcome: damage or an instant eat. Not designed.

## History

- 2026-10-30 — Created with the planned design from the tech stack research.
