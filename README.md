# Assumption Killer

<!-- Banner -->
<p align="center">
  <img src="./assets/banner.svg" alt="Assumption Killer — Your code runs. But does it do the right thing?" width="100%"/>
</p>

<p align="center">
  <strong>Kills the untested assumptions hiding in your codebase.</strong>
</p>

<p align="center">
  <!-- TODO: uncomment when listed on skills.sh
  <a href="https://skills.sh/rashadsternes/assumption-killer">
    <img src="https://img.shields.io/badge/skills.sh-listed-3fb950?style=flat-square" alt="Listed on skills.sh"/>
  </a>
  -->
  <a href="./LICENSE">
    <img src="https://img.shields.io/badge/license-MIT-blue?style=flat-square" alt="MIT License"/>
  </a>
</p>

---

## The problem

AI writes code that runs. It compiles, passes linters, handles edge cases. But "runs correctly" and "does what you actually meant" are different things.

Somewhere between what you described and what got built, something slipped. The checkout flow applies a discount to items it shouldn't. The auth redirect silently drops data. The onboarding skips a step you assumed was there. No error. Tests pass. Everything looks fine.

This is **intent drift**. The code is technically correct but no longer matches what you wanted. The only way to find it is to make your beliefs about the code explicit, then check each one. That is what this skill does.

## Quick start

Install with one command:

```bash
npx skills add rashadsternes/assumption-killer
```

*Tip: run this in your terminal. The installer will ask one confirmation question.*

Or clone directly if you want to inspect the source before running:

```bash
git clone https://github.com/rashadsternes/assumption-killer.git ~/.claude/skills/assumption-killer
```

Then run the three steps in order:

```bash
# Step 1: Generate assumptions about every user facing flow
/assumption-killer generate

# Step 2: Verify each assumption against actual code
/assumption-killer verify

# Step 3: Fix what's wrong
/assumption-killer fix
```

That's it. Three commands. Every assumption checked.

## What it finds

The skill surfaces three categories of problems that other tools miss:

**Bugs in business logic.** Code that does the wrong thing but never throws an error. A discount applied where it shouldn't be. A permission check that grants access too broadly. The kind of bug that only shows up when a real user hits a real edge case.

**Wrong assumptions.** Beliefs about how the system works that were never true. You think the API validates input on the server. It doesn't. You think the cache invalidates on update. It doesn't. The code is fine. Your mental model was wrong.

**Intent drift.** Unique to AI assisted development. The code does exactly what was described in the prompt, but what was described in the prompt doesn't match what you actually needed. The AI built what you said, not what you meant. No linter catches this. No test covers it. The only way to surface it is the teach back: make the AI tell you what it thinks the code does, then correct it.

| Approach | What it catches | What it misses |
| --- | --- | --- |
| Linters and types | Syntax, type errors | Business logic gaps |
| Code review | Style issues, obvious bugs | Cross flow failures |
| Automated tests | Regressions in tested paths | Untested assumptions |
| **Assumption Killer** | Logic gaps, missing validation, intent drift | Nothing it's pointed at |

## How it works

The core insight: **"audit my codebase" is an unbounded task. "Here are 10 things I believe are true about this code. Check each one." That's a bounded, falsifiable task.**

### Step 1: Generate

The skill assesses repo health (git history, code organization, documentation, test coverage), then walks through every user facing flow end to end. It flags uncertainties with `[UNCERTAIN]`, `[ASSUMPTION]`, and `[UNCLEAR]` markers. Outputs to your docs directory.

**You review the output and correct anything wrong before proceeding.** This human correction step is what makes it work. You know your business rules, your edge cases, your "it does that on purpose" decisions better than any prompt can describe.

### Step 2: Verify

Takes every assumption from Step 1 and verifies it against actual code. Each one gets classified:

| Classification | Meaning |
| --- | --- |
| **Confirmed** | Code does what the assumption says |
| **Incorrect (code bug)** | The code doesn't do what it should |
| **Incorrect (wrong assumption)** | The code is fine but the assumption was wrong |
| **Partially correct** | Some aspects confirmed, others not |
| **Intent drift** | Code matches the prompt but not the actual intent |

High risk items from uncertainty markers get investigated deepest.

### Step 3: Fix

Presents actionable items from verification and asks which to fix. Addresses bugs, updates incorrect assumptions, and resolves action items in your chosen priority order. Nothing changes without your approval.

## When to use it

This works whether you're building a side project over a weekend or maintaining a production app with paying users.

Some good moments to run it:

- **After AI generates significant code.** Did it build what you meant, or what you said? Those are often different.
- **Before launch.** A verification pass catches the things that would otherwise become customer reports.
- **Before a major integration.** Payment flows, auth overhauls, third party APIs. The stakes justify the depth.
- **After inheriting a codebase.** The fastest way to learn what a codebase actually does versus what everyone thinks it does.
- **When something feels off but you can't point to why.** Trust that instinct. Make it concrete with assumptions, then check them.

## How it stays safe

