---
name: grill
description: "Use whenever the user asks to review code, review a diff, review a CR or PR, 'grill this', 'grill my diff', 'check my changes', 'review before I push', 'find bugs in this', or wants a thorough multi-lens code review. Spins up parallel reviewer agents (each applying one analytical framework -- STAMP/STPA, pre-mortem, inversion, data-flow/taint, contract/invariant, omission, concurrency, plus focused dimension specialists), runs a multi-angle adversarial refutation pass on every finding, and returns calibrated findings with exact file and line, the problem in plain language, and a ready-to-paste human comment. Never posts anything. Report-only. A finding scored 7 or higher is safe to paste without reading."
---

# Code Review

A deterministic, miss-nothing code review. It does not post, comment, or run `cr`/`gh`. It produces findings for the user to paste themselves.

## What the output must be

For every finding, return exactly this shape:

```
[<confidence>/10] <file>:<line>  <short title>

Problem: <one or two sentences, plain English, what is actually wrong and when it bites>
Why it is real: <the evidence from the code, the triggering input/state, why it survived refutation>
Lenses that flagged it: <which independent lenses found this, if multiple>
Comment to paste:
> <a normal human review comment, the way a real engineer types it on a CR>
```

Rules for the output, non-negotiable:

- **Never post.** No `cr`, no `gh pr comment`, no writing to any review system. You hand the user text; they post it.
- **The paste-ready comment must read like a person wrote it.** Short, direct, lowercase-ok, the way reviewers actually talk. No "Consider refactoring to enhance maintainability." Write "this drops the error on the empty-list path, caller Foo can hit it, want a guard here?" No marketing, no praise sandwich, no restating the obvious.
- **No em-dashes anywhere.** Use commas, periods, or parentheses. (User notices every time.)
- **Confidence is a 0-10 score and it means something specific** (see Step 6). A finding at **7 or higher is meant to be paste-without-reading**, so it must be airtight: a real defect, introduced by this change, with a concrete trigger, that survived 2+ independent refutation angles. If you are not that sure, it is below 7.
- **Sort findings by confidence, highest first.** Group: "Paste-ready (7+)" then "Worth a look (4-6)" then "Low / FYI (1-3)".

## Hard stances (the user's standing rules)

These shape every reviewer. Bake them in:

1. **Do not trust the diff description.** The PR/commit message says what the author *thinks* they did. Re-derive what the code actually does from the code.
2. **Do not trust comments.** A comment is a claim, not truth. Flag comment-vs-code mismatch as its own finding class.
3. **Flag the staleness/dead/mismatch classes explicitly**, these are first-class, not nits:
   - comment says X, code does Y (stale or lying comment)
   - doc/README/ADR says X, code does Y (stale doc)
   - code that nothing calls (dead/unused), or a path that can never run
   - a check that is now redundant, or a guard that no longer guards anything
   - a name that no longer matches behavior
4. **Read beyond the diff.** The diff alone hides bugs. Read the surrounding function, the callers, the type definitions, and the package conventions, to the depth the change's risk demands (see Triage). High-confidence findings REQUIRE having read the surrounding code, you cannot score 7+ on a guess.
5. **Verify, do not assume.** If a finding depends on "X is never null" or "this is only called from Y", go check. An unverified assumption caps confidence at 4.
6. **Trace through long functions.** When a function exceeds ~80 lines or has 15+ conditional branches, do NOT attempt to reason about it holistically. Instead: identify the variables the finding depends on, trace their state through each branch path explicitly, and note which path combinations reach the bug. This is the zone where review tools fail silently, so slow down.

## The pipeline (run in order)

### Step 1: Scope and gather

