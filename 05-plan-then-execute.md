# Pattern: Plan Then Execute

## The situation

You give Claude a multi-step task: "set up CI for this repo," "migrate this table from MySQL to Postgres," "add authentication to this API." Claude starts executing immediately, makes a few good moves, then hits a decision point and picks something you wouldn't have. Now there's a half-finished implementation with assumptions baked in and a session full of code you have to either accept, undo, or argue about.

The root cause is that Claude treats "do X" as permission to execute. For anything with branching decisions, it shouldn't.

## The pattern

Split into two phases with an explicit gate between them.

### Phase 1: Plan only

> *"I want you to plan a [task]. Do not execute anything. Do not write any code yet. Produce a plan with: (1) the steps in order, (2) the decision points where multiple reasonable approaches exist and which you'd pick with reasoning, (3) what could go wrong, (4) what you'll need from me before starting. After you give me the plan, wait. Don't proceed."*

The "wait, don't proceed" line is essential. Claude's default is to start executing as soon as it has a plan. You're explicitly inserting a gate.

### Gate: review and edit

Read the plan. The decision points are where the value is. For each one, either accept Claude's choice or override it. If you don't understand a step, ask. If a step seems too vague ("set up the database"), push for specifics.

### Phase 2: Execute, one step at a time

> *"Plan approved with these changes: [your edits]. Start step 1. Show me what you did before moving to step 2. If anything in the plan turns out to be wrong, stop and tell me, don't try to recover silently."*

That last clause is also important. Without it, Claude will sometimes silently adjust the plan when it hits a snag, and you only find out three steps later.

## Why it works

Three things are happening here.

First, Claude is a much better planner when planning is the explicit task. When the task is "do X," planning is incidental and rushed. When the task is "plan X," the entire response is shaped by analysis.

Second, the plan creates accountability. If step 4 goes wrong, you can reference the plan: "step 4 said this would do Y, but it did Z. What changed?" Without a plan, every disagreement becomes "well, I thought you wanted..."

Third, the gate is a forcing function on you. Reading a plan takes a few minutes. Reading a wrong implementation and figuring out what to change takes much longer.

## Worked example

> **You:** I want to add rate limiting to our FastAPI service. Plan only. Don't execute.

> **Claude:** Plan for adding rate limiting:
> 
> **Steps:**
> 1. Choose a rate limiter library and storage backend
> 2. Add the dependency to `pyproject.toml`
> 3. Create middleware or dependency that enforces limits per endpoint
> 4. Define limits (per-endpoint or global)
> 5. Add a way to identify clients (IP? API key? auth token?)
> 6. Add tests
> 7. Add metrics/logging so we can see when limits are hit
> 8. Document the limits in the API docs
> 
> **Decision points:**
> 
> - **Library:** I'd pick `slowapi` (FastAPI-friendly wrapper around `limits`). Alternatives: hand-roll with Redis, use a CDN/gateway upstream. Slowapi is the lowest effort and well-supported. *(Override here if you'd prefer a gateway-level approach, since that changes the entire shape of this work.)*
> - **Storage:** Redis (we already run Redis). Alternatives: in-memory (won't survive restart, won't work across instances), database (too slow). Going with Redis.
> - **Client identification:** I'd use the authenticated user ID where available, falling back to IP. Alternatives: always IP (simple but unfair behind NAT), always API key (breaks for unauthenticated routes). The hybrid is most fair.
> - **Limit values:** I don't know your traffic. I need you to tell me starting limits per endpoint, or I'll propose defaults and you can adjust.
> 
> **What could go wrong:**
> - Client identification fails behind a load balancer if X-Forwarded-For isn't trusted properly. We'll need to confirm `request.client.host` actually returns the real IP in your deployment.
> - Redis goes down. Decision: fail open (no rate limiting) or fail closed (block requests)? I'd default to fail open with loud alerts, but it's a judgment call.
> - Tests need a way to manipulate time/Redis state. We'll either use a fake clock or actually hit a test Redis.
> 
> **What I need from you:**
> - Confirm the library choice (or override)
> - Starting limit values per endpoint, or permission to propose defaults
> - Fail-open vs. fail-closed preference
> - Whether the existing auth middleware exposes user_id in a way we can hook into (I'll find out, but flagging)
> 
> Waiting for review before starting.

> **You:** Library is fine. Use defaults for limits, I'll tune later. Fail open. Skip step 8 for now, we'll do docs at the end. Start step 1.

> **Claude:** *(starts step 1, shows the dependency change)*

The session continues, one step at a time. When step 5 hits the X-Forwarded-For question, Claude flags it instead of guessing. That moment alone saved you a debugging session next month.

## Why this beats "just give it the task"

Compare with the version where you say "add rate limiting to our FastAPI service." Claude picks slowapi, picks IP-based identification, picks reasonable defaults, picks fail-closed because it sounds safer, and writes 300 lines of code. You read it. You realize you wanted user-based identification. You realize fail-closed will cause an outage if Redis blips. Now you're either accepting code you'd have written differently, or rewriting it, or having an annoying conversation about why Claude made those choices.

The plan-then-execute version surfaces those decisions in 90 seconds of reading.

## Failure modes

**Claude plans and immediately starts executing anyway.** Some Claude Code sessions, especially with auto-accept on, treat "plan" as a soft suggestion. Be explicit: *"Output the plan as text. Do not run any tools. Do not edit any files. Wait for my response."*

**The plan is too high-level.** "Set up the database" is not a step. If a step is more than ~30 minutes of work, it should be broken down further. Push: *"Step 3 is too coarse. Break it into substeps."*

**The plan misses decision points.** Claude will sometimes hide decisions inside steps, presenting them as obvious. Ask: *"For each step, is there a reasonable alternative I should consider? If yes, name it and tell me why you didn't pick it."*

**You skip the gate.** "Looks fine, go" defeats the entire pattern. The gate is the value.

**Step-by-step execution is slow.** Yes. The pattern is for tasks where wrong execution is more expensive than slow execution. For trivial tasks, skip the pattern.

**You don't update the plan when reality diverges.** When step 4 reveals that step 5 needs to change, update the plan visibly. *"OK, given what we found in step 4, the plan changes: instead of using X, we'll use Y. Updated plan: ..."* Otherwise the plan-as-contract value evaporates.

## Variations

**For exploratory work:** skip this pattern. If you don't yet know what you want, planning is premature. Use chat to think instead.

**For very large tasks:** the plan is a hierarchy. Top level has 5-8 steps. Some of those steps deserve their own sub-plans, produced just before that step starts.

**For high-trust mode:** if you trust Claude on this task and the cost of wrong execution is low, allow it to execute steps autonomously and only check in at "milestones" (every 3 steps, say). The pattern degrades gracefully.

**Combining with [Spec-First Refactor](./01-spec-first-refactor.md):** the spec is the artifact for refactor work, the plan is the artifact for setup/migration/integration work. Same family of moves, different framing.

**For agentic loops:** when Claude is going to run many tools in sequence (test, fix, retest), the plan is what keeps the loop from running away from you. *"Before each iteration, state what you tried, what you observed, and what you'll change."*
