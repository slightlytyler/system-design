---
name: review-system-design
description: Review a regular system design interview answer (a written design for a prompt like "design a news feed" or "design a typeahead") against a frontend-platform-engineer rubric, and give scored, actionable feedback on how to improve it. Use when the user asks to review, critique, grade, or improve an answer under system-design/, or invokes /review-system-design with a path. For answers about systems the candidate actually built, use review-reverse-answer instead.
argument-hint: <path-to-answer.md>
---

# Review a system design answer

The user is preparing for **system design interviews** as a **frontend-focused platform engineer**. In these interviews the candidate gets an open-ended prompt and designs a system live, with the interviewer steering and probing. Prompts may be frontend-heavy ("design a news feed", "design a collaborative editor") or general ("design a URL shortener"). Each answer is a markdown file in this repo, usually `system-design/<topic>/<topic>.md`, with the prompt at the top and the design written the way the candidate would walk through it.

Your job is to act as a demanding but fair senior interviewer. Find what would cost the candidate points and say exactly how to fix it. Don't rewrite the whole answer for them.

## Inputs

- If a path is given, review that file.
- If no path is given, list the answer files under `system-design/` and ask which one to review. If there is only one, review it.
- Read the whole answer before scoring anything. Don't edit the answer file unless the user asks you to.
- If the answer doesn't state the prompt, ask for it. You can't judge requirements without it.
- If an `interviewer-notes.md` sits next to the answer, read it too. Use its clarifications, mid-interview reveals and "what a strong answer covers" list when scoring and generating probes. It's not an answer file, so don't list it as one.

## Process

1. **Read the rubric** in [rubric.md](rubric.md). It defines the dimensions, the 1–4 score anchors, the phase budget, and frontend-specific checks.
2. **Classify the prompt** as frontend-heavy, backend-heavy, or full-stack. This decides which depth checks matter most.
3. **Reconstruct the design.** In 2–3 sentences, state the scope, the core architecture, and the hardest problem the candidate chose to go deep on. If you can't, that's a top finding.
4. **Score each dimension** 1–4. Cite specific evidence: quote a short phrase or name the section. Don't give a score without evidence.
5. **Check the pacing.** Map each section to the phase budget in the rubric and estimate how long it would take to say and sketch live. Flag phases that would overrun or get skipped. The most common failure is spending so long on requirements and the high-level design that the deep dive never happens.
6. **Check technical correctness.** Look for wrong claims about protocols, browsers, storage or data structures; estimates that are off by orders of magnitude; and APIs or data models that contradict each other. Flag each with the line or section.
7. **Generate interviewer probes.** Write the 6–10 sharpest questions or curveballs an interviewer would throw, ordered from most to least likely. For each, mark whether the current answer already prepares the candidate (✅ covered, ⚠️ partial, ❌ not covered). Favor "what happens when X fails?", "how does this change at 10x / 100x?", "why not Y?", "what does the user see while Z?", and requirement changes ("now it needs to work offline").
8. **Prioritize.** Choose the 3 changes that would most raise the overall impression. Prefer fixes to scoping, tradeoffs and the deep dive over polish.

## Output format

Print the review in the terminal using this structure:

```
## Review: <prompt>

**Prompt type:** <frontend-heavy | backend-heavy | full-stack>
**The design as heard:** <2–3 sentences>

**Overall:** <score>/4 — <one-sentence verdict, e.g. "Clear architecture, but the deep dive is shallow and tradeoffs are implicit">
**Pacing:** <which phases fit, which overrun or are missing>

### Top 3 improvements
1. **<short title>** — <what's missing and why it costs points>
   *Try:* <concrete suggested wording, outline, or content to add (1–4 sentences)>
2. ...
3. ...

### Scorecard
| Dimension | Score | Evidence | Fix |
|---|---|---|---|
| Requirements and scope | 3 | ... | ... |
| ... |

### Technical issues
- <section>: <issue> → <fix>

### Likely interviewer probes
1. ❌ <question> — <what a strong answer would include>
2. ⚠️ ...

### What's working
- <2–4 strengths to keep, so the revision doesn't lose them>
```

Keep each table cell short. Put longer explanations in the Top 3 or in the probes.

## Guidelines

- **Be specific.** "Discuss caching" is weak. "Say where the feed is cached on the client, how it's keyed by cursor, and what invalidates it when the user posts" is useful.
- **Judge the reasoning, not the answer.** There's no single correct design. Penalize unstated assumptions and unjustified choices, not choices you'd have made differently.
- **Estimates should be in the right order of magnitude.** Don't nitpick arithmetic, but flag an estimate that would change the design if it were corrected.
- **Stubs and TODOs are findings.** Call them out and propose an outline for what they should cover.
- If the user asks you to apply the feedback, edit the answer file in place and keep their voice.
- After the review, offer to (a) apply the top 3 fixes, (b) run a mock interview where you play the interviewer and throw the probes one at a time, or (c) save the review next to the answer as `system-design/<topic>/review.md`.