- Determine what to review. Default: unstaged + staged changes (`git diff` and `git diff --cached`). For a CR/PR, the commit range. Ask the user only if ambiguous.
- Capture the requirement: what was this change supposed to do? (from the user, the ticket, the commit). You will check the code against the requirement, not against the description's self-report.
- **Pull ticket context (mandatory if a ticket is named).** Scan the user's request, the commit messages, and the diff/branch for a ticket id (any `LETTERS-digits` key, e.g. `PROJ-1234`). If one is found, OR the user gives one manually, you MUST invoke the `jira` skill to read that ticket and use its description / acceptance criteria as the real requirement to review against. The code is judged against the ticket, not the PR's self-description. Jira ONLY, never SIM. If no ticket is found, proceed without one (do not invent or guess a key).
- Find the guidance files: root `CLAUDE.md` / `AGENTS.md`, any `CLAUDE.md` in modified dirs, `.rules/`, `.agents/`, language style configs. Note the paths; reviewers check the code against these.
- Read the changed files in full (not just the hunks), plus the immediate callers and the types involved.

### Step 2: Build structured context (the context package)

Before any reviewer fires, construct a context package that every reviewer agent will receive. This prevents agents from working with raw diff snippets (which research shows produces 40-72% accuracy) versus structured context (86-98% accuracy).

The context package includes:
1. **The diff itself** (with sufficient surrounding lines, at least 50 lines above and below each hunk).
2. **Caller map**: for each modified function/method, list its callers (at least the immediate ones). If a function has 10+ callers, note that and list the top 5 most relevant.
3. **Type context**: the full type definitions of parameters, return types, and struct/class fields used in the modified code.
4. **State transitions**: if the code modifies state (struct fields, global variables, DB writes, cache entries), note what other code reads that state.
5. **Contract summary**: extract from the types, doc comments, and tests what this code is supposed to guarantee. This is what the code CLAIMS, which reviewers will check against what it DOES.
6. **Cross-file impact**: if the change modifies a shared type, interface, exported function signature, or wire format, list the consumers.

Emit this as a structured block in the agent prompts. Do NOT ask agents to gather this themselves (they will take shortcuts).

### Step 3: Triage with Cynefin (sets depth)

Classify the change to decide how heavy the review is. **Escalate on the single strongest signal, do not average down.** See `references/cynefin-triage.md`. Outcome:

- **Clear** (rename, format, config, comment, dead-code removal, dependency bump, generated code): light pass. Run the focused dimension agents only, skip the heavy framework fan-out. Be honest that it is light.
- **Complicated** (normal feature/bugfix in one well-understood module, local effects): the starred framework lenses plus all focused dimension agents.
- **Complex** (touches a contract/API/wire format, concurrency/locking, a state machine, a shared/high-fan-in dependency, error/retry/timeout behavior, a migration or backward-compat, a security/trust boundary, money/quotas, or has large blast radius / low reversibility): ALL lenses. Reading the diff is NOT enough; probe the package.
- **Chaotic** (active incident hotfix): review only blast radius and reversibility, defer the rest, say so.

### Step 4: Parallel framework reviewers (the fan-out)

Launch reviewers IN PARALLEL (one Agent call per lens, all in a single message so they run concurrently). Each is a separate agent applying ONE lens, and returns candidate findings with `file:line`, the lens name, and the triggering condition.

**HARD REQUIREMENT, do not skip:** each reviewer agent's prompt MUST begin with an instruction to `Read` its lens reference file FIRST, before looking at any code, and to apply that exact method. Do not paraphrase the lens from memory and do not assume the agent knows the framework. The reference file IS the method; an agent that has not read it is not running the lens.

To locate reference files: resolve paths relative to this skill's base directory (the directory containing this SKILL.md file). For example, if this skill lives at `<SKILL_DIR>/SKILL.md`, then references are at `<SKILL_DIR>/references/<file>`.

Construct each agent prompt as:

> "First Read `<SKILL_DIR>/references/<file>` in full. That file is your review method, follow it exactly. Then review this change using the context package below. Read the surrounding code, the callers, and the types. Do not trust the diff description or any code comments. For every finding give file:line plus the concrete triggering input/state. For functions over 80 lines, trace variable state through branches explicitly. Return findings only, no preamble."

Where `<SKILL_DIR>` is replaced with the actual absolute path to this skill's directory at runtime.

Then include the context package from Step 2.

#### Framework lenses (the deep analytical passes)

