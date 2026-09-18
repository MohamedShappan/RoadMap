# 13 — The Study Schedule

## The ratio

You proposed 30% theory / 50% implementation / 20% review+interview. **I'd adjust it slightly:**

> ## 25% theory · 55% implementation · 20% review & interview practice

**Why:** the failure mode for motivated learners is always over-consuming and under-building. Knowledge from reading decays in weeks; knowledge from debugging your own broken index persists for years. The extra 5% moved into implementation also includes *reading real production code*, which is closer to building than to reading.

Two structural rules that matter more than the exact percentages:

1. **Never study a concept without touching it within 48 hours.** Read about indexes Monday, add one and run `EXPLAIN` Tuesday. Unapplied theory is a leaky bucket.
2. **The 20% review is non-negotiable, and it must be active.** Passive re-reading does nothing. Review means: explain it out loud without notes, redo a scenario question, or re-solve a problem you solved a month ago.

---

## The 1-hour/day version (~9–12 months)

Realistic for people with a full-time job and other commitments. Slower, but it works if you're consistent — consistency beats intensity every time.

| Day | Block | Content |
|---|---|---|
| **Mon** | 60 min | Theory: read one focused topic (15) + apply it in code immediately (45) |
| **Tue** | 60 min | Build: current project |
| **Wed** | 60 min | Build: current project |
| **Thu** | 60 min | Theory + exercise from the current stage |
| **Fri** | 60 min | Build: current project |
| **Sat** | 60 min | **Active review**: explain this week's concepts out loud, no notes + answer 3 interview questions aloud |
| **Sun** | 60 min | From Stage 4 onward: one system design, timed, out loud. Before Stage 4: build. |

**Weekly:** ~15 min theory-only, ~4h build, ~2h review/interview. Adjust stage durations ×2 from [02](./02-roadmap-stages.md).

**The one rule that makes this version work:** never skip two days in a row. A 20-minute day beats a zero day, because zero days compound into quitting.

---

## The 2-hours/day version (~6 months) ← the recommended default

This maps directly onto the stage durations in [02](./02-roadmap-stages.md) and onto [16 — the 6-month plan](./16-six-months.md).

| Day | Block 1 (45 min) | Block 2 (75 min) |
|---|---|---|
| **Mon** | Theory: new topic | Implement it in a scratch project |
| **Tue** | Exercise from the stage | Build: current project |
| **Wed** | Theory: new topic | Build: current project |
| **Thu** | Exercise from the stage | Build: current project |
| **Fri** | Build | Build: current project |
| **Sat** | **Review**: explain the week aloud, redo one exercise from memory | Interview questions out loud + (from Stage 4) one system design |
| **Sun** | Rest, or catch-up on unfinished work | Rest |

**Weekly:** ~3h theory, ~7.5h implementation, ~2.5h review/interview. Six days on, one off — the rest day is part of the plan, not a failure.

---

## The 3-hours/day version (~4 months)

For people between jobs or studying full-time. **The risk here is burnout and shallow passes**, so this version deliberately adds more review, not more theory.

| Day | Block 1 (60 min) | Block 2 (90 min) | Block 3 (30 min) |
|---|---|---|---|
| **Mon** | Theory: new topic | Build: current project | Interview questions aloud |
| **Tue** | Exercise / deliberate practice | Build | Review yesterday's concepts from memory |
| **Wed** | Theory: new topic | Build | Interview questions aloud |
| **Thu** | Exercise / deliberate practice | Build | Review |
| **Fri** | Theory or catch-up | Build | Review the week |
| **Sat** | **One full timed system design, out loud** | Build / project polish | Write up what you missed |
| **Sun** | Rest — genuinely | | |

**Weekly:** ~4h theory, ~10h implementation, ~4h review/interview.

**Warning for this version:** three hours of *focused* work is a lot. Two genuinely focused hours beat four distracted ones. If you find yourself re-reading the same paragraph, stop and go build something instead.

---

## How to spend each type of time

### Theory time (25%)
- One topic at a time. Read until you can explain *why it exists*, then stop and go apply it.
- Prefer official documentation and one good book per area over YouTube playlists. Docs are denser and more accurate.
- **Take notes in the form of questions you can later answer**, not summaries you'll never re-read.
- **Hard cap:** if you've been reading for 45 minutes without writing code, you're in tutorial hell. Stop.

### Implementation time (55%)
- Work on the **current stage's project** by default.
- Getting stuck is the point. Struggle for 30 minutes before searching; that struggle is what builds retrieval strength.
- **Always measure.** Add the index and time it. Add the cache and time it. Numbers make the learning concrete and give you interview material.
- Deliberately break things: kill Redis, unplug the database, send duplicate requests. The failure modes are half the curriculum.

### Review & interview time (20%)
- **Explain aloud from memory.** If you can't explain an index without notes, you don't know it yet.
- **Spaced repetition on the scenarios**, not definitions: redo a [checkpoint](./14-mastery-checkpoints.md) question from three weeks ago.
- **From Stage 4 onward: one timed system design per week, spoken.** By month 6 you'll have done ~20. That is the single highest-ROI interview activity available to you.
- Keep a running **"things I couldn't explain" list**. That list *is* your study plan for next week — it's better targeted than any curriculum, including this one.

---

## Weekly rhythm regardless of version

```
Mon–Fri : learn + build
Saturday: active review + interview practice + (later) one system design
Sunday  : rest, or a light catch-up if the week slipped
```

## Monthly rhythm

At the end of every month, spend one session on:
1. **Take the checkpoint** for the stage you just finished ([14](./14-mastery-checkpoints.md)). Be honest about what you couldn't answer.
2. **Update your "can't explain yet" list.**
3. **Write one paragraph** about the most interesting thing you learned. Publishing it is even better — explaining publicly forces the gaps into the open.
4. **Adjust the plan.** If a stage took 1.5× as long, that's information, not failure. Better to genuinely finish Stage 1 in six weeks than to fake it in four.

---

## Sustainability rules

- **Consistency > intensity.** 1 hour daily beats 7 hours on Sunday, by a wide margin.
- **Protect the rest day.** Skill consolidates during the gaps.
- **Track streaks, not hours.** Hours invite self-deception; "did I do the thing today?" doesn't.
- **Expect plateaus.** Around month 3 you'll feel like you're learning nothing. You're consolidating. Keep going.
- **One project at a time, finished.** The urge to start something new is usually avoidance of the hard part of the current thing.
