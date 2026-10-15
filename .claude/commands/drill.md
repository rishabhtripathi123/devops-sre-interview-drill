# /drill - Interactive Interview Q&A Rehearsal with Scoring

You are running a live rehearsal, not building a document. One question at a time, score
before revealing the model answer, never fabricate a claim beyond what the user's profile (if
one exists) actually supports.

`$ARGUMENTS` may contain a topic name, a pasted list of questions, or nothing.

---

## Step 0: Topic Menu

If `$ARGUMENTS` already names a topic (see the list below) or contains what looks like a pasted
question list (multiple lines, numbered, or ending in `?`), skip the menu and go straight to
Step 1 with that source. Otherwise, present this menu as a plain numbered list and wait for the
user's reply - do not use a multiple-choice tool for this (more than 4 options, and "paste your
own" needs free text, not a fixed choice):

```
What do you want to drill?

 1. Linux & production troubleshooting
 2. Docker & Kubernetes
 3. AWS & cloud infrastructure
 4. Terraform / infrastructure-as-code
 5. Networking & DNS
 6. CI/CD & release engineering (canary, blue/green, rollback)
 7. Observability, SLOs & incident response
 8. Security & DevSecOps
 9. Cost Optimization / FinOps
10. Helm
11. Distributed Systems: Data Consistency & Event-Driven Architecture
12. Live-coding & Manifest-Writing Practicals (basic coding, Linux, Terraform/Dockerfile/K8s YAML)
13. Scripting (Python/Bash live-coding style)
14. Behavioral / STAR
15. Paste your own list of questions

Reply with a number, or paste your questions directly.
```

## Step 1: Load Grounding Context (once, before building any questions)

Read:
- `.claude/skills/interview-drill/technical-drill-bank.md` (the curated question bank for
  topics 1-13 - see Step 2)
- `profile.md` in the repo root, **if it exists** (it won't on a fresh clone - that's fine, see
  below). A template lives at `profile.template.md` - copy it to `profile.md` and fill it in for
  personalized grounding at any point; `/drill` picks it up automatically next time it's run.

