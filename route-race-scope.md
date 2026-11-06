# Route Race — Project Scope (Draft v0.1)

> Working title. A map-based puzzle game where you build itineraries across the **real, currently flown** airline network, scored against a solver.

---

## 1. One-line pitch

Pick a start and end airport. Get there using only real nonstop routes, under a rule that makes the obvious answer illegal (e.g. *"no eastward legs"*, *"one airline only"*, *"no hubs"*). A solver finds the best possible route; your score is how close you got.

**Example puzzle:** *Tokyo (NRT) → Mexico City (MEX), westward legs only.* You can't cross the Pacific, so you go the long way around: NRT → AMS → MEX (fewer legs) vs. NRT → DXB → BCN → MEX (more legs, maybe different total time). Which wins depends on the real network.

### Why this is worth building
- **Real data, no simulation:** the network is built from flights that actually happened.
- **Map-first and interactive:** exploring "where can I go from here?" is the core fun.
- **Genuine algorithms:** constrained shortest path, Pareto frontiers, puzzle generation.
- **Full stack with a data pipeline:** ingestion, inference, solver, API, web UI, leaderboard.
- **Shareable:** a daily puzzle and a result card people can compare.

---

## 2. Core gameplay

### 2.1 Loop
1. See the puzzle: origin, destination, active rule mode(s), and par.
2. Click the origin airport. Reachable airports light up; illegal ones are dimmed (with the reason on hover).
3. Click a destination to add a leg. Repeat until you reach the target.
4. Submit. See your time, legs, and score.
5. Reveal the solver's best route(s) next to yours, with a short "why" note (e.g. "the solver avoided the Dubai hub because the onward leg is thin").

### 2.2 Scoring (initial proposal, tunable)
- **Time mode:** `score = 100 × (optimal_total_time / your_total_time)`, capped at 100.
- **Legs mode:** `score = 100 × (optimal_legs / your_legs)`.
- **Balanced mode:** score against the Pareto frontier (see §6.3): full marks if you're on it, partial based on distance from it.
- Bonuses (optional): streaks, "found the unique optimum", "beat the obvious route".

### 2.3 Total time model
`total_time = Σ leg_block_time + Σ connection_time`
- `leg_block_time`: median observed gate-to-gate-ish duration for that airport pair (from the observed flights).
- `connection_time`: assumed minimum connection (tunable, e.g. 90 min domestic / 120 min international) plus an expected wait derived from frequency: `min(headway / 2, cap)` where `headway = 168h / flights_per_week`.
- **Be honest in the UI:** this is a *network* puzzle, not a timetable. We know how often a route flies, not which departure connects.

---

## 3. Rule modes (the thing that stops direct flights from being the answer)

### 3.1 Puzzle selection rules (always on)
- No nonstop exists between origin and destination in the active network.
- Optimal solution needs ≥2 legs (easy), ≥3 legs (hard).
- More than one *meaningfully different* good answer exists (see §7).
- Minimum frequency filter on every edge so rare/one-off routes aren't used.

### 3.2 Modes

| Mode | Rule | Notes |
|---|---|---|
| **Direction lock** | Every leg must head west (or east) | See §3.3. Signature mode. |
| **Airline lock** | All legs on one airline | Needs callsign → airline mapping. |
| **Alliance lock** | All legs within one alliance | Needs an alliance table (static, hand-maintained). |
| **Aircraft lock** | Every leg flown (with reasonable probability) by an allowed type set, e.g. 777 or 787 | See §3.4. Uses per-edge aircraft mix. |
| **No hubs** | Can't connect through top-N busiest airports | N scales with difficulty. |
| **Must pass through** | Route must touch a given country or airport | Creates "via" puzzles. |
| **Leg budget** | Max N legs | Pairs well with time scoring. |
| **Time cap** | Total time ≤ X | Find any route under the cap. |
| **Explorer** | Maximize distinct countries (or continents) in N legs | Not shortest-path; a different solver. |
| **Round-the-world** | Keep going one direction and return to the start | Variation of direction lock. |
| **Reverse puzzle** | Given a route, guess the origin/destination | Low-effort daily variant. |
| **Mixed** | Two modes at once (e.g. westward + no hubs) | Unlocks late; combinatorially rich. |

