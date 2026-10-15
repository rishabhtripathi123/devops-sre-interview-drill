# devops-sre-interview-drill

A free, open interview-rehearsal tool for DevOps / SRE / Platform Engineering interviews,
built as a [Claude Code](https://docs.claude.com/claude-code) slash command.

Run `/drill`, pick a topic (Kubernetes, Terraform, AWS, CI/CD, Observability & SLOs, Security,
Helm, live-coding/manifest-writing, behavioral, and more), and Claude asks you **real,
senior-level interview questions one at a time** — curated from actual reported interview
loops, not generated trivia. You answer cold, get scored against a rubric, and only then see a
graded model answer — the same way a real interview grades you, except you get to fail safely
here first.

Optionally, fill in your own background in `profile.md` and every model answer gets grounded in
*your* real experience — including an honest "acknowledge the gap → bridge to adjacent
experience → say how fast you'd ramp" pattern for anything you genuinely haven't done, instead
of ever inventing a story for you.

## Why this exists

Reading a list of interview questions doesn't prepare you for an interview — producing an
answer under time pressure, cold, does. Most interview-prep content online is either a static
question list (no feedback) or a chatbot you can prompt into giving you an easy pass. This tool
is deliberately structured so you can't skip the hard part: it asks one question, waits for your
real answer, scores it honestly, and only reveals the ideal answer afterward.

## What's inside

```
.claude/
  commands/
    drill.md                              — the /drill slash command
  skills/interview-drill/
    technical-drill-bank.md               — 150+ real, curated interview questions across:
                                             Linux & Networking, Docker & Kubernetes, AWS,
                                             Terraform/IaC, CI/CD, Observability/SLOs/Incident
                                             Response, Security & DevSecOps, Cost/FinOps,
                                             Scripting, Helm, Distributed Systems, live-coding
                                             & manifest-writing practicals, stretch/staff-level
                                             scenarios, and behavioral warm-ups
profile.template.md                       — copy this to profile.md and fill it in (optional,
                                             but makes every answer personal instead of generic)
```

## Setup — works on any machine, no accounts or API keys beyond Claude Code itself

1. **Install Claude Code** if you don't have it yet:
   ```
   npm install -g @anthropic-ai/claude-code
   ```
   (see [docs.claude.com/claude-code](https://docs.claude.com/claude-code) if you hit any
   issues — you'll need a Claude.ai or Anthropic Console account to log in)

2. **Clone this repo**:
   ```
   git clone https://github.com/rishabhtripathi123/devops-sre-interview-drill.git
   cd devops-sre-interview-drill
   ```

3. **(Optional but recommended) Personalize it**:
   ```
   cp profile.template.md profile.md
   ```
   Open `profile.md` and fill in your real experience, skills, known gaps, and a few STAR-shaped
   stories. `profile.md` is already in `.gitignore` — it will never get committed, even by
   accident, so your real background stays local to your machine.

4. **Start Claude Code** in this directory:
   ```
   claude
   ```

5. **Run it**:
   ```
   /drill
   ```
   Pick a topic number, or just paste your own list of questions straight from a job posting or
   a friend's interview notes — `/drill` works from either source.

That's it. No API keys beyond what Claude Code itself needs, no database, no build step — the
whole thing is markdown files Claude Code reads at runtime.

## How a session works

- **One question at a time.** You answer, Claude scores it (`Score: NN/100` with a breakdown by
  dimension — approach/depth/specificity/relevance for technical questions, or
  structure/specificity/relevance/honesty for behavioral ones), *then* shows you the model
  answer.
- **Control commands**, typed instead of an answer: `skip` (move on, no score), `hint` (a nudge,
  not the answer), `elaborate` (the long-form version of the last model answer), `stop`/`end`
  (end the session early with a summary).
- **Session summary** at the end: your average score, per-question breakdown, the *recurring*
  weakness pattern across your answers (not just per-question noise), and an offer to re-drill
  anything you scored below 70.

## Keeping the question bank current

`technical-drill-bank.md` is meant to grow. Found a real question from your own interview, a
friend's, or somewhere online? Open a Claude Code session in this repo, paste it in, and ask
Claude to add it to the bank — it'll check for duplicates and fold in only genuinely new
material, in the same style as what's already there (a question plus a one-line pointer to
what a strong answer touches, not a full scripted answer, so the bank doesn't go stale).

Contributions via pull request are welcome too, in the same style: real reported questions, not
AI-generated trivia, with a pointer rather than a scripted answer.

## A note on how this was built

This repo started as the `/drill` component of a larger personal job-search framework, then was
pulled out, genericized, and open-sourced as a standalone tool. If you use it and build on it,
a star or a mention is always appreciated but never required.

## License

MIT — see [LICENSE](LICENSE). Use it, fork it, adapt it, drop it into your own job-search setup.
