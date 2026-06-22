# grill

A deterministic, multi-lens adversarial code review skill for [Claude Code](https://docs.anthropic.com/en/docs/claude-code). Finds real bugs with zero false positives by design.

## How it works

```
Your diff
    |
    v
[Cynefin Triage] --- classifies change complexity
    |
    v
[7 Framework Lenses] --- parallel agents, each applying one analytical method
    |
    v
[Focused Dimension Agents] --- staleness, dead code, silent failure, test gaps, security, ...
    |
    v
[Adversarial Refutation] --- 2-3 skeptics per finding, each assigned a specific attack angle
    |
    v
[Calibrated Output] --- only findings that survived independent refutation score 7+
```

A finding scored **7 or higher is safe to paste as a review comment without reading the code yourself.** That's the bar. If it's below 7, the skill says so honestly.

## The lenses

| Framework | What it catches | Reference |
|---|---|---|
| **STAMP/STPA** | Missing guards, unsafe actions, feedback ignored, resources held too long | Leveson, "Engineering a Safer World" |
| **Pre-mortem** | The prod incident this diff will cause in 3 months | Klein, "Performing a Project Premortem" (HBR 2007) |
| **Inversion** | Assumed invariants that nothing enforces | Jacobi/Munger: "invert, always invert" |
| **Data-flow / Taint** | Untrusted data reaching trusting contexts without sanitization | Static analysis taint tracking |
| **Contract / Invariant** | Silent contract drift that breaks callers | Meyer's Design by Contract, Hoare logic |
| **Omission** | What's missing (the #1 class that escapes human review) | HAZOP "no/not" guideword |
| **Concurrency** | Races, atomicity violations, deadlocks, ordering bugs | Lamport, Herlihy & Shavit, RacerD |

## The refutation pass (why it has zero false positives)

Every finding gets attacked by 2-3 independent skeptic agents, each assigned a **specific attack angle**:

| Attack angle | What the skeptic tries to prove |
|---|---|
| Reachability | The triggering input can't actually reach this code path |
| Guard-elsewhere | A guard upstream already makes this safe |
| Type-system | The compiler/type system prevents this at build time |
| Pre-existing | This existed before the diff (out of scope for a change review) |
| Intended-behavior | This is documented, tested, deliberate behavior |
| Tooling-catches | A linter/test already catches this before ship |
| Mechanism-wrong | The proponent misread the code; the failure doesn't actually occur |
| Already-mitigated | A retry/fallback/timeout elsewhere already handles this |

A single valid kill (with cited code) collapses confidence to 0-1 regardless of how strong the original argument was. This is what eliminates false positives.

## Cynefin triage (sets review depth)

Not every change needs 7 lenses and 3 skeptics per finding. The skill classifies first:

| Tier | Example | Depth |
|---|---|---|
| **Clear** | Rename, format, config, dep bump | Dimension agents only. Light pass. |
| **Complicated** | Normal feature/bugfix, one module | 3 core lenses + dimensions. 2 skeptics per finding. |
| **Complex** | API change, concurrency, billing, migrations | All 7 lenses + dimensions. 3 skeptics per finding. |
| **Chaotic** | Incident hotfix | Blast radius + reversibility only. |

Escalates on the **single strongest signal**, never averages down.

## Output format

```
[8/10] handlers/upload.go:142  missing-file path returns success

Problem: the function returns nil (no error) when the file lookup misses,
so callers treat a missing file as a successful empty upload.
Why it is real: read the function and its one caller; the caller has no
separate not-found branch. Survived 3 skeptics (Reachability, Guard-elsewhere,
Mechanism-wrong).
Lenses that flagged it: STAMP (Class 1), Contract (postcondition violation)
Comment to paste:
> the comment says this errors when the file is missing, but it returns nil
> and the caller treats that as success. so a bad id looks like an empty upload.
> fix the comment, or return a not-found error here?
```

The paste-ready comment reads like a human wrote it. No AI slop, no praise sandwich, no "consider refactoring to enhance maintainability."

## Install

### Claude Code (local skill)

```bash
# Clone into your skills directory
git clone https://github.com/charsree/grill.git ~/.claude/skills/grill
```

Then invoke with `/grill` or any trigger phrase ("review my diff", "grill this", "check my changes").

### Claude Code (project skill)

```bash
# Clone into your project's skills directory
git clone https://github.com/charsree/grill.git .claude/skills/grill
```

## Usage

```
/grill                              # reviews unstaged + staged changes
/grill https://github.com/...       # reviews a GitHub PR
/grill CR-123456                    # reviews an Amazon code review
/grill path/to/file.kt             # reviews a specific file
```

## Design principles

1. **Precision over recall at 7+.** Below 7, it reports everything worth a look. At 7+, it must be airtight.
2. **Read beyond the diff.** Callers, types, and surrounding code are mandatory context. A finding based only on the diff is capped at 4.
3. **Verify, do not assume.** "X is never null" must be checked. Unverified assumptions cap confidence at 4.
4. **Never post.** The skill produces text. You decide what to paste.
5. **Honest coverage reporting.** States what it read, what it skipped, and where its limitations hit.

## Limitations (stated explicitly when hit)

- Functions over 100 lines with 20+ conditionals: variable state tracking degrades
- Cross-service distributed interactions: can't read both sides
- Subtle numeric precision/overflow: unless the language makes it explicit
- Build-system-resolved dependencies: the closure at build time may differ

The skill flags these in its coverage report rather than pretending it covered them.

## License

MIT
