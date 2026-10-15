# My Profile

Copy this file to `profile.md` in the repo root and fill it in with your own real experience.
`/drill` reads `profile.md` automatically if it exists, and grounds every model answer in the
facts here. Leave sections blank rather than guessing or padding — an honest gap produces a
better rehearsal than invented experience, since `/drill` will build an honest
acknowledge-bridge answer for anything you mark as a gap, and a fabricated strength will fall
apart the moment a real interviewer asks a follow-up.

This file stays local — it's already covered by `.gitignore`, so it won't get committed or
pushed even if you `git add .` by habit. Keep it that way if it has anything you'd rather not
have in your git history, even privately.

## Identity

- **Target roles:** e.g. Senior DevOps Engineer, SRE, Platform Engineer, DevOps Lead
- **Years of experience:** e.g. 6.5 years in cloud infrastructure / DevOps / SRE
- **Primary cloud:** e.g. AWS (how many years, how deep)
- **Current status:** e.g. employed and looking / between roles / open to relocation

## Professional Experience

List your real roles, most recent first. For each: company, title, dates, and 3-6 bullets of
what you actually did — lead with concrete numbers wherever you have them (a percentage, a
dollar figure, a before/after, a timeframe). This is the single most important section — it's
what `/drill` pulls real anchors from.

```
### <Title> - <Company> (<start> - <end or "Present">)
- <what you built/owned, with a number if you have one>
- <a migration, an incident you led, an automation you built>
- <a cost/reliability/velocity improvement, quantified>
```

## Technical Skills

- **Strong / deep, real production experience:** e.g. Kubernetes, Terraform, AWS (EC2/EKS/RDS/
  IAM), CI/CD (name your actual tools), observability (name your actual stack)
- **Moderate / working familiarity, not deep ownership:** things you've used but wouldn't claim
  flagship depth in — be honest here, this is exactly what keeps `/drill`'s scoring honest
- **Known gaps:** tools/platforms the roles you're targeting ask for for that you genuinely
  don't have — e.g. "no production GCP/Azure experience, AWS only," "no Go, Python/Bash only,"
  "no formal Incident Commander title." `/drill` weights these topics for extra reps and always
  answers them with an acknowledge-bridge pattern, never an invented story.

## Notable Stories (for behavioral / "tell me about a time" questions)

A handful of real STAR-shaped stories (Situation, Task, Action, Result) you can draw on —
a hard migration, an outage you led, something you automated, a disagreement you navigated,
a mistake you learned from. Three to six is plenty to start; add more as you think of them.

```
### <Short name for the story>
**S:** <context>
**T:** <what you were responsible for>
**A:** <what you actually did>
**R:** <the outcome, with a number if you have one>
**Use for:** <what kinds of questions this answers well>
```

## Career Goals / What You're Looking For

A line or two on what you want next and why — this shapes how `/drill` scores and frames
"why are you leaving," "where do you see yourself in 5 years," and similar questions.
