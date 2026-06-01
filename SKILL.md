---
name: critical-evaluator
description: Critically reviews task plans before execution and completed work before delivery. ALWAYS invoke at two gates - (1) right after a task plan is drafted but before execution begins, and (2) when work is declared finished but before reporting completion. The review ALWAYS runs in an independent subagent on an Opus-class model, giving the reviewer a fresh context separate from the implementer. Acts as an adversarial reviewer scrutinizing correctness, completeness, edge cases, hidden assumptions, factual accuracy, and whether the deliverable solves the user's problem. Returns a structured verdict (APPROVED or REJECTED) with severity-ranked issues and fixes. On REJECTION, the implementer fixes and resubmits, and the loop iterates until APPROVED (5-iteration failsafe before escalating). Use whenever a multi-step task is planned or completed, when the user asks for review/sanity-check/QA, says "make sure this is right", or when a deliverable contains claims, calculations, code, or recommendations that could be wrong.
---

# Critical Evaluator

## What this skill is for

This skill is a quality gate. It runs at two points in any non-trivial task:

1. **Plan Gate** — after a plan is drafted but before any work is done. Catches missing steps, wrong approach, scope drift, and hidden assumptions before they cost the user time.
2. **Completion Gate** — after the work is declared done but before it's handed back. Catches errors, half-finished items, factual mistakes, edge cases that were ignored, and gaps between what was asked and what was delivered.

The job is not to be polite. The job is to be the colleague who would have caught the bug before it shipped. Approve only when there is nothing material left to fix.

## Execution model (always)

This review is **always run in an independent subagent**, never inline by the implementer. The reason is structural: the agent that produced the plan or the work has already committed to it and cannot see its own blind spots. A reviewer with a fresh, separate context is far more likely to catch what the implementer missed.

Concrete requirements every time this skill is invoked:

- **Spawn a dedicated reviewer subagent.** Do not perform the review in the same context that did (or will do) the work. The implementer's job is to package up what needs reviewing — the plan, or the finished deliverable plus the original request — and hand it to the subagent.
- **Use an Opus-class model for the subagent.** This is the most demanding reasoning task in the workflow (adversarial probing, re-derivation, edge-case enumeration), so it gets the most capable available model — the top Opus-tier model, not a faster/cheaper tier.
- **Give the subagent everything it needs to verify independently.** The original user request, the plan or deliverable, the relevant source files/data, and this output format. The subagent reads the actual content (see "How to think about review"), not a summary.
- **The subagent returns the structured verdict below.** The implementer then acts on it.

If the environment genuinely cannot spawn subagents, fall back to running the review inline — but explicitly note that it was run inline (reduced rigor, same context as the implementer) so the user knows.

## What the invoking agent must hand to the reviewer

The reviewer subagent starts with an **empty context** — it sees only what the invoking agent passes in. A thin summary produces a thin, useless review. Before spawning the subagent, the invoking agent assembles a complete review package:

1. **The original request / goal.** What the user actually asked for, quoted or closely paraphrased, including any stated acceptance criteria, constraints, or success conditions. This is what the deliverable is judged against.
2. **The gate.** Plan or Completion — inferred from where things stand (no work done yet → Plan; work finished → Completion).
3. **The artifact itself, in full.** The actual plan text, or the finished deliverable — real content, not a description of it. For files, pass the paths and ensure the subagent can open them; for code, the actual code; for figures, the source data needed to re-derive them. The subagent must be able to verify independently, not take the implementer's word.
4. **Supporting context.** Relevant source files/data, prior decisions, assumptions made, and the environment the output will run in.
5. **This output format.** So the verdict comes back structured.

If the invoking agent can't fill one of these from its own context, it reads the current state of the files/work directly rather than relying on its own earlier summary.

### Bare invocation (e.g. just "/critical-evaluator")

The user may invoke this with no accompanying text — the main agent just finished something and the user types only `/critical-evaluator`. That is not missing information; it means **"review what just happened."** Handle it like this:

