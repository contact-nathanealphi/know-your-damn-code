# Guru Curriculum Engine

Load this file for any curriculum command, session start with existing state, `roadmap`, `checkpoint`, `revisit`, or when generating curriculum-driven `Next:` recommendations.

**Contents:** 1. Concept Graph | 2. Learner State | 3. Discrete Mastery & Evidence | 4. Spaced Repetition | 5. Next-Topic Engine | 6. Gap Radar | 7. Goals | 8. Commands | 9. Persistence | 10. Guardrails

---

## 1. Concept Graph

A directed acyclic graph (DAG) of **concepts** (not raw files), built by `roadmap` or `curriculum init`.

```json
{
  "v": 2,
  "scanned_at": "<git sha or YYYY-MM-DD>",
  "inferred": true,
  "nodes": [
    {
      "id": "async-await",
      "name": "Async/Await",
      "tier": 2,
      "prereq": ["promises"],
      "files": ["src/api/client.js"],
      "priority": "high",
      "effort": "S"
    }
  ]
}
```

Rules:

* Aim for 15-40 nodes for a typical repository, <=3 prerequisites per node, and zero cycles.
* `tier` 1-4 matches the 4 roadmap tiers. `priority` is `high`, `medium`, or `low` based on how central the concept is to the project. `effort` is `S`, `M`, or `L`.
* Language foundation concepts (e.g., `promises`) may have `files: []`.
* Use kebab-case IDs and keep them **stable across rescans** so saved learner progress survives. Reuse existing IDs, add new ones, and mark removed concepts `"retired": true`.
* Mark newly scanned graphs `"inferred": true` and clearly present them as an estimate until the user confirms or adjusts them.

## 2. Learner State

```json
{
  "v": 2,
  "goal": "extend",
  "level": "standard",
  "render": "markdown",
  "autosave": false,
  "sessions": 4,
  "snooze": null,
  "known": [],
  "concepts": {
    "async-await": {
      "level": 4,
      "status": "mastered",
      "ev": ["quiz", "transfer"],
      "miss": [],
      "seen": "2026-11-23",
      "next": "2026-12-14",
      "streak": 2,
      "flags": []
    }
  }
}
```

* `level` (0-4) and `status`: `0 = unseen`, `1 = introduced`, `2 = practiced`, `3 = solid`, `4 = mastered`.
* `flags`: optional tags such as `decaying` (overdue review) or `stale` (underlying files changed in git).
* `ev`: distinct evidence types collected (`quiz`, `practice`, `teachback`, `interview`, `transfer`, `checkpoint`).
* `miss`: short phrases recording unresolved misconceptions.
* Any concept not present in `concepts` is `unseen` (`level: 0`). **Never invent progress.**

## 3. Discrete Mastery & Evidence Rules

Use deterministic level transitions instead of fragile decimal math:

| Level | Status | How to Reach It |
| --- | --- | --- |
| 0 | `unseen` | Not yet studied |
| 1 | `introduced` | Explained in a walkthrough, or user self-reports ("I already know this") without verification |
| 2 | `practiced` | User completes 1 verified check (`quiz`, `practice` with hints, `teachback`, or `interview`) |
| 3 | `solid` | User passes an unassisted `practice`, `teachback`, `checkpoint`, or `interview` with strong reasoning (at least 2 distinct evidence types) |
| 4 | `mastered` | Currently `solid` PLUS at least 1 verified `transfer` (correctly applying or explaining the concept in a *different* file or context) with zero open `miss` items |

Additional rules:

* Self-report alone never exceeds Level 1 (`introduced`).
* A wrong answer that reveals a misconception adds a short phrase to `miss`, tags the matching `⚠️` note, and keeps the concept at or lowers it by 1 level (minimum `1`).
* Resolving a misconception in a later check removes it from `miss`.

## 4. Spaced Repetition

Fixed review schedule based on successful review `streak`:

| Streak | Next Review |
| --- | --- |
| 0 | +1 day |
| 1 | +3 days |
| 2 | +7 days |
| 3 | +21 days |
| 4+ | +60 days |

* **Pass:** Increment `streak` by 1, remove `decaying` flag, and set `next` using the table above.
* **Fail:** Reset `streak` to 0, set `next` to `+1 day`, and record the misconception in `miss`.
* **Overdue decay:** If `today` is past `next` by more than 1 full interval, add the `decaying` flag and drop `level` by 1 (floor `2`).
* Draw `revisit` drills from `🔑`/`⚠️` notes and past `miss` entries (3-5 quick items per session).

## 5. Next-Topic Engine

A concept is **ready** if all of its prerequisites are at least Level 3 (`solid`). Recommend topics in strict priority order:

1. **Reviews due:** Concepts flagged `decaying`, `stale`, or with `next <= today`.
2. **Open misconceptions:** Concepts with non-empty `miss`.
3. **Blocking gaps:** Ready, unlearned concepts that are prerequisites for the most downstream nodes.
4. **Continuity:** Direct child or same-file neighbor of the concept just studied.
5. **Goal-aligned branch:** Unlearned concepts matching the active `--goal` focus areas (see §7), breaking ties by `priority` (`high` > `medium` > `low`).