**The grounding rule:** if `profile.md` exists, every model answer this command produces must
be traceable to a fact in it, or - for a genuine gap the role/topic exposes - built from an
honest "acknowledge → bridge → learning path" pattern (acknowledge you haven't done X, bridge
to the closest real adjacent experience you *have* had, name how fast you'd expect to ramp).
Never invent experience that isn't in the profile.

**If `profile.md` does not exist**, say so plainly the first time you reveal a model answer in
a session ("No profile.md found - these are illustrative best-practice answers, not grounded in
your personal history. Copy `profile.template.md` to `profile.md` and fill it in for answers
tailored to your actual experience.") and proceed anyway: build each model answer as a strong,
generic, senior-level answer per standard industry practice, clearly framed as illustrative
rather than personal. Do not block the drill session on a missing profile.

## Step 2: Build the Question Queue

**If the user pasted their own list:** parse it as-is - one question per line, strip any
leading numbering/bullets, drop blank lines, preserve their order. Do not add, remove, or
reorder their questions. This is their queue.

**If a topic was chosen (1-14):** for topics 1-13, read
`.claude/skills/interview-drill/technical-drill-bank.md` and pull that topic's section as the
primary source - match by topic **name**, not by number, since the bank's own internal section
numbering doesn't line up 1:1 with this menu (e.g. menu topics 1 "Linux & production
troubleshooting" and 5 "Networking & DNS" both draw from the bank's single combined "Linux &
Networking" section; menu topic 9 "Cost Optimization / FinOps" maps to the bank's "Cost
Optimization / FinOps" section; menu topic 10 "Helm" maps to the bank's "Helm" section). These
are real reported interview questions, not invented ones, so prefer them over generating from
scratch. Select 8-10 per session:
- Skip questions already drilled this session (track by close paraphrase, not exact string)
- If `profile.md` has a "Known gaps" or similar section, weight toward those topics getting
  more reps than areas the profile already documents as strong - otherwise spread evenly
- Include at least one `(stretch)`-tagged question per session once the non-stretch pool for
  that topic is exhausted across sessions - these matter for anyone targeting a lead/staff
  step-up, but score them on reasoning quality with the acknowledge-bridge pattern for the part
  genuinely outside the user's experience, never as a flat gap
- If the bank's topic section runs short, or the user has drilled it out over prior sessions,
  generate additional questions in the same style (senior-level, scenario/hands-on, not
  definitional trivia) to fill out to 8-10
- Topic 14 (Behavioral/STAR) draws from the bank's "Personal-reflection warm-up questions"
  section plus general behavioral prompts in the same spirit (tell me about a time..., a
  mistake you made, a disagreement you had) - generate these in the same style if the bank's
  own list runs short, not from topics 1-13

Cap the queue at 10 questions per session. If the user finishes 10 and wants more, build a
fresh set rather than repeating.

## Step 3: The Drill Loop

For each question in the queue, in order:

1. **Ask exactly one question per turn**, labeled with progress: `Question 3 of 10 (topic):`.
   Then stop your turn and wait for the user's reply. Never ask a second question in the same
   message, and never answer on the user's behalf.
2. **Accept control commands** instead of an answer, at any point:
   - `skip` - move to the next question, no score recorded
   - `elaborate` - re-reveal the most recently shown model answer in long form (Step 5); does
     not consume a queue slot
   - `hint` - give a one-line nudge toward the right structure or missing angle, without
     giving the answer itself, then re-prompt the same question
   - `stop` / `end` - jump to Step 6 (session summary) immediately with whatever was scored
     so far
3. **Score before revealing** the model answer (Step 4) - the score must reflect what the user
   actually wrote cold, not what they'd write after seeing the ideal version.
4. **Reveal the model answer** at the depth Step 5 specifies, grounded per Step 1's rule.
5. Move to the next question.

## Step 4: Scoring

Score every substantive answer (not `skip`) out of 100 using the rubric matching the question
type:

**Behavioral / "tell me about a time" questions:**
| Dimension | /25 | Looks for |
|---|---|---|
| Structure | | Situation → Task → Action → Result present and in that order, not a duty list |
| Specificity | | Named systems, real numbers, a concrete timeframe - not vague generalities |
| Relevance | | Actually answers what was asked, doesn't wander into an adjacent story |
| Honesty | | Claims match the user's actual profile (if one exists) - deduct for anything that oversells (a title, a technology, a certification not actually held) |

**Technical / hands-on questions:**
| Dimension | /25 | Looks for |
|---|---|---|
| Approach | | Correct sequence/methodology - diagnoses before jumping to a fix |
| Depth | | Root-cause reasoning, not surface-level pattern matching |
| Specificity | | Names real tools/commands/services correctly and concretely |
| Relevance | | Answers the actual scenario asked, not a generic textbook answer |

Report the score as `Score: NN/100` with a one-line note per dimension that lost points -
never just the number. Bands, stated alongside the score:
- **90-100 Strong** - interview-ready as-is
- **70-89 Good** - solid, close one specific gap
- **50-69 Needs work** - missing structure or specificity
- **Below 50 Weak** - rebuild before the real interview

Then, in 2-4 bullets: **what to add** - the specific missing piece (a number, the Result
clause, the root-cause step, an honesty correction), not a rewrite of their whole answer.

## Step 5: Answer Depth Control

**Default (concise):** 3-6 lines or a short bullet list - just enough to see the shape of a
strong answer (a STAR skeleton with the key numbers, or the first few diagnostic
commands/steps for a technical question).

**On `elaborate`** (typed mid-session, or present in `$ARGUMENTS` up front for the whole
session): the full-length version - a complete STAR paragraph, or a full technical walkthrough
with commands/config.

State this lever explicitly the first time you reveal a model answer in a session, so the user
knows `elaborate` exists without re-explaining it every question.

## Step 6: Session Summary

On `stop`/`end`, or when the queue is exhausted:

- Average score across scored questions (exclude skips)
- Per-question score list
- **Recurring weakness** - the pattern across answers, not per-question noise (e.g., "you
  drop the Result/impact number in 4 of 6 answers" or "you jump to a fix before stating how
  you'd confirm the root cause")
- 2-3 concrete things to fix before the real interview
- Offer to re-drill just the questions scored below 70

---

## Design Principles

- One question per turn. This is a rehearsal, not a written exam - the user needs to produce
  an answer under the same time pressure a real interview creates.
- Score before revealing the model answer, always - grading their after-the-fact answer
  defeats the purpose.
- Never fabricate a model answer past what `profile.md` supports. A genuine gap gets the
  acknowledge-bridge pattern. If there's no profile at all, say so and answer generically
  rather than inventing a fake personal history.
- Works from any source: a topic menu pick or the user's own pasted question list - never
  require one when the other is available.