### 3.3 Direction lock — definition
For a leg from longitude λ1 to λ2 (degrees east):

```
Δλ = ((λ2 − λ1 + 540) mod 360) − 180      # signed shortest-arc longitude change
eastward  if Δλ >  +ε
westward  if Δλ <  −ε
neutral   otherwise (|Δλ| ≤ ε), allowed in either mode
```

- `ε` is a tolerance (start with ~3°) so near-north/south legs aren't punished.
- Polar routes can have odd longitude behavior; we use the shortest-arc definition and accept edge cases, but they need playtesting.
- **Confirmed:** the signature rule is **"no eastward legs"** (westward-only). The mode supports either direction.

### 3.4 Aircraft lock — definition
Rule: every leg must be flown, with reasonable likelihood, by an **allowed aircraft family or set of families**.

Airlines swap equipment, so an edge doesn't have one aircraft type; it has a **mix** measured from observed flights, e.g.:

```
aircraft_mix = { B77W: 0.55, B788: 0.30, A333: 0.15 }
```

A leg is eligible if the combined share of allowed types meets a threshold:

```
eligible  if  Σ share(t) for t in allowed_set  ≥  τ
```

- The **allowed set can contain several families** — e.g. `{777, 787}`. A route dominated by 777s and 787s qualifies even though neither type alone is dominant.
- `τ` is a puzzle parameter shown to the player (tunable):
  - **Strict (τ ≈ 0.70):** "usually flown with".
  - **Standard (τ ≈ 0.40):** "commonly flown with".
  - **Loose (τ ≈ 0.15):** "sometimes flown with" — more edges, more risk.
- **Family groupings** map ICAO type codes to families, e.g. 777 = `B772/B773/B77L/B77W`, 787 = `B788/B789/B78X`, A350 = `A359/A35K`, plus A380, 747, 737, A320-family, A220, E-jets, and turboprops (ATR/Dash 8).
- **Preset locks:** *777 or 787*, *Widebody only*, *Quads only* (A380/747/A340), *Airbus only*, *Boeing only*, *Newest types* (A350/787/A220), *Turboprop challenge*.
- **Combines with airline lock.** For "Emirates, 777/A380 only", the mix is computed per `(edge, airline)`, so the builder must store the mix at that granularity.

**Optional "reliability" mechanic:** show each leg's allowed-share as a confidence bar, and a route-level confidence `Π share` across legs (assumes independence between legs — a simplification). Scoring can optionally reward higher confidence among equally fast routes.

**Data requirements:** aircraft type per flight needs a join from the transponder address (icao24) to an aircraft registry. OpenSky publishes aircraft metadata, but **verify coverage, currency, and accuracy** (new registrations and re-registrations will be missing or wrong). Flights with unknown type count as "unknown" and never as an allowed type; report the unknown rate as a data-quality metric.

**Generator implications:** aircraft locks shrink the graph a lot — the widebody network is sparse and hub-heavy. The generator must test feasibility on the *filtered* graph and will likely need regional limits for the strictest presets. Extra mode: **Aircraft Collector** — fly as many distinct types (or families) as possible within N legs.

---

## 4. Data

