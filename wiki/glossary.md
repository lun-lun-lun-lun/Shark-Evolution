# Glossary

Use these terms exactly, in code names and in docs. Add new terms here when you introduce them.

| Term | Meaning |
|---|---|
| Calm school (tier A) | A group of fish whose positions are a closed-form function of the school record, each fish's index and server time. Never stepped. |
| Reacting fish (tier B) | A fish promoted out of its school (fleeing, chased) and stepped individually on the server at about 10 Hz. |
| AI predator (tier C) | A server-controlled predator: a jecs entity that decides at 2–5 Hz and steers at 10–20 Hz. |
| School record | id, species, seed, spawn time, path parameters and alive bitmask. Sent once when the school becomes relevant to a player. |
| Promotion | Moving a fish from tier A to tier B. |
| Scatter | A reliable event (school id, start time, origin, seed) after which the server and clients evaluate the same closed-form burst. |
| Record | The 10-byte quantized state of one tier-B or tier-C entity in the state stream. |
| State stream | The unreliable, sequence-numbered buffer packets that carry records. |
| Server time | `workspace:GetServerTimeNow()`. Every timestamp on the wire uses it. |
| Render time (`renderServerTime`) | The server time at which a client drew a target. Sent with a bite claim. |
| Bite claim | Client-to-server message: "I bit target X at render time T". |
| Rewind | Evaluating a target's position at a past server time to validate a claim. |
| Rewind clamp | The maximum rewind allowed for one claim, set per player and per tier. |
| Interest (relevancy) | The set of grid cells, and the entities in them, that a player receives. |
| Cell | One cell of the interest grid (128–256 studs across). |
| Hysteresis band | The extra distance (20–40 studs) an entity must move past the interest radius before it stops being relevant. Prevents flicker at the edge. |
| Priority accumulator | Per-player priority that grows every tick until something is sent, so nothing starves (Gaffer On Games). |
| Interpolation buffer | How far behind server time a client draws streamed entities: about 3 send intervals. |
| Error offset | A visual offset that hides a correction (for example at promotion) and decays every frame. |
| Leaky bucket | A budget that refills at a fixed rate and is spent by actions; used for speed checks and eats per second. |
| Pull-back | A server correction that returns a client-owned shark to its last valid position. |
| RTT | Round-trip time between a client and the server. |
| Data ping | Latency including queueing. It rises above network ping when a link is saturated. |
| p95, p99 | The 95th and 99th percentile of a measurement, such as frame time. Budgets are set on p95. |
| Dry ocean | The world has no Terrain water; the underwater look and swimming are our own. |
| Working title | "Shark Evolution". Not the name the game ships under ([D14](decisions.md#d14-name-and-originality)). |
