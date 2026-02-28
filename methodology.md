# The Assumption Killer — Methodology

## The problem

AI writes code that runs. It passes linters, compiles, even handles edge cases. But "runs correctly" and "does what the business needs" are different things. The same gap exists in any large codebase — the code works, but does it match the intent?

Code review catches syntax and style. Tests catch regressions. Neither catches the case where the code confidently does the wrong thing because nobody verified it against business intent.

## The core insight

"Audit my codebase" is an unbounded task. "Here are 10 things I believe are true about my customer flows — check each one" is a bounded, falsifiable task.

The difference is enormous. An audit produces vague observations. A verification produces confirmed facts, discovered bugs, and corrected misunderstandings.

## The principles

| Principle | What it means |
|-----------|---------------|
| **Give it something to be wrong about** | Assumptions with uncertainty flags give the agent a falsifiable claim to investigate. Without claims to test, the agent just describes what it sees. |
| **Scope the output format** | "New file, don't fix anything" prevents premature action and keeps the output as a clean verification artifact. |
| **Prioritize within the prompt** | Uncertainty markers (`[UNCERTAIN]`, `[ASSUMPTION]`, `[UNCLEAR]`) tell the agent where to dig deepest. Not all assumptions deserve equal investigation time. |
| **Layer documents over sessions** | Each document adds precision without repeating context. The assumptions doc references the codebase. The verification doc references the assumptions. Each layer is richer than the last. |
| **Calibrate against repo health** | A well-maintained repo with clean commit history deserves more benefit-of-the-doubt. A messy repo with cryptic commits deserves deeper scrutiny on every assumption. The repo assessment sets the baseline skepticism level. |
| **Two-step chain** | Step 1 (teach-back) generates the target. Step 2 (verification) tests it. Neither works as well alone. The teach-back surfaces what you *think* is true. Verification tests it against what *is* true. |
| **Human correction between steps** | The user reviews Step 1 output before Step 2 runs. This injects domain knowledge the AI can't infer from code — business rules, customer behavior, historical decisions, intent behind weird code. |
| **Parallel subagents for depth** | Each flow gets its own investigation agent with a narrow, specific question to answer. This is what enables exhaustive verification without running out of context or cutting corners. |

## What makes it different

| Approach | What it catches | What it misses |
|----------|----------------|----------------|
| **Linters/types** | Syntax, type errors | Business logic gaps |
| **Code review** | Style issues, obvious bugs | Cross-flow failures, intent mismatches |
| **Automated tests** | Regressions in tested paths | Untested assumptions about behavior |
| **The Assumption Killer** | Logic gaps, missing validation, intent vs implementation mismatches, cross-flow failures | Nothing it's pointed at — but it's slow and token-heavy |

| Code Review | The Assumption Killer |
|-------------|-------------------|
| Looks at code and reports what it sees | Tests specific beliefs against code |
| Produces observations | Produces verdicts (confirmed / incorrect) |
| Scope is "this file" or "this PR" | Scope is "every customer flow" |
| Finds style issues and obvious bugs | Finds where business intent diverges from implementation |
| One pass | Two passes with human correction between |
| Reviewer brings their own focus | Uncertainty markers direct focus to highest-risk areas |

## Why the teach-back matters

The instinct is to skip Step 1 and go straight to "find bugs." But the teach-back step is where the real value comes from:

1. **It surfaces implicit knowledge.** You know things about your system that aren't in the code — business rules, customer behavior, why that weird workaround exists. The teach-back makes the AI state its understanding so you can correct it.

2. **It creates falsifiable claims.** "The system handles X" is vague. "When a user does Y, function Z in file A calls function W in file B, which returns X" is testable. Step 1 generates the testable version.

3. **It reveals the AI's blind spots.** The `[UNCERTAIN]` markers are honest admissions of ignorance. These are exactly the spots where bugs hide — if the AI can't figure out what the code does, there's a good chance the code is unclear enough to harbor issues.

## When to use it

- **After AI generates significant code** — verify the code matches your business intent, not just your prompt
- **Onboarding** to a new codebase — generates a verified mental model of customer flows
- **Before a major change** (payment integration, auth overhaul, API migration) — surfaces hidden assumptions that could break
- **After inheriting code** — calibrates trust in what the previous team built
- **Periodic health check** — re-verify assumptions as code evolves
- **Pre-launch audit** — find the bugs that only appear when real users hit edge cases

## When NOT to use it

- Small, well-tested utility libraries (not enough customer flows to analyze)
- Code you wrote yesterday (you already know the assumptions)
- As a substitute for automated tests (this finds what to test, not a replacement for tests)
- Trivial CRUD apps with no business logic (nothing interesting to be wrong about)

## Token usage

This skill is intentionally token-heavy. The depth is the point.

- **Generate** uses parallel subagents to trace every flow. Each agent investigates independently, which means thorough coverage but significant token usage.
- **Verify** also uses parallel subagents, one per flow or assumption group. Each agent traces full execution paths including error handling.
- A codebase with 10 flows might spawn 10+ subagent investigations across both steps.

The tradeoff is explicit: shallow analysis is cheap but misses bugs. This workflow found 2 bugs and corrected 3 assumptions in its first use on a production codebase. The token cost paid for itself in prevented customer-facing issues.