### 4.1 Sources
| Need | Source | Notes |
|---|---|---|
| Airports (coords, codes, country) | **OurAirports** (public domain, community-maintained) | IATA/ICAO, lat/lon, type, country. |
| Observed flights (origin, dest, times, callsign, aircraft) | **OpenSky Network** | Flight list with estimated departure/arrival airports; also historical access via their database. |
| Airline identification | Callsign ICAO prefix (first 3 letters) + a curated airline table | OpenFlights `airlines.dat` is usable as a *code lookup*, not as a route source. |
| Validation (US-touching routes) | **BTS T-100** | Official nonstop segment data; use to measure precision/recall of inferred edges. |
| Alliance membership | Hand-maintained table | Small, changes rarely. |
| Aircraft type per flight | OpenSky aircraft metadata joined on transponder address (icao24) | Needed for aircraft lock; verify coverage and accuracy (§3.4). |

**Do not use OpenFlights `routes.dat` as the live network.** Its route source stopped updating in 2014. It is fine for prototyping the solver before the real pipeline exists.

### 4.2 OpenSky notes (verify before building)
- Flight records include **estimated** departure and arrival airport, which can be missing. Expect a chunk of unusable records.
- Free accounts have a daily credit limit, and airport-based queries have a maximum time window per request. Bulk history may be better via their historical database — **check access requirements and terms**.
- Terms are **research / non-commercial**. Fine for a personal project; revisit before any monetization.
- Coverage depends on community receivers. **Expect uneven coverage by region** — measuring this is the first technical task.

### 4.3 Inferring the network
An airport pair (A→B) becomes a **nonstop edge** if, over a trailing window (start with 90 days):
1. ≥ `MIN_FLIGHTS` observed flights from A to B (start with ~12, i.e. roughly weekly), **and**
2. Flights are operated by airlines on a **passenger-airline allowlist**, **and**
3. Aircraft type is plausible for passenger service (exclude freighters/GA if aircraft metadata allows).

Edge attributes:
- `median_block_minutes`, `p25/p75` (for spread)
- `flights_per_week` (from counts)
- `airlines` (set of operators with counts)
- `aircraft_mix` (share of flights by ICAO type, overall **and per airline**; includes an `unknown` share)
- `last_seen`, `first_seen_in_window`

**Known contamination sources:** cargo, charters, positioning/ferry flights, repositioning for maintenance, GA/business jets, military. Mitigation: airline allowlist, minimum frequency, aircraft-type filter, and the T-100 cross-check.

### 4.4 Data quality metrics (publish these in the README)
- **Edge precision/recall vs T-100** for US-touching routes.
- **Coverage by region:** share of expected major hubs with plausible edge counts.
- **Spot-check list:** a hand-curated set of ~50 known routes (present?) and ~20 known non-routes (absent?).
- **Stability:** how much the edge set changes between consecutive windows (should be small).

---

## 5. Architecture

```
OpenSky ──► ingest jobs ──► raw flights (Parquet)
                                  │
OurAirports, airline tables ──────┤
                                  ▼
                        network builder (DuckDB/Python)
                                  │
                       edges.parquet + airports.parquet
                                  │
              ┌───────────────────┴───────────────────┐
              ▼                                       ▼
      puzzle generator (offline)               solver library
              │                                       │
              └──────────────► Postgres ◄─────────────┘
                                  │
                              FastAPI
                                  │
                         Web app (Next.js + MapLibre)
```

### 5.1 Components
- **Ingestion:** scheduled Python jobs; idempotent; stores raw responses so the network can be rebuilt without re-fetching.
- **Network builder:** DuckDB SQL over Parquet; versioned snapshots (`network_2026-11-01`) so puzzles stay reproducible.
- **Solver library:** pure Python (later optionally Rust) with unit tests; same code used by the generator and the API.
- **Puzzle generator:** offline batch job producing a pool of vetted puzzles per mode and difficulty.
- **API:** puzzle fetch, move validation, submission, reveal, leaderboard.
- **Frontend:** MapLibre GL + deck.gl arcs for routes; airport hover shows frequency/airlines/typical time.

### 5.2 Why snapshots matter
The network changes over time. Every puzzle records the network version it was generated from, so the "optimal" answer never silently changes after the fact.

---

## 6. Solver