Steps 1 and 2 are read only. The skill examines your codebase, generates assumptions, and verifies them against the actual implementation. It writes its findings to markdown files in your docs directory. No code modifications, no file deletions, no network calls.

Step 3 is the only step that proposes changes, and it presents every fix for your approval before applying anything. You pick which items to address and in what order. Nothing ships without you saying yes.

**What it does not do:**
- Install dependencies
- Modify your git history
- Make network requests
- Write to any directory outside your project
- Run your code or execute tests

**What it does do:**
- Read files across your codebase (it needs to, that's the whole point)
- Spawn subagents for parallel investigation (Claude Code subagents with the same sandboxing and permissions)
- Write markdown reports to your docs directory
- Use significant tokens (see the next section)

The entire skill is two files: [SKILL.md](./SKILL.md) and [methodology.md](./methodology.md). Read them before you install if you want. Or paste SKILL.md into a Claude conversation and ask "is this safe to run on my codebase?" We're not above being assumption killed ourselves.

## Token warning

This skill is intentionally thorough. The `generate` step uses parallel subagents to investigate flows concurrently. The `verify` step uses a three phase pipeline to scale to large codebases:

- **Phase 0 (Extract):** Agents pre extract focused code sections into temp files
- **Phase 1 (Verify):** Agents verify assumptions against pre extracted context, writing findings to files
- **Phase 2 (Compile):** One agent reads all findings and writes the final verification document

All agents write to files instead of returning results in context, keeping the main conversation lean. Expect significant token usage. The tradeoff is explicit. Shallow analysis is cheap but misses bugs. The token cost pays for itself in prevented customer facing issues.

## The methodology

The full principles are in [methodology.md](./methodology.md), but the short version:

1. **Give it something to be wrong about.** Assumptions with uncertainty flags create falsifiable claims to investigate.
2. **Human correction between steps.** You inject domain knowledge the AI can't infer. Business rules, customer behavior, historical decisions, "it does that on purpose" context.
3. **Parallel subagents for depth.** Each flow gets its own investigation agent. This enables exhaustive verification without cutting corners.
4. **Repo health calibration.** A messy repo gets deeper scrutiny. A clean one gets benefit of the doubt.
5. **Intent drift detection.** The teach back pattern surfaces the gap between what you described and what got built, the category of problem unique to AI assisted development.

## Origin

This skill was born from a coding session where everything looked right and wasn't.

The AI's code compiled, passed linting, handled edge cases. But it silently did the wrong thing. Not wrong enough to break. Wrong enough to cost money if it shipped. The workflow that caught it was simple: instead of asking "audit this code" (unbounded, vague), the approach was to write down specific beliefs about what the code was doing, then check each one against the actual implementation. Bugs surfaced. Wrong assumptions surfaced. And a new category appeared: intent drift, where the code perfectly matched the prompt but the prompt didn't match the intent.

This skill is that workflow, packaged so anyone can run it in three commands.

## FAQ

<details>
<summary><strong>I already have tests. Why do I need this?</strong></summary>

Tests verify that code does what the tests say it should do. This skill verifies that code does what the business says it should do. Those are different things.

Your test suite can have high coverage and still miss the case where a promo code applies a discount to items that should be excluded. Nobody wrote a test for that because everybody assumed the discount logic handled it correctly. This skill makes those assumptions visible, then checks them.

</details>

<details>
<summary><strong>What if it finds nothing?</strong></summary>

If it finds nothing, either your codebase is solid (worth knowing, that's a verified clean bill of health) or the assumptions were too vague (which is why you review and correct them at Step 1 before the real token spend at Step 2). Either way, you learn something.

</details>

<details>
<summary><strong>Will this work on my stack?</strong></summary>

The skill is language agnostic. It reads code and reasons about business logic. It doesn't parse ASTs or depend on language specific tooling. It works best on codebases with clear user facing flows: checkout, onboarding, data pipelines, API endpoints. It's less useful for pure libraries or framework internals where "business intent" is less defined.

</details>

<details>
<summary><strong>This is a new skill with a small install count. Why should I trust it?</strong></summary>

Three reasons. First, the code is open. SKILL.md and methodology.md are both public. You can read the entire skill in ten minutes and decide if the approach makes sense before installing. Second, the results are falsifiable. It either finds something in your codebase or it doesn't. There's no subjective quality to evaluate. Third, the cost of trying is low. One session on a codebase you know well. If the assumptions it generates are off, you'll know immediately at Step 1.

The skills ecosystem is young. Every skill with thousands of installs started at zero. Try it once and see what it finds.

</details>

<details>
<summary><strong>Isn't this just a fancy prompt?</strong></summary>

The core insight (check specific beliefs instead of auditing everything) is simple. The execution is where it gets nuanced. The skill handles repo health calibration, uncertainty markers that create falsifiable claims, parallel subagent architecture where each flow gets its own investigation agent, a three phase verification pipeline that scales to large codebases, and a structured human correction loop between steps. You could replicate the idea with a prompt. The skill handles the orchestration.

</details>

## License

MIT

---

<p align="center">
  <sub>Built by <a href="https://github.com/rashadsternes">Rashad Sternes</a></sub>
</p>
