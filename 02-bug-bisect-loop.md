# Pattern: Bug Bisect Loop

## The situation

There's a real bug. You've described it to Claude. Claude has confidently suggested a fix. You apply the fix. The bug is still there. Claude suggests another fix. Same outcome. You're now three rounds in and Claude is starting to suggest things that contradict its earlier suggestions.

This is the most common Claude-coding failure mode that isn't talked about enough. It's not that Claude can't debug. It's that without structure, Claude debugs by guessing, and so do you.

## The pattern

Force a hypothesis-test loop with explicit roles.

1. **State the bug as observed behavior, not a theory.** "When I call `process_events` with an empty list, it raises a KeyError on line 47." Not "I think there's something wrong with how we handle empty lists."

2. **Ask Claude for hypotheses, plural.** *"Give me three plausible hypotheses for why this is happening. Rank them by likelihood. For each, tell me what evidence would confirm or rule it out."*

3. **Pick one hypothesis. Design the smallest test that falsifies it.** Critical word: falsifies. You're not looking for evidence the hypothesis is right, you're looking for evidence it's wrong.

4. **Run the test. Report the result verbatim.** Paste the actual output, not your interpretation.

5. **Update beliefs.** If the hypothesis survived, dig deeper into it. If it died, move to the next ranked hypothesis. Either way, narrate what you've learned: *"OK, so hypothesis 1 is dead because X. The remaining candidates are 2 and 3. Which test next?"*

6. **Repeat until you have one hypothesis with positive confirming evidence.**

## Why it works

Claude is excellent at proposing hypotheses, designing tests, and updating in response to evidence. Claude is poor at remembering, mid-conversation, that an earlier hypothesis was already ruled out.

The pattern fixes this by making the state of investigation explicit and external. After each test, you write down what's still alive and what's dead. Claude can read that summary and respond to it. Without that summary, Claude defaults to "what's the most plausible cause?" each turn, and the most plausible cause is often the same one you already ruled out.

The other critical move is "smallest test that falsifies." Most bug-hunting wastes time because tests confirm rather than falsify. A test that "shows the bug happens" is much weaker than a test that "shows the cause is X and not Y."

## Worked example

> **You:** Bug: in our event processor, when I run it against today's data, three rows are missing from the output that should be there. The rows have `event_type = 'click'` and `user_id IS NULL`. The processor doesn't error, it just silently drops them. Give me three hypotheses ranked by likelihood, with falsification tests for each.

> **Claude:** Three hypotheses:
> 
> 1. **A `WHERE user_id IS NOT NULL` filter exists somewhere in the pipeline.** This is the most common cause of "silent row drops on null fields." Test: search the codebase for `IS NOT NULL` and `user_id`, then trace each match to confirm whether it's on the path our rows take.
> 2. **A JOIN drops the rows because the join key is on `user_id` and null doesn't equal null in SQL.** Test: find the JOINs in this pipeline and check whether any join on `user_id`. If yes, replace one with a LEFT JOIN locally and re-run to see if the rows reappear.
> 3. **Pydantic or similar validation drops them before they're written.** Test: add a log line right before the write step and confirm whether all rows reach it. If they do, the issue is in the writer, not validation.

> **You:** Going with #1 first. Run the search and report back exactly what you find.

> **Claude:** *(searches codebase)*
>
> Three matches for `user_id IS NOT NULL`:
> - `analytics/dau.py:23` (not on our path)
> - `event_processor/clean.py:67` (on our path)
> - `tests/test_dau.py:12` (test fixture)
>
> The match in `clean.py:67` is inside a function `drop_invalid_users` that gets called from the main pipeline. The function's docstring says "Removes events from users that no longer exist." Hypothesis 1 looks alive.

> **You:** Confirm by removing that filter locally and re-running. Do the three rows show up?

> **Claude:** *(runs)*
>
> Yes. After removing the filter, all three rows appear in the output. Hypothesis 1 confirmed.

> **You:** Good. Now: is the filter wrong, or is something else creating events with null user_id that shouldn't exist? Don't fix yet. Investigate which case this is.

The pattern continues. The bug is caught not because Claude was clever, but because the loop forced rigor.

## Failure modes

**You let Claude skip to a fix.** Claude's natural inclination is to propose a fix as soon as it has a guess. Push back: "I want to understand the cause before we fix it."

**You don't paste the actual output.** Summarizing test output ("it didn't work") strips information Claude needs. Paste the literal error message, the actual stack trace, the real query result.

**You stop tracking what's been ruled out.** After three rounds, the list of dead hypotheses gets fuzzy. Maintain it explicitly. If your session has 20+ turns, write a one-line state summary every few rounds: "Confirmed: A. Ruled out: B, C. Still alive: D, E."

**The bug is environmental and you keep hunting in the code.** If three rounds of code-level hypotheses all die, broaden the search. Library version, environment variable, data shape change, infrastructure config. State this shift explicitly: "Code-level hypotheses are exhausted. Move up to environment."

**You start a new session and lose the state.** If you must start fresh (because the session is rotting), bring forward the dead-hypotheses list as your first message. Don't restart from scratch.

## Variations

**For flaky bugs:** add a hypothesis about timing or ordering. Falsification test is usually "run it 100 times and see if the failure rate matches our model."

**For cross-system bugs:** force Claude to draw the system boundary first. "Where could the bug live? List every component on the path from user click to database write." Then bisect across boundaries before bisecting within one.

**When Claude is stuck:** ask explicitly *"What evidence would change your mind?"* If Claude can't answer, the current hypothesis isn't a hypothesis, it's a belief.

**For "intermittent" bugs:** the first hypothesis is almost always wrong. Give them three rounds of skepticism before accepting any "I found it."