### 6.1 Graph model
- Nodes: airports. Directed edges: observed nonstop routes with weights from §2.3.
- Connection cost is added when switching flights at a node.
- Edge filters per mode (direction, airline, alliance, hub exclusions) are applied *before* search.

### 6.2 Algorithms
- **Basic modes:** Dijkstra on time; BFS for fewest legs.
- **Constrained modes:** search over an expanded state `(airport, extra_state)` where `extra_state` can be the airline used (airline lock), a "visited required node" flag (must pass through), or the set of countries visited (explorer; may need a bounded DFS / beam search rather than exact search).
- **Round-the-world:** track cumulative signed longitude; a valid itinerary returns to the start after accumulating ≈ ±360°.

### 6.3 Pareto frontier (time vs legs)
For balanced mode, compute all non-dominated `(total_time, legs)` itineraries. This is also how we measure puzzle quality (§7). A label-setting multi-criteria search handles this at our graph size.

---

## 7. Puzzle generation and difficulty

### 7.1 Candidate pairs
Sample origin/destination pairs, then keep those satisfying §3.1.

### 7.2 Quality metrics
| Metric | Why it matters |
|---|---|
| Pareto frontier size ≥ 2 | There is a real tradeoff, not one obvious answer. |
| Gap between "greedy/obvious" route and optimal | Obvious = route through the highest-degree hub. A gap means skill matters. |
| Number of distinct near-optimal routes (within 10%) | Too many = trivial; one = brittle. Target a small number. |
| Edge frequency along the optimal path | Avoid puzzles whose answer hinges on a once-a-week route. |
| Region diversity across the daily pool | Don't serve only Europe↔US pairs. |

### 7.3 Difficulty tiers
- **Easy:** 2 legs optimal, loose rules, large hubs.
- **Medium:** 3 legs, one rule mode, mix of hubs and mid-size airports.
- **Hard:** 3–5 legs, two modes, thin spokes, hub traps.

### 7.4 Daily schedule
One featured puzzle per day (seeded), plus practice mode with generated puzzles on demand.

---

## 8. UX notes

- **Map:** dark base map; reachable airports glow; illegal ones dim with a tooltip reason ("eastbound — not allowed").
- **Airport tooltip:** code, city, flights/week out, top airlines.
- **Route builder panel:** running totals for time and legs, with undo.
- **Reveal screen:** animated comparison of your route vs. the solver's, with short auto-generated notes (e.g. "solver avoided DXB: onward leg to BCN has only 5 flights/week").
- **Share card:** route drawn on a mini map, score, mode — no spoilers in the text.
- **Accessibility:** keyboard-searchable airport picker as an alternative to clicking the map.

---

## 9. Feature backlog / more ideas

**Gameplay**
- Weekly "hard mode" challenge with a global leaderboard.
- Multiplayer race: same puzzle, first valid route within par wins.
- "Ghost" replays of top players' routes.
- Custom puzzles: pick any pair and rules, share a link.
- Achievements: first round-the-world, ten countries in one itinerary, beat par on a hard puzzle.

**Data-driven extras**
- "Network health" page: airports ranked by how many puzzles they appear in, regional coverage, and changes since last snapshot.
- Route trivia from the data: thinnest widebody-like route, longest observed nonstop in the window, most connected non-hub.
- Time-travel mode: replay the same puzzle against an older network snapshot.

**Platform**
- Public API for the network snapshot.
- Embeddable "daily puzzle" widget.

---

## 10. Milestones

Each milestone has an exit criterion. If one fails, stop and rethink instead of building on top of it.