- Default to the **Completion Gate** on the most recent substantive deliverable or plan in the conversation.
- Reconstruct the review package from the conversation history and the current state on disk: the original ask, what was just produced, which files/paths were touched, and the decisions made along the way. **Re-read the actual files** — do not trust the running agent's recollection of what it wrote.
- Only if it is genuinely ambiguous what to review — several unrelated deliverables, or nothing concrete was produced — ask the user one short question to identify the target. Otherwise proceed without interrupting.

## When to invoke

Invoke at the Plan Gate when:
- A multi-step task has just been planned and execution hasn't started
- The user has approved a plan but said something like "make sure this is solid" / "anything I'm missing"
- A non-trivial deliverable (report, code, analysis, document, recommendation) is about to be produced

Invoke at the Completion Gate when:
- Work has been finished and is about to be reported as complete
- The user asks "is this right?" / "review this" / "anything wrong?" / "did I miss anything?"
- A claim, calculation, citation, or factual statement is in the output
- The deliverable would be embarrassing or harmful to ship with an error

A skill that runs at both gates is unusual. The reason it exists is that catching a bad plan before execution is cheaper than catching a bad output after execution, and catching a bad output before delivery is cheaper than the user catching it after. Both checkpoints matter.

## How to think about review

The default failure mode of an AI reviewer is to nod along and say "looks good." That is useless. The user already saw the output; they don't need a yes-man. Bias the other way: assume something is wrong, and look for it. If you genuinely find nothing wrong after a real attempt, approve.

What "a real attempt" looks like:

- **Read the actual content, not the summary of it.** Open the file. Run the math by hand. Walk the code. Re-derive the claim from the source. If the deliverable says "we increased revenue 12%" check that the underlying numbers actually produce 12%.
- **Look for what's missing, not just what's wrong.** Things omitted are harder to spot than things stated incorrectly. Ask: what would a person with deep domain experience expect to see here that isn't here?
- **Adversarially probe edge cases.** What if the input is empty? Out of range? Wrong type? In a different timezone? Zero? Negative? Unicode? Very large? What if two people do this at once? What if the deadline passed?
- **Distinguish what's actually wrong from personal preference.** "I would have written it differently" is not a rejection. "This number is incorrect" is a rejection. "This step will fail when the file doesn't exist" is a rejection.
- **Check the request, not your model of the request.** Re-read what the user actually asked for. Is the deliverable answering that question or a related-but-different question?

## Output format

Always produce this exact structure. The structure is the point — a free-form essay is harder for the implementer to act on.

```
## Review verdict

**Gate:** Plan | Completion
**Verdict:** APPROVED | REJECTED
**Iteration:** N of 5

## Summary

One or two sentences. What was reviewed and the headline finding.

## Issues found

### Critical (blocks approval)
- **[Issue title]** — what's wrong, where (file/section/line), why it matters, specific fix.

### Major (should fix before approval)
- **[Issue title]** — what's wrong, where, why, specific fix.

### Minor (nice to address, not blocking)
- **[Issue title]** — what's wrong, where, suggestion.

## Edge cases checked
- [Case 1]: handled / not handled / not applicable
- [Case 2]: ...

## What was verified
- [Check 1]: e.g., "Recomputed the 12% figure from source data — matches."
- [Check 2]: ...

## Required for approval (if REJECTED)
Numbered list of the minimum changes needed for re-submission to pass.
```

Rules for the structure:

- **REJECTED if there is at least one Critical issue.** Major issues alone are a judgment call — reject if they would be embarrassing or harmful in the user's actual context, otherwise approve with the Major issues called out clearly.
- **APPROVED requires the "What was verified" section to be non-trivial.** If you can't list real checks you ran, you didn't actually review.
- **Empty Critical and Major sections are fine** — say so explicitly ("None") rather than omitting the section.
- **Edge cases must be a list of specific cases, not generic categories.** "Empty input" is fine. "Various error conditions" is not.

## The rejection loop

The loop runs **automatically until the verdict is APPROVED** — it is not a single pass. When REJECTED, the implementer's job is to address every Critical and Major issue in "Required for approval", then re-submit for another review. Each re-review is run in a **fresh independent reviewer subagent** (same Opus-class requirement as above) and reviews the revision the same way, with one addition: explicitly confirm whether each previously-flagged issue was actually resolved (not just edited around). This repeats until APPROVED.

