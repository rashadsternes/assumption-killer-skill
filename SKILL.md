---
name: the-assumption-killer
description: "Verifies that AI-generated code or large codebases behave as intended. Surfaces gaps between business intent and actual implementation by generating falsifiable assumptions about customer flows and system behavior, then proving or disproving each against the code."
disable-model-invocation: true
argument-hint: "[generate|verify|fix]"
---

# The Assumption Killer

Verify that your code does what you think it does.

AI-generated code runs. It passes linters. It might even have tests. But does it match your business intent? Does the checkout flow actually enforce the rules you described? Does the auth system link accounts the way you expect?

This skill makes those beliefs explicit, then tests each one against the actual code. It applies to customer flows, business logic, cross-system dependencies — anywhere the gap between "what I asked for" and "what got built" can hide bugs.

> **Token warning:** This skill is intentionally thorough. The `generate` and `verify` steps use parallel subagents to investigate multiple flows concurrently. Expect significant token usage, especially on large codebases. This is the tradeoff — shallow analysis misses bugs, deep analysis costs tokens.

## Step routing

- If `$0` is `generate` → run [Step 1: Generate](#step-1-generate)
- If `$0` is `verify` → run [Step 2: Verify](#step-2-verify)
- If `$0` is `fix` → run [Step 3: Fix](#step-3-fix)
- If `$0` is empty or unrecognized → explain the three steps and ask which to run

---

## Step 1: Generate

### Phase 1A: Repo Health Assessment

Before generating assumptions, assess the codebase to calibrate how much to trust what you read. A well-maintained repo gets benefit of the doubt. A neglected one gets deeper scrutiny on every assumption.

Investigate:

1. **Git history** — Run `git log --oneline -30` and `git shortlog -sn`. Evaluate:
   - Commit message quality (descriptive vs "fix" / "update" / "wip")
   - Commit frequency and recency
   - Number of contributors
   - Evidence of code review (merge commits, PR references)

2. **Code organization** — Use Glob and Read to assess:
   - Directory structure (logical grouping vs flat)
   - Naming conventions (consistent vs mixed)
   - Dead code, commented-out blocks, TODO density
   - Separation of concerns

3. **Documentation** — Check for:
   - README quality and currency
   - Architecture docs, decision records
   - CLAUDE.md or similar context files
   - Inline comment quality (meaningful vs noise)

4. **Test coverage** — Look for:
   - Test files and organization
   - Test runner config
   - Test-to-source ratio

5. **Dependency health** — Check package manifests for:
   - Lock file presence
   - Obviously outdated or deprecated packages

Write a brief **Repo Health Assessment** (5-10 lines) summarizing what you found. This calibrates the skepticism level for everything that follows.

### Phase 1B: Assumption Generation

Walk through **every user-facing flow** in the codebase, end to end. Be exhaustive — cover happy paths, error paths, edge cases, and boundary conditions.

**Use parallel subagents** (Task tool with `subagent_type: Explore`) to investigate multiple flows concurrently. Each flow is an independent investigation. Assign one subagent per flow or logical group of flows. This is where the depth comes from — each subagent can trace a full execution path without competing for context.

For each flow:

1. **Describe what happens step by step** — which files, which functions, in what order
2. **Flag anything you're uncertain about** — mark it explicitly with one of:
   - `[UNCERTAIN]` — you can't tell from the code alone
   - `[ASSUMPTION]` — you're making a reasonable guess based on patterns
   - `[UNCLEAR]` — the code is ambiguous or contradictory
3. **Call out cross-flow dependencies** — where one flow's failure could break another
4. **Rate your confidence** — calibrated against the repo health assessment. Lower confidence for areas with poor test coverage, cryptic naming, or missing docs.

### Output

Detect where the project keeps documentation:
- Look for `docs/`, `doc/`, `documentation/` directories
- Look for clusters of `.md` files in the project
- If nothing found, create `docs/`

Save to `{docs_dir}/assumptions.md` with this structure:

```markdown
# System Assumptions

> **Generated:** {date}
> **Repo health:** {one-line summary}

## Repo Health Assessment
{5-10 line calibration of codebase quality and what it means for trust levels}

## Flow 1: {Flow Name}

### What happens
{step-by-step walkthrough with file:line references}

### Assumptions
- [ASSUMPTION] {what you believe is true}
- [UNCERTAIN] {what you can't determine from code alone}
- [UNCLEAR] {where the code is ambiguous}

### Cross-flow risks
{where this flow depends on or affects other flows}

## Flow 2: ...
```

After saving, tell the user:

**"Review this file. Correct anything I got wrong. Add context I'm missing — business rules, customer behavior, historical decisions. The more you correct now, the more bugs verification will find. Run `/the-assumption-killer verify` when ready."**

**Critical:** Do NOT proceed to verification. Do NOT fix anything. The human correction step between generate and verify is what makes this work. Without it, you're just auditing code — with it, you're testing beliefs.

---

## Step 2: Verify

### Prerequisites

1. Find the assumptions file — look for `assumptions.md` in the docs directory. If not found, ask the user for the path.
2. Read it completely.
3. Auto-detect high-risk assumptions — scan for `[UNCERTAIN]`, `[ASSUMPTION]`, `[UNCLEAR]` markers. These get the deepest investigation.

### Verification Process

**Use parallel subagents** (Task tool with `subagent_type: Explore`) to verify multiple flows concurrently. Each flow or group of related assumptions is an independent investigation. Give each subagent:
- The specific assumptions to verify
- Instructions to trace the full execution path
- The confidence calibration from the repo health assessment

For **every** assumption in the document:

1. **Find the relevant code** — use Grep, Glob, and Read to locate the actual implementation
2. **Trace the execution path** — follow through function calls, not just the entry point
3. **Classify the result:**
   - **Confirmed** — code does what the assumption says. Cite file:line.
   - **Incorrect (code bug)** — the code doesn't do what it should. Describe the bug, impact, and fix options.
   - **Incorrect (wrong assumption)** — the code is fine but the assumption was wrong. Explain actual behavior.
   - **Partially correct** — some aspects confirmed, others not. Be specific.

4. **For high-risk items** (tagged `[UNCERTAIN]`, `[ASSUMPTION]`, `[UNCLEAR]`):
   - Trace the FULL execution path, including error handling
   - Check edge cases and boundary conditions
   - Look for related code that might contradict the assumption
   - Cross-reference with tests if they exist
   - Check git blame for recent changes that might have introduced inconsistencies

### Output

Save to `{docs_dir}/assumption-verification.md` with this structure:

```markdown
# Assumption Verification

> **Verified:** {date}
> **Input:** {path to assumptions file}
> **Repo health:** {summary from assumptions file}

## Verdict Summary

| Flow | Assumption | Status | Action Needed |
|------|-----------|--------|---------------|
| ... | ... | Confirmed / Incorrect (bug) / Incorrect (wrong assumption) / Partially correct | ... |

**Bugs found:** {count}
**Assumptions corrected:** {count}
**High-risk items investigated:** {count}

## Flow-by-Flow Verification

### Flow 1: {Flow Name} — {STATUS}

**Assumption:** {what was claimed}

**Verification:** {what you found, with file:line references}

**Impact:** {if incorrect — real-world consequence}

**Action:** {what to do about it}

---

## New Bugs Found

### Bug A: {Title}
**Severity:** Critical / High / Medium / Low
**Impact:** {user-facing consequence}
**Fix:** {concrete fix description}
**Priority:** P0-P3

## Corrections to Assumptions Doc
{list each assumption that needs updating with the corrected text}

## Prerequisites Discovered
{blocking issues found that must be addressed before planned work}
```

**Critical:** Do NOT fix anything. Output the verification document only. The user decides what to act on and in what order.

---

## Step 3: Fix

### Prerequisites

1. Find the verification file — look for `assumption-verification.md` in the docs directory.
2. Read it completely.
3. Present the list of actionable items to the user and **ask which to fix**. Do not assume all items should be fixed. The user may want to defer some, skip others, or reorder priorities.

### Fix Process

Work through approved items in the user's chosen priority order:

1. **Bugs found** (Incorrect — code bug):
   - Fix the code
   - Update or add tests if test infrastructure exists for the area
   - Note what was changed

2. **Assumption corrections** (Incorrect — wrong assumption):
   - Update the original `assumptions.md` with corrected information
   - These are documentation fixes, not code fixes

3. **Action items** (Confirmed with action needed):
   - Address action items noted in the verification
   - May be code fixes, documentation updates, or new issues to track

After fixing, update the verification doc to mark each addressed item as `[RESOLVED]`.

### Output

Provide a summary:
- Code changes made (files and brief description)
- Documentation updates
- Items deferred or skipped (and why)
- Remaining open items

---

## Additional resources

- For the methodology and principles behind this workflow, see [methodology.md](methodology.md)
- For an example of what verification output looks like, see [examples/sample-verification.md](examples/sample-verification.md)
