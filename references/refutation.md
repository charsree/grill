# Adversarial refutation pass (the calibrator)

Source: Kahneman's adversarial collaboration; Tetlock's forecasting work (seek disconfirming evidence, do not defend a position); LLM4PFA's precision methodology (ICML 2025). A single reviewer's confidence conflates "I believe this is a bug" with "this survived a real attempt to disprove it." Only the second earns a high score. This pass makes the confidence number mean "withstood N independent attempts from different angles to kill it", not "the reviewer felt sure".

## How to run it

After all framework + dimension reviewers produce candidate findings:

1. Pool everything. Dedupe by `file:line` + root cause (the same bug found by three lenses is one finding, not three; note it was multiply-flagged, that is corroboration).
2. For EACH finding, run 2-3 skeptic agents, each assigned a SPECIFIC attack angle (see below). Skeptics run in parallel. For Complex-tier changes or findings initially scored 6+, always run 3.
3. Each skeptic must cite specific code (file, line, function) for any kill. "It's probably fine" or "this seems unlikely" is NOT a valid refutation and does NOT lower confidence. An unsupported challenge is ignored.

## Attack angles (assign the most relevant per finding)

Each skeptic is assigned ONE primary angle. This prevents all skeptics from running the same generic pass.

| Angle | What the skeptic does | Best against |
|---|---|---|
| **Reachability** | Prove the triggering input/state/path cannot actually reach the buggy code. Trace from the system boundary through callers, input validation, type constraints, feature flags, guards. | Inversion/STAMP findings about dangerous inputs |
| **Guard-elsewhere** | Find a guard, precondition, invariant, or middleware enforced earlier in the call chain that makes this specific path safe. | Findings about missing validation |
| **Type-system** | Show the type system, borrow checker, nullability, exhaustiveness check, or ownership model already prevents this failure at compile time. | Null deref, wrong-type, exhaustiveness findings |
| **Pre-existing** | Prove this exact issue existed in the code BEFORE the diff. If the diff did not introduce or measurably worsen it, it is out of scope for a change review (may still be noted at Low). | Any finding, but especially staleness/dead-code |
| **Intended-behavior** | Show this is the documented, tested, deliberate, or contractually-required behavior. Find the test that asserts it, the doc that specifies it, or the design decision that chose it. | Pre-mortem / inversion findings that flag design choices |
| **Tooling-catches** | Show a compiler error, type-checker warning, linter rule (that is enabled and runs in CI), or existing test already catches this before ship. Identify the specific tool and rule. | Anything a linter/compiler/test covers |
| **Mechanism-wrong** | The proponent misread the operator, index bound, lifetime, interleaving, or control flow. The failure does NOT actually occur as described. Re-derive from the code. | Any finding where the "trigger" may be wrong |
| **Already-mitigated** | A retry, fallback, timeout, circuit breaker, idempotency key, or error handler elsewhere already handles this specific failure mode. Identify it by file:line. | Pre-mortem/STAMP findings about failure paths |

## Assignment rules

- **Every finding gets at least 2 skeptics with DIFFERENT angles.** Two skeptics on the same angle is wasted effort.
- **For Complex-tier or findings initially scored 6+: 3 skeptics minimum.**
- **Pick the angles most likely to kill the finding.** Think: "If this finding IS wrong, what is the most likely reason?" That reason is the angle.
- Common assignments by finding type:
  - Null/empty dereference: Reachability + Type-system + Guard-elsewhere
  - Race condition: Mechanism-wrong + Already-mitigated + Reachability
  - Missing error handling: Intended-behavior + Tooling-catches + Guard-elsewhere
  - Security/injection: Reachability + Guard-elsewhere + Pre-existing
  - Contract violation: Mechanism-wrong + Intended-behavior + Type-system
  - Omission (missing thing): Intended-behavior + Tooling-catches + Pre-existing

## Skeptic execution rules

1. **Read the code.** The skeptic MUST read the actual file, the surrounding function, the callers, and the types. A refutation based on "probably" or "usually" without reading code is invalid.
2. **Cite specific code for a kill.** File, line, function name, and WHY that code prevents the bug. "Line 32 validates input against a regex that rejects empty strings, so the empty-string trigger is unreachable" is a kill. "Input is probably validated somewhere" is not.
3. **Honest survival.** If the skeptic cannot kill the finding after genuinely trying its assigned angle, it must say "SURVIVED [angle]: I could not find evidence to refute this. [What I checked and did not find]." A finding that survives is STRONGER, not weaker, for having been attacked.
4. **No sympathy kills.** The skeptic's job is to KILL the finding. It must not soften ("well, it's technically possible but unlikely"). Either it found code that prevents the bug (kill) or it did not (survived).
5. **Independence.** Each skeptic works alone. They do not see other skeptics' results. They must reach their conclusion independently.

## Scoring (0-10)

| Outcome | Score | Bucket |
|---|---|---|
| Any skeptic killed it with cited code | 0-1 | Drop |
| Survived, but a real caveat remains (narrow reachability, needs a test to confirm, rests on an assumption not verified against real code) | 2-4 | Low / FYI |
| Survived 2 skeptics with different angles, concrete trigger, surrounding code actually read | 5-6 | Worth a look |
| Survived 3 independent skeptics from different angles, concrete trigger, root cause confirmed in real code, introduced by this diff | 7-8 | Paste-ready |
| Survived 3+ skeptics AND multiply-flagged by independent framework lenses (corroboration from different analytical methods) | 9-10 | Paste-ready (high corroboration) |

## Honesty rules (non-negotiable)

- A single valid refutation (with cited code) collapses confidence to 0-1, regardless of how strongly the proponent argued or how many other skeptics failed to kill it. One valid kill = dead.
- Confidence rises only with INDEPENDENT survival (different attack angles, different skeptics), not by re-running the same angle or having the proponent argue harder.
- Items "Tooling-catches" and "Already-mitigated" are anti-double-counting: if a gate or existing mitigation already covers it, cap the score even if the mechanism is technically real. The number tracks "is this worth the user's time to paste as a comment", not theoretical correctness.
- Anything not verified against the real surrounding code is capped at 4. 7+ asserts: I (the reviewer) read the code, the skeptics read the code, and the finding held under assigned-angle attack.
- **Corroboration boost**: if the finding was independently discovered by 2+ different framework lenses (not dimension checks, actual frameworks like STAMP + Inversion), add +1 to the score. Independent analytical paths converging on the same bug is strong evidence.
- Log, per finding: what each skeptic's assigned angle was, what they checked, and the outcome (killed with citation / survived). The score is an audit trail.

## The meta-check (after scoring)

After all findings are scored, the main agent does one final check:

1. **Are there any 7+ findings that feel wrong?** Read the code yourself. If your own reading contradicts the finding, demote it.
2. **Are there any 5-6 findings that, in combination, reveal a pattern?** (Three "Worth a look" findings that are all "forgot to handle the error case" may indicate a systematic pattern that deserves a single 7+ finding about the pattern itself.)
3. **Is the total finding count suspiciously high?** If >10 findings survive for a Complicated change, something is wrong (likely the skeptics were too lenient or the lenses are pattern-matching rather than reasoning). Re-examine the top findings critically.
