# DSA Interview Prep Plan: 2–3 Months, Starting From "Done It All, Trust None of It"

For someone who's already been through most DSA topics once but doesn't feel confident applying any of them under pressure. The problem here usually isn't missing knowledge — it's that knowledge was never turned into fast, reliable *pattern recognition*. The plan below is built around fixing that, not re-teaching topics from scratch.

## The four phases

| Weeks | Phase | Focus | Rough weekly time |
|---|---|---|---|
| 1–2 | Diagnostic | Timed problems across *every* major topic; log where confidence actually breaks down, instead of guessing | ~10–12 hrs |
| 3–6 | Pattern drilling | Deep-dive the weak topics the diagnostic surfaced; build a pattern notebook; interleave topics instead of blocking them | ~12–15 hrs |
| 7–9 | Applied practice | Curated medium/hard list (NeetCode 150 / Blind 75), strict timing, 1–2 mock interviews/week | ~12–15 hrs |
| 10–12 | Mock-heavy consolidation | 2–3 mocks/week, revisit *only* what mocks expose as shaky, freeze new material | ~10 hrs |

## Phase 1 (Weeks 1–2): Diagnostic, not more studying

Solve 3 timed Medium problems from *each* of 12 topics, closed-book, no hints, and log the result honestly — this replaces the vague feeling of "I'm not confident" with an actual list. Expect to fail or struggle on several; that's the point of this phase, not a bad sign.

Full detail — the exact rules for each diagnostic problem, the 12-topic list with 3 representative problems each, a day-by-day 2-week schedule, the tracking sheet columns, and how to read the results into a Phase 2 weak-topic list — is in [phase-1-diagnostic.md](phase-1-diagnostic.md).

## Phase 2 (Weeks 3–6): Pattern drilling on the weak list

For each weak topic (usually 6–8 come out of the diagnostic):

- Re-learn the *trigger signal* for the pattern, not the topic label — e.g. "sorted array + pair/triplet sum" → two pointers; "substring/subarray with a constraint" → sliding window; "shortest path, unweighted graph" → BFS; "overlapping subproblems + optimal substructure" → DP. Write each trigger down in a running **pattern notebook** — this is what you'll actually skim before a real interview, not your old notes.
- Solve 8–12 problems per topic, Easy → Medium → Hard, in that order. Don't jump straight to Hard because you feel behind — confidence compounds from a run of wins, not from repeated failure on problems above your current level.
- **Interleave topics within the week** instead of finishing one topic before starting the next. A real interview never tells you "this is a DP problem" — you need practice *recognizing* the pattern cold, and blocked practice (all DP, then all graphs) never builds that.

## Phase 3 (Weeks 7–9): Applied practice under real constraints

- Switch to a curated list — NeetCode 150 or Blind 75 — at Medium/Hard, strictly timed (30–45 min per problem, no exceptions, stop and review the approach if you blow past it).
- Start 1–2 mock interviews per week (Pramp for free peer mocks, interviewing.io for paid mocks with real engineers, or a friend). Verbalizing your thought process out loud while solving is a distinct skill from solving silently — most candidates who "know DSA" but bomb interviews are failing at *this*, not at the algorithms.

## Phase 4 (Weeks 10–12): Mock-heavy, freeze new material

- Ramp mocks to 2–3/week.
- Revisit only what a mock actually exposes as shaky — not a general re-review of everything. By this point the goal is narrowing gaps, not broadening coverage.
- **Stop learning new topics 1–2 weeks before real interviews.** The last stretch is pure consolidation: re-skim your pattern notebook, re-solve a few problems you solved weeks ago *cold* (closed-book) to confirm retention — that's a better confidence check than doing yet another new problem.

## Common DSA-prep mistakes vs. what actually works

- **Weak** ❌: Solving problems randomly, by whatever feels appealing that day, with no tracking. — You end up with a felt sense of progress that doesn't map to actual topic coverage, and blind spots stay invisible until an interview finds them for you.
- **High-value** ✅: Structured tracking (the sheet from Phase 1) by topic and pattern, so gaps are visible on a spreadsheet, not just felt. — Turns "I don't feel confident" into "I'm weak specifically in graphs and DP," which is fixable.

- **Weak** ❌: Re-reading solutions/tutorials for a topic you feel weak in, without re-attempting problems cold afterward.
- **High-value** ✅: Timed, closed-book re-attempts on that topic days later. Confidence comes from doing it again yourself under the same conditions as a real interview, not from having read someone else's explanation.

- **Weak** ❌: Jumping straight to Hard problems on a topic because you feel behind on time.
- **High-value** ✅: Easy → Medium → Hard ladder per topic, even when short on time. A string of solved Easy/Medium problems builds real confidence; a string of failed Hards mostly reinforces the feeling you're trying to fix.

- **Weak** ❌: Cramming new topics in the final week before interviews.
- **High-value** ✅: Freeze new material 1–2 weeks out — pure revision and mocks only. New material this late adds anxiety without enough time to actually absorb it.

- **Weak** ❌: Practicing exclusively alone and silently.
- **High-value** ✅: Regular mock interviews, verbalizing your thought process out loud. This is a separate, trainable skill from quietly solving a problem, and it's what an interviewer is actually grading. See [english-speaking-practice.md](../interview-prep/english-speaking-practice.md) if the friction here is specifically spoken-English fluency rather than the DSA content itself.

## Common gotchas

- **"I've done almost all of DSA" and "I can apply it cold under time pressure" are different claims** — the diagnostic phase exists specifically to find where that gap actually is, instead of assuming it's uniform across all topics.
- **Confidence built from re-reading isn't the same as confidence built from re-solving** — if a topic still feels shaky after re-reading notes, that's a sign to schedule a timed re-attempt, not to read more.
- **Don't skip mocks until "ready"** — verbalizing under mild pressure is the actual interview skill, and it degrades under adrenaline in a real interview if it was never practiced in a lower-stakes mock first.