Limits: Introduce at most 3 new concepts per session. Always format output as **1 primary + 1 alternative + concise reasons** (translated to the user's language):

```text
Completed:   async-await (Level 4 - mastered)
Recommended: error-handling (Level 2) - review due + 1 open misconception
Alternative: api-calls (Level 0) - prerequisites met, direct next step
High-impact untouched: retry-logic, auth-flow
```

## 6. Gap Radar (Non-Intrusive Reminders)

* **Start/end of session:** Show at most 1 compact reminder block (<=3 lines) listing the highest-impact untouched concepts. Respect `snooze` if set.
* **Prerequisite skip detection:** If the user asks about concept X while prerequisite Y is below Level 2 (`practiced`), warn once per `(X, Y)` pair per session: *"Note: X builds on Y (not yet practiced)."* Provide a <=5-line micro-bridge for Y, then answer X directly. Never block the user.
* **Stagnation check:** If a concept stays at the same level across 3 sessions, suggest changing tactics (smaller chunks, a fresh analogy, `--socratic`, or a `practice` drill).
* **Stale code detection:** When `git` is available, compare `git diff --stat <scanned_at>..HEAD`. Flag nodes whose mapped files changed as `stale` and queue them for a quick refresher.

## 7. Goals & Adaptive Pathing

| Goal | Focus Areas Prioritized First |
| --- | --- |
| `maintain` | Recently changed files, core flows, debugging, error paths |
| `extend` | Data models, extension points, module boundaries, conventions |
| `interview` | Architecture, trade-offs, security, scalability, design rationale |
| `audit` | Security boundaries, validation, error handling, tests, dependencies |
| `rebuild` | Foundations, build order, Tier 1-2 core mechanics |

Skip `mastered` concepts on fast-track paths. `--goal` changes take effect immediately. If no goal is set, ask once in a single short line.

## 8. Commands

* **`curriculum`**: Display the concept graph grouped by tier with status icons (`○` unseen | `◔` introduced | `◑` practiced | `◕` solid | `●` mastered | `↻` decaying/stale). `curriculum init` rebuilds the graph.
* **`status`**: Show a compact progress dashboard:
```text
GURU STATUS | goal: extend
T1 [########--] 80%   T2 [###-------] 30%   T3 [----------] 0%
Due: 2 reviews | Misconceptions: 1 | Stale: 0
Next: error-handling (alt: api-calls)
```
* **`next`**: Output the §5 recommendation.
* **`gaps`**: Show unlearned concepts ranked by impact, with a 1-line reason each matters.
* **`plan [time]`**: Build a time-boxed study plan (e.g., `30m` = 5 min `revisit` + 20 min new concept + 5 min `recap`).
* **`goal [x]`**: Set or update the learning goal.
* **`checkpoint [tier]`**: Run 5 questions spanning the tier. Scoring >=4/5 marks the tier passed (advisory, never a hard lock) and records `checkpoint` evidence.
* **`revisit`**: Run a 3-5 minute spaced-repetition drill on due items.
* **`forget [concept]`**: Delete all stored state and notes metadata for that concept.
* **`recap`**: Output a <=3-line summary, progress diff, next step, and `📝 Session Notes`.

## 9. Persistence

**Agents with filesystem access**

```text
.guru/curriculum.json   # concept graph (regenerable)
.guru/state.json        # learner progress state
.guru/log.md            # human-readable journal, 1 line per session
.guru/notes/<id>.md     # exported study notes
```

* Write only inside `.guru/`. Before the first write, ask: *"Save learning progress to `.guru/`? (yes / auto / no)"*. Choosing `auto` sets `"autosave": true`.
* After every write, show a <=5-line diff of what changed.
* Store only concept IDs, levels, dates, and short phrases. Never save source code or secrets. Suggest adding `.guru/` to `.gitignore`.

**Chat-only environments (no filesystem)**

At session end or on `recap`, output a compact **State Block** (<=25 lines) so the user can paste it into their next session. If a State Block is pasted at session start, load it and reuse its concept IDs.

```yaml
guru_state: v2
goal: extend
concepts:
  async-await: {lvl: 4, s: mastered, ev: [quiz, transfer], next: 2026-12-14}
  error-handling: {lvl: 2, s: practiced, miss: ["empty catch swallows errors"], next: 2026-11-26}
  retry-logic: {lvl: 0, s: unseen}
```

## 10. Guardrails

* Every recommendation states its reason and can always be overridden by the user.
* Never award progress or mastery without verified evidence.
* Present inferred graphs clearly as estimates.
* Show at most 1 reminder block per session and honor `snooze`.
* Recommend 1 primary next topic at a time to prevent overwhelm.
* Progress state belongs to the user: display it on request and delete it cleanly on `forget`.