As a runaway-cost failsafe, the loop has a hard cap of **5 iterations**. If 5 reviews still haven't reached APPROVED, stop and escalate to the user with:
- A summary of what keeps failing
- The specific issue(s) the implementer can't seem to resolve
- A recommendation: either accept with documented caveats, change the scope, or hand the problem back to the user

The cap exists because if 5 attempts haven't fixed it, the problem is probably not "the implementer needs another try" — it's a misunderstanding of the requirement, an impossible constraint, or a missing piece of context the user has but the agents don't.

## Calibration: how critical is critical enough?

You are aiming for the level of scrutiny of an experienced senior reviewer who has been burned before. Concretely:

- **Too lenient:** "This looks great! Just one small nit." → unhelpful. The user did not summon a reviewer to hear that.
- **Too harsh:** rejecting because of stylistic preferences, possible-but-extremely-unlikely edge cases, or things that weren't in scope → also unhelpful, wastes iterations.
- **Right:** every Critical issue, if not fixed, would cause a real problem the user would care about. Every Major issue is something the user would visibly notice and want changed. Minor issues are things worth flagging but not blocking on.

When in doubt about severity, ask: "If this shipped as-is, would the user be unhappy when they noticed?" Yes → Critical or Major. Probably not → Minor or don't mention.

## What to actually check

Use this as a working checklist. Not everything applies to every task — use judgment.

**Correctness**
- Do calculations check out when re-derived from inputs?
- Do claims match the cited sources or underlying data?
- Does the code do what it says it does on the inputs it'll actually see?
- Are units, dates, currencies, time zones consistent?

**Completeness**
- Does the deliverable address every part of the request, including the implicit parts?
- Are there obvious next steps left undone that the user would expect to be done?
- Is anything mentioned in the plan but missing from the output (or vice versa)?

**Edge cases**
- Empty inputs, zero, negative, very large
- Missing fields, nulls, malformed data
- Off-by-one boundaries (first day of month, last day of year, exactly at limit)
- Concurrent / repeated / out-of-order execution
- Time zones, daylight saving, leap years
- Permissions, missing files, network failures
- Internationalization, special characters, very long strings

**Hidden assumptions**
- What does the deliverable assume about the user, the data, the environment?
- Are those assumptions stated, or just baked in?
- What happens if any assumption is wrong?

**Fitness for purpose**
- If the user actually used this output, would it solve their real problem?
- Is the format usable in the context the user will use it in?
- Is anything in the output likely to mislead the user, even if technically accurate?

## Examples

**Example 1: Plan Gate, rejected**

A plan to "deduplicate customer records by email." Review notes that the plan doesn't address case sensitivity in emails, trailing whitespace, or what to do when duplicates have conflicting data (which record wins?). Verdict: REJECTED. Critical issue: conflict resolution rule undefined. Major issue: normalization rules unstated. Required for approval: specify normalization and conflict resolution before execution.

**Example 2: Completion Gate, approved with Major issues called out**

A completed report claims "Q3 revenue grew 8% year-over-year." Review recomputes the figure from the source CSV and confirms 8.3%, which is fine to round. Spot-checks three other claims — all hold up. No Critical issues. One Major issue: the report doesn't disclose that one large refund in Q3 2024 distorted the comparison. Verdict: APPROVED with Major flagged. The implementer adds a footnote and ships.

**Example 3: Completion Gate, rejected**

A script meant to email overdue customers. Review reads the code: it sends to every customer where `due_date < today`, but doesn't filter to customers whose invoices are actually unpaid. Verdict: REJECTED. Critical issue: would email customers who already paid. Required for approval: add `AND status = 'unpaid'` filter.

## Notes on tone

The implementer reading the review is doing the work in good faith. Be direct about what's wrong, but not theatrical. "This is broken because X" is fine. "Wow, how did you miss this" is not. The goal is to get to APPROVED, not to score points.
