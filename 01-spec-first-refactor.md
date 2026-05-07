# Pattern: Spec-First Refactor

## The situation

You need to rewrite a non-trivial piece of code. Not a tiny function, but something with real surface area: a module, a service, a complex query, a data pipeline. You've tried "refactor this" and Claude either changes too much, too little, or in directions you didn't want.

## The pattern

Don't ask Claude to refactor. Ask Claude to write a spec for the refactor first.

1. **Show Claude the current code.** Paste it or point at the file in Claude Code.
2. **State the goal in one sentence.** "Make this testable without a real database connection." Or "Split this into a parser and an executor." Specificity matters.
3. **Ask for a spec, not code.** Use language like: *"Before writing any code, write a short spec for this refactor. Include: the new shape (modules, classes, functions), what changes from the caller's perspective, what stays the same, what could go wrong, and the order I should implement the changes."*
4. **Read the spec. Push back.** This is the load-bearing step. Disagree, ask why, request alternatives. Edit the spec until it matches what you actually want.
5. **Now ask for the code, against the spec.** "Implement step 1 of the spec. Show me the diff. Don't move on until I confirm."
6. **Step through, one piece at a time.** Confirm each step works before the next.

## Why it works

Three reasons.

First, the spec exposes Claude's assumptions before they get baked into code. You can correct course in 30 seconds of reading prose rather than 10 minutes of reading code.

Second, the spec becomes a contract. If Claude drifts during implementation, you have something to point at: "step 3 of your spec said the parser would not depend on the executor."

Third, it forces Claude to think before it writes. Claude is significantly better at detecting design issues when asked to plan than when asked to implement. The first response to "write code" is shaped by inertia. The first response to "write a spec" is shaped by analysis.

## Worked example

> **You:** Here's our current `event_processor.py` (300 lines). Goal: make it testable without hitting Snowflake. Before writing code, give me a refactor spec.

> **Claude:** Spec for testability refactor of `event_processor.py`:
> 
> *Current shape:* one module, one class `EventProcessor`, with a `run()` method that opens a Snowflake connection, queries events, processes them, writes results back.
>
> *New shape:*
> - `EventSource` (protocol): defines `fetch_events() -> list[Event]`
> - `EventSink` (protocol): defines `write_results(results: list[Result]) -> None`
> - `EventProcessor`: takes an `EventSource` and `EventSink` in its constructor; the `run()` method has no I/O of its own
> - `SnowflakeEventSource`, `SnowflakeEventSink`: production implementations
> - `InMemoryEventSource`, `InMemoryEventSink`: test implementations
>
> *Caller changes:* production callers will need to construct the source and sink and pass them in. One-liner change in `main.py`.
>
> *What stays the same:* the SQL queries, the processing logic, the result shape.
>
> *What could go wrong:* the current code uses Snowflake-specific features (e.g., `MERGE INTO`) inside what looked like generic logic. We may discover during implementation that the boundary isn't clean.
>
> *Suggested order:*
> 1. Extract the protocols
> 2. Refactor `EventProcessor` to use them (no behavior change)
> 3. Move the Snowflake code into `SnowflakeEventSource/Sink`
> 4. Add the in-memory implementations
> 5. Write a test using the in-memory implementations to confirm the refactor preserved behavior

> **You:** I don't love `EventSource` as a name, our codebase uses `Reader/Writer`. Also flag step 3 as risky: that's where we're most likely to hit hidden coupling. Update the spec, then start step 1.

> **Claude:** *(updates spec, starts implementation)*

The session continues, one step at a time. If something surprising shows up in step 3 (it usually does), you and Claude update the spec together and continue.

## Failure modes

**Spec is too vague.** If the spec reads like "we'll improve error handling and split things up nicely," push for specifics. Names, signatures, file boundaries. If Claude resists specificity, the design isn't actually figured out yet.

**Spec is too detailed.** If the spec is 800 lines, you've gone too far. The spec should be readable in two minutes. Anything longer means Claude is writing pseudo-code instead of design.

**You skip pushback.** The pattern fails the moment you say "looks good, go" without actually engaging. The pushback step is where the value lives.

**You let Claude do all 5 steps in one response.** Don't. The whole point is that each step is a checkpoint. Claude will offer to "just do it all at once." Decline.

**The codebase fights the spec.** Sometimes step 3 reveals that the spec was based on wrong assumptions. That's the pattern working correctly. Update the spec and continue. Don't try to force the original spec through.

## Variations

**For tiny refactors:** skip the spec. The overhead isn't worth it for a 20-line function.

**For risky refactors:** add a "rollback plan" section to the spec. What's the path back if step 3 turns out to be a disaster?

**For unfamiliar codebases:** combine with the [Codebase Tour](./03-codebase-tour.md) pattern first. Don't refactor what you don't yet understand.

**For team work:** the spec doc itself becomes a PR description, a design review artifact, or a Linear ticket. Don't throw it away.
