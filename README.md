# The Assumption Killer

A Claude Code skill that verifies AI-generated code and large codebases behave as intended.

AI writes code that runs. It passes linters, compiles, even handles edge cases. But "runs correctly" and "does what the business needs" are different things. This skill makes beliefs about your code explicit, then tests each one against the actual implementation.

## What it finds

| Approach | What it catches | What it misses |
|----------|----------------|----------------|
| Linters/types | Syntax, type errors | Business logic gaps |
| Code review | Style issues, obvious bugs | Cross-flow failures |
| Automated tests | Regressions in tested paths | Untested assumptions |
| **The Assumption Killer** | Logic gaps, missing validation, intent vs implementation mismatches | Nothing it's pointed at |

In its first use on a production codebase, it found **2 bugs** and corrected **3 wrong assumptions** in a single pass.

## Install

Copy the skill to your personal Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills
cp -r the-assumption-killer ~/.claude/skills/
```

Or clone directly:

```bash
git clone https://github.com/rashad/assumption-killer-skill.git ~/.claude/skills/the-assumption-killer
```

## Usage

The skill has three steps, run in order:

### Step 1: Generate assumptions

```
/the-assumption-killer generate
```

Assesses repo health (git history, code organization, documentation, test coverage), then walks through **every customer-facing flow** end to end. Flags uncertainties with `[UNCERTAIN]`, `[ASSUMPTION]`, and `[UNCLEAR]` markers. Outputs to your docs directory.

**You review the output and correct anything wrong before proceeding.** This human correction step is what makes it work — you know your business rules better than any prompt can describe.

### Step 2: Verify against code

```
/the-assumption-killer verify
```

Takes every assumption from Step 1 and verifies it against actual code. Each one gets classified:

- **Confirmed** — code does what the assumption says
- **Incorrect (code bug)** — the code doesn't do what it should
- **Incorrect (wrong assumption)** — the code is fine but the assumption was wrong
- **Partially correct** — some aspects confirmed, others not

Auto-detects high-risk items from uncertainty markers and investigates those deepest.

### Step 3: Fix issues

```
/the-assumption-killer fix
```

Presents actionable items from verification and asks which to fix. Addresses bugs, updates incorrect assumptions, and resolves action items in your chosen priority order.

## How it works

The core insight: **"audit my codebase" is an unbounded task. "Here are 10 things I believe are true — check each one" is a bounded, falsifiable task.**

1. **Give it something to be wrong about.** Assumptions with uncertainty flags create falsifiable claims to investigate.
2. **Human correction between steps.** You inject domain knowledge the AI can't infer — business rules, customer behavior, historical decisions.
3. **Parallel subagents for depth.** Each flow gets its own investigation agent. This enables exhaustive verification without cutting corners.
4. **Repo health calibration.** A well-maintained repo gets benefit of the doubt. A messy one gets deeper scrutiny.

See [methodology.md](methodology.md) for the full principles.

## Token warning

This skill is intentionally thorough. Both `generate` and `verify` use parallel subagents to investigate multiple flows concurrently. Expect significant token usage on large codebases. The tradeoff is explicit — shallow analysis is cheap but misses bugs.

## When to use it

- After AI generates significant code — verify it matches business intent
- Onboarding to a new codebase
- Before a major change (payment integration, auth overhaul)
- After inheriting code from another team
- Pre-launch audit

## Origin

Built from a workflow that reverse-engineered why assumption-driven verification works better than code auditing. The two-step chain (teach-back → verification) with human correction between steps consistently surfaces bugs that other approaches miss — specifically the kind where code runs fine but does the wrong thing.

## License

MIT
