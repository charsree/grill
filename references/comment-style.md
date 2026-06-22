# How the paste-ready comment must read

The whole point of the 7+ score is the user pastes it without editing. So it has to sound like the user, a normal engineer leaving a CR comment, not an AI.

## Rules

- **No em-dashes. No en-dashes used as em-dashes.** Comma, period, or parens. (The user calls this out every time.)
- **No AI-slop vocabulary:** delve, leverage, robust, seamless, ensure, facilitate, in order to, it is worth noting, consider refactoring, enhance maintainability, best practices, holistic, streamline, utilize, optimal, comprehensive, notably, furthermore, additionally, it should be noted, it's important to, moving forward. If you typed one, rewrite the sentence.
- **No hedge words that add nothing:** potentially, possibly, might want to, may want to consider, perhaps, arguably. Either you found a bug or you did not. "This can throw" not "this could potentially throw."
- **No praise sandwich, no preamble.** Do not open with "Nice work!" or "Great change overall." Just the point.
- **Short. One to three sentences.** A CR comment, not an essay. If it takes more than 3 sentences to explain, you do not understand the bug well enough.
- **Direct and concrete.** Name the line, the trigger, the fix or the question. Real reviewers ask a question or state the problem, they do not lecture.
- **Lowercase starts and casual phrasing are fine.** "this throws if list is empty" beats "This code will raise an exception in the event that the list is empty."
- **End with a question or a concrete suggestion when it fits.** "want a guard here?" / "should this be `>=`?" / "is empty reachable from the API path?"
- **Name the trigger.** The input, state, or condition that makes it fail. "when projectId is empty" not "in certain cases."
- **Name the consequence.** What actually goes wrong for the user/system. "returns 500 to the caller" not "may cause issues."

## Good (paste these)

> this throws on the empty-list path. the caller can hit it when the response has no items. guard it or return early?

> comment says this retries 3 times, but the loop runs `attempts < maxRetries` which is 0 by default here. comment's stale or the default's wrong.

> this timeout defaults to 0, which means no timeout in prod. was that meant to be the 30s from the config?

> this reads the value once at startup and caches it. if it changes while the process is up, you serve the stale one. probably fine here, flagging in case.

> the lock is held across the await on line 87. if the executor is single-threaded (which it is in tests), this deadlocks. move the await outside the lock scope?

> `processOrder` returns null on the timeout path (line 142), but the three callers all do `.orderId` on the result without null checks. timeout in prod = NPE.

> new field `priority` added to Task but not included in `Task.equals()` or `Task.hashCode()`. two tasks with different priorities will look equal in sets/maps.

## Bad (never)

> Consider refactoring this function to enhance readability and maintainability. (vague, bot, no line, no problem)

> Great work on this change! One small nit: it might be worth ensuring the list is non-empty to leverage a more robust approach. (praise sandwich, slop)

> This violates best practices. (which practice? where? what breaks?)

> This could potentially lead to issues in certain edge cases. (what issues? what cases? what breaks?)

> It's worth noting that this approach may not scale well. (at what scale? what happens? measured or guessed?)

## Format per finding (what the skill returns)

```
[8/10] handlers/upload.go:142  missing-file path returns success

Problem: the function returns nil (no error) when the file lookup misses, so callers treat a missing file as a successful empty upload. The doc comment above says it errors on a missing file, which is wrong.
Why it is real: read the function and its one caller; the caller has no separate not-found branch, so the miss is silently swallowed. The triggering input is any id that is not in the store. Survived 3 skeptics (Reachability, Guard-elsewhere, Mechanism-wrong).
Lenses that flagged it: STAMP (Class 1: required action not provided), Contract (postcondition violation)
Comment to paste:
> the comment says this errors when the file is missing, but it returns nil and the caller treats that as success. so a bad id looks like an empty upload. fix the comment, or return a not-found error here?
```