| Lens | Reference file | Tier |
|---|---|---|
| STAMP/STPA control-hazard | `references/framework-stamp.md` | Complicated+ |
| Pre-mortem | `references/framework-premortem.md` | Complicated+ |
| Inversion | `references/framework-inversion.md` | Complicated+ |
| Data-flow / taint | `references/framework-dataflow.md` | Complex only |
| Contract / invariant | `references/framework-contract.md` | Complex only |
| Omission | `references/framework-omission.md` | Complex only |
| Concurrency | `references/framework-concurrency.md` | Complex only (or if ANY concurrency signal in triage) |

**Exception:** if the triage found ANY concurrency signal (lock, mutex, async, channel, shared mutable state, atomic, thread), run the concurrency lens regardless of tier.

#### Focused dimension agents (the targeted checks)

These are single-concern agents, NOT bundled into one. Each runs in parallel alongside the framework lenses. Each gets the context package and `references/comment-style.md` for output voice.

| Dimension agent | What it checks | Tier |
|---|---|---|
| Staleness | comment-vs-code mismatch, stale docs, names that no longer match behavior | All tiers |
| Dead code | unused functions/variables/imports, unreachable branches, redundant guards | All tiers |
| Silent failure | unchecked returns, empty catch, log-and-continue, swallowed errors, missing error propagation | All tiers |
| Test gap | is the risky path tested? does a new branch/error-case have a test? regression test for a fix? | Complicated+ |
| Guidance compliance | check against CLAUDE.md / AGENTS.md / .rules paths gathered in Step 1 | All tiers |
| Security surface | input validation, auth checks on new paths, secret handling, deserialization trust, injection vectors | Complicated+ |
| Requirement gap | what the ticket/requirement says vs what the code implements (missing acceptance criteria, partial implementation, scope drift) | Complicated+ (only if ticket context exists) |

If an agent reports it could not read its reference file, that lens did not run. Say so in the final coverage line rather than pretending it ran.

### Step 5: Adversarial refutation pass (the calibrator)

Pool all candidate findings from Step 4. Dedupe by file:line + root cause (the same bug found by multiple lenses is ONE finding; note it was multiply-flagged as corroboration).

Then for EACH surviving finding, launch skeptic agents. **HARD REQUIREMENT:** each skeptic agent's prompt MUST begin with "First Read `<SKILL_DIR>/references/refutation.md` in full, that is your refutation method and scoring rubric, follow it exactly."

#### Skeptic assignment (the key precision lever)

Do NOT send generic "try to kill this" skeptics. Each skeptic is ASSIGNED a specific attack angle from this list. Assign the 2-3 most relevant angles per finding:

| Attack angle | What the skeptic tries |
|---|---|
| **Reachability** | Prove the triggering input/state cannot actually reach this code path. Check callers, input validation, type constraints, feature flags. |
| **Guard-elsewhere** | Find a guard, precondition, or invariant enforced at a caller/earlier-in-the-chain that makes this path safe. |
| **Type-system** | Show the type system, borrow checker, nullability annotations, or enum exhaustiveness already prevents this. |
| **Pre-existing** | Prove this issue existed before the diff (not introduced or worsened by it). Out-of-scope for a change review. |
| **Intended-behavior** | Show this is the documented, tested, or contractually-required behavior (fail-open by design, intentional saturation, etc.). |
| **Tooling-catches** | Show a compiler warning, linter rule, type checker error, or existing test already catches this before ship. |
| **Mechanism-wrong** | The proponent misread the operator, index, lifetime, interleaving, or control flow. The failure does not actually occur as described. |
| **Already-mitigated** | A retry, fallback, timeout, idempotency key, or circuit breaker elsewhere already handles this failure mode. |

**Assignment rules:**
- Every finding gets at least 2 skeptics with DIFFERENT angles (even for Complicated tier).
- For Complex tier or findings initially scored 6+: 3 skeptics minimum.
- Pick the angles most likely to kill it. If the finding is about a null dereference, assign Reachability + Guard-elsewhere + Type-system. If it is about a race condition, assign Mechanism-wrong + Already-mitigated + Reachability.
- Skeptics run IN PARALLEL (all in one message).
- Each skeptic must cite specific file:line code for any kill. "Probably fine" is not a valid refutation.

