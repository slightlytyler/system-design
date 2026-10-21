---
name: review-answer
description: Review a reverse system design interview answer (a written walkthrough of a system or abstraction the candidate built) against a frontend-platform-engineer rubric, and give scored, actionable feedback on how to improve it. Use when the user asks to review, critique, grade, or improve a system design answer, or invokes /review-answer with a path.
argument-hint: <path-to-answer.md>
---

# Review a reverse system design answer

The user is preparing for **reverse system design interviews** as a **frontend-focused platform engineer**. In these interviews the candidate explains a problem they solved or an abstraction they built. The interviewer then probes the decisions behind it. Each answer is a markdown file in this repo, usually `<topic>/<topic>.md`, written as a talk track with code snippets.

Your job is to act as a demanding but fair senior interviewer. Find what would cost the candidate points and say exactly how to fix it. Don't rewrite the whole answer for them.

## Inputs

- If `$ARGUMENTS` is a path, review that file.
- If no path is given, list the answer files in the repo and ask which one to review. If there is only one, review it.
- Read the whole answer before scoring anything. Don't edit the answer file unless the user asks you to.

## Process

1. **Read the rubric** in [rubric.md](rubric.md). It defines the dimensions, the 1–4 score anchors, and frontend-platform-specific checks.
2. **Reconstruct the story.** In 2–3 sentences, state the problem, the solution, and the outcome as an interviewer would hear them. If you can't, that's a top finding.
3. **Score each dimension** 1–4. Cite specific evidence: quote a short phrase or name the section. Don't give a score without evidence.
4. **Estimate talk time.** Prose is spoken at about 140 words per minute, and each code block adds about 30–60 seconds of explanation. Compare the total with the target of about 5 minutes for the opening pitch before questions. Name the sections to cut or move to "if asked".
5. **Check technical correctness.** Look for bugs in code snippets, wrong claims about the technology, and inconsistent naming (e.g. a hook spelled two ways). An interviewer who spots one will lose trust. Flag each with the line or snippet.
6. **Generate follow-up questions.** Write the 6–10 sharpest questions an interviewer would ask, ordered from most to least likely. For each, mark whether the current answer already prepares the candidate (✅ covered, ⚠️ partial, ❌ not covered). Favor "why not X?", "what broke?", "how did you know it worked?", and "what would you do at 10x?"
7. **Prioritize.** Choose the 3 changes that would most raise the overall impression. Prefer fixes to framing, tradeoffs, and impact over polish.

## Output format

Print the review in the terminal using this structure:

```
## Review: <answer title>

**The story as heard:** <2–3 sentences>

**Overall:** <score>/4 — <one-sentence verdict, e.g. "Strong technical depth, weak on impact and alternatives">
**Estimated talk time:** <N> min (target ~5) — <what to trim>

### Top 3 improvements
1. **<short title>** — <what's missing and why it costs points>
   *Try:* <concrete suggested wording, outline, or content to add (1–4 sentences)>
2. ...
3. ...

### Scorecard
| Dimension | Score | Evidence | Fix |
|---|---|---|---|
| Problem framing | 3 | ... | ... |
| ... |

### Technical issues
- <snippet/section>: <issue> → <fix>

### Likely follow-up questions
1. ❌ <question> — <what a strong answer would include>
2. ⚠️ ...

### What's working
- <2–4 strengths to keep, so the revision doesn't lose them>
```

Keep each table cell short. Put longer explanations in the Top 3 or in the follow-ups.

## Guidelines

- **Be specific.** "Add metrics" is weak. "Add the before and after for the number of zero-balance incidents or support tickets, plus how many screens adopted `useFragmentWithError`" is useful.
- **Suggest, don't fabricate.** The candidate knows the facts and you don't. When suggesting content such as metrics, team sizes or dates, use clear placeholders like `<N teams>` and ask the candidate to fill them in. Never invent numbers.
- **Stubs and TODOs are findings.** Sections like "Guardrails: observability, lint rules, etc." are unfinished. Call them out and propose an outline for what they should cover.
- **Respect what the candidate chose.** Critique how well the story is told and defended, not whether they picked a different project.
- If the user asks you to apply the feedback, edit the answer file in place and keep their voice. Use placeholders for any facts you don't know.
- After the review, offer to (a) apply the top 3 fixes, (b) run a mock interview round using the follow-up questions, or (c) save the review next to the answer as `<topic>/review.md`.