| # | Milestone | Exit criterion |
|---|---|---|
| M0 | **Coverage check** — pull a few weeks of OpenSky flights; compute edges for a set of known routes across regions | Known routes present in ≥ ~85% of test cases in your target regions; if not, shrink the region scope |
| M1 | **Network builder** — edges with block time and frequency; validate vs T-100 (US) | Reasonable precision/recall; spot-check list passes |
| M2 | **Solver** — Dijkstra + direction lock + tests | NRT→MEX westward gives sensible 2–3 leg answers |
| M3 | **Puzzle generator** — filters and quality metrics | A pool of puzzles that feel varied when you play ~20 by hand |
| M4 | **Map UI (single-player)** — build routes, submit, reveal | Playable end to end locally |
| M5 | **Backend + daily puzzle + leaderboard** | Deployed; daily puzzle rotates |
| M6 | **More modes + polish + share card** | Airline lock, no hubs, must-pass-through shipped |

Rough effort for a solo builder at weekend pace: M0–M2 in 1–2 weekends, M3–M4 in 2 weekends, M5–M6 in 2–3 weekends.

---

## 11. Risks and open questions

| Risk | Mitigation |
|---|---|
| OpenSky coverage gaps in some regions | Measure early (M0); launch with well-covered regions; widen later |
| Missing/incorrect estimated airports in flight records | Filter low-confidence records; require repeated observations |
| Cargo/charter contamination of the passenger network | Airline allowlist; frequency threshold; aircraft-type filter; T-100 check |
| Connection feasibility unknown (no timetable) | Use estimated waits; disclose clearly; consider "minimum frequency" rules |
| API credit / rate limits for bulk history | Cache raw data; consider the bulk historical database route; check terms |
| Non-commercial terms | Keep it a personal/portfolio project; revisit before monetizing |
| Puzzle pool feels repetitive (everything via the same hubs) | Quality metrics in §7; "no hubs" and "must pass through" modes |
| Polar/dateline weirdness in the direction rule | Playtest; add tolerance; manual exceptions list |
| Aircraft type data missing or wrong for some flights | Count unknowns separately; never assume an allowed type; publish the unknown rate; spot-check known routes |
| Aircraft-lock graphs too sparse to make puzzles | Feasibility check on the filtered graph; looser `τ` or regional limits for strict presets |

### Open decisions
1. ~~Direction rule~~ — **decided:** "no eastward" is the signature mode; the mode also supports the opposite direction. **Aircraft lock** is also in scope (§3.4), with multi-type sets like 777 + 787.
2. ~~Region scope for v1~~ — **decided: whole world.** Because OpenSky coverage is uneven, global scope is handled with **coverage tiers**:
   - Every airport gets a coverage score from M0 (how many known routes the data recovers around it, and how many flights are observed with a usable estimated airport).
   - **Tier A** (well covered): eligible for puzzles at launch. **Tier B** (partial): appears on the map and can be used mid-route, but is never a puzzle origin or destination. **Tier C** (sparse): shown greyed out as "limited data", never in puzzles.
   - Tiers are recomputed with each network snapshot, so regions graduate as data improves (or as more data sources are added).
   - The coverage map is a public page. Being open about gaps is better than silently serving wrong answers, and it is a good portfolio artifact in itself.
   - If M0 shows an entire region is too thin to use (for example, most of a continent), the fallback is to launch with that region as Tier B/C and keep the rest of the world live. This is still global, just honest about gaps.
3. **Solo project or public game?** (affects terms of use, hosting, and how much polish matters)
4. **Single-player first, or design for multiplayer from day one?**
5. **Name.** Candidates: *Route Race*, *Long Way Round*, *Hub & Spoke*, *Layover*, *Westbound*.

---

## 12. Suggested first week

1. Create an OpenSky account; read the current terms and access options.
2. Pull ~2–4 weeks of flight records for a handful of major airports in different regions.
3. Compute edge counts for ~50 routes you know exist and ~20 you know don't; see what the data says.
4. Load OurAirports; write the direction-rule function with unit tests (NRT→MEX, NRT→AMS, AMS→MEX, edge cases).
5. Hand-build a tiny graph (10–20 airports) and run Dijkstra with the direction filter end to end.

If steps 3 and 5 look good, the project is viable; everything else is engineering.