### Step 6: Calibrate confidence (0-10)

| Outcome | Score | Bucket |
|---|---|---|
| Any skeptic killed it with cited code | 0-1 | Drop |
| Survived, but a real caveat remains (narrow reachability, needs a test to confirm, rests on an assumption not verified) | 2-4 | Low / FYI |
| Survived 2 skeptics with different angles, concrete trigger, surrounding code read | 5-6 | Worth a look |
| Survived 3+ independent skeptics from different angles, concrete trigger, root cause confirmed in real code, introduced by this diff | 7-8 | Paste-ready |
| Survived 3+ skeptics AND multiply-flagged by independent lenses (corroboration) | 9-10 | Paste-ready (high corroboration) |

**Honesty rules (non-negotiable):**
- A single valid refutation (with cited code) collapses confidence to 0-1, no matter how strong the original argument.
- Confidence rises only with INDEPENDENT survival (different attack angles, different skeptics), not repetition or enthusiasm.
- If a linter/compiler/test would already catch it, cap at 2. Not worth a human comment.
- Anything not verified against the real surrounding code is capped at 4. 7+ means the reviewer READ the code and the skeptics READ the code and it held.
- **Corroboration boost**: if 2+ independent lenses (different frameworks, not just dimension checks) flagged the same root cause, add +1 to the post-refutation score. Independent discovery from different analytical angles is evidence.
- Log, per finding, what each skeptic tried and the specific outcome. The score is an audit trail, not a vibe.

### Step 7: Synthesize (main agent, you)

You (main Claude) make the final call. Do not just concatenate agent output. For each surviving finding:
- Confirm the file:line is exact and points at the changed code (or the line the change makes wrong). **Read the file yourself to verify.**
- Write the plain-English problem and the paste-ready human comment yourself, in the user's voice (short, real, no em-dashes, no bot phrasing). See `references/comment-style.md`.
- Note which lenses flagged it (corroboration signal for the user).
- Order by confidence. Lead with the paste-ready (7+) set.
- If nothing is 7+, say so plainly: "Nothing I'd paste blind. N things worth a look." Do not inflate.

**End with an honest coverage report:**
```
Coverage: read [files]. Reviewed via [lenses that ran]. Did not read [what you skipped].
Limitations: [anything the review could not check, e.g. "did not run tests", "integration behavior across service X not traced", "function Y at 200 lines may have state-tracking gaps"].
```

## Determinism

Same diff in, same review out. Fixed lenses per Cynefin tier (not random). The triage tier, the lens set, the skeptic angle assignments, and the scoring rubric are all fixed. Variance comes only from the code, not from the tool's mood.

## Known failure modes (be explicit when you hit them)

These are cases where LLM-based review has proven limitations. When you encounter them, say so in the coverage report rather than pretending you covered them:

1. **Functions over 100 lines with 20+ conditionals**: variable state tracking degrades. Flag: "function X is long/complex, traced key paths but full state coverage not guaranteed."
2. **Cross-service distributed interactions**: unless you can read both sides, flag: "downstream behavior of service X not verified."
3. **Subtle numeric precision / overflow**: unless the language makes this explicit (Rust overflow panics, checked arithmetic), flag: "numeric precision not fully verified."
4. **Build-system-resolved dependencies**: the dependency closure at build time may differ from what you see locally. Flag when relevant.

## References

All paths are relative to this skill's directory (`<SKILL_DIR>/`):

- `references/cynefin-triage.md` - the triage classifier and depth rules
- `references/framework-stamp.md` - STAMP/STPA control-hazard lens (4 UCA classes)
- `references/framework-premortem.md` - pre-mortem lens
- `references/framework-inversion.md` - inversion lens
- `references/framework-dataflow.md` - data-flow / taint tracking lens
- `references/framework-contract.md` - contract / invariant lens
- `references/framework-omission.md` - omission lens (what is missing)
- `references/framework-concurrency.md` - concurrency / shared-state lens
- `references/refutation.md` - the adversarial skeptic method + assigned attack angles + scoring
- `references/comment-style.md` - how the paste-ready comments must read
