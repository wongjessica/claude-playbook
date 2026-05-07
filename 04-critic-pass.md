# Pattern: Critic Pass

## The situation

Claude just produced something: a function, a doc, a query, a plan. It looks fine. It probably is fine. But you have that nagging feeling, and you're going to merge it, and if it's wrong it'll cost you tomorrow.

The problem is that Claude's first response is shaped by inertia toward producing output. Reviewing its own output requires a different stance, and Claude does it well only when explicitly asked to.

## The pattern

Force a review pass with a hostile stance. Three variants, increasing in rigor.

### Variant A: Same-session critic (cheap, fast)

After Claude produces something, in the same session:

> *"Now switch roles. You're a senior engineer reviewing this in code review. You are skeptical. List every issue, ranked by severity. Include things that are 'probably fine' as long as you can articulate the risk. Don't defend the original answer."*

That last line matters. Without it, Claude tends to review and conclude "this looks good." With it, Claude looks for problems.

### Variant B: Fresh-session critic (better, slower)

Open a new Claude session. Paste only the artifact, with minimal context. No history of how it was produced. Then ask:

> *"Review this [code/spec/query/plan]. Assume it was written by someone untrusted. List concrete issues you can find: bugs, edge cases, design smells, missing error handling, things the author didn't think about. Don't be polite. If it looks fine, say so explicitly with reasoning."*

This is better than variant A because the new session has no anchoring to the original generation. It reads the artifact fresh, the way a reviewer would.

### Variant C: Adversarial critic (best, slowest)

Two sessions, an adversarial pair.

Session 1 produced the artifact. In session 2, you paste the artifact and say:

> *"Try to break this. Concretely: list inputs, edge cases, or environmental conditions where this would behave wrong. For each, explain the failure and how confident you are it's actually a failure (vs. you misreading the code). Order by likelihood of occurring in production."*

Then take session 2's findings back to session 1:

> *"Another reviewer raised these concerns. Address each one. For each: confirm it's real and propose a fix, or argue why it's not actually a problem (with reasoning, not assertions)."*

You're now mediating between two stances. The output of this dance is significantly more robust than either session alone.

## Why it works

Generation and review are different cognitive tasks. When Claude is generating, it's optimizing for a coherent answer. When Claude is reviewing, it can optimize for finding problems, but only if you make that the explicit task.

The "same session" failure pattern is anchoring: Claude has already committed to an answer, and "reviewing" it tends to produce confirmation rather than critique. The fresh session doesn't have that anchor.

The adversarial variant works because real bugs hide in the gap between two perspectives. The author knows what they meant. The reviewer knows what's actually written. The disagreement between them is where the bugs are.

## Worked example

Claude has just written a SQL query for a dashboard, joining several tables to produce daily active users by region.

### Variant A (same session)

> **You:** Switch roles. Senior data engineer reviewing this query in code review. Be skeptical. List issues by severity, including "probably fine but here's the risk."

> **Claude:** Issues, ranked:
> 
> 1. **Severity: high.** The `LEFT JOIN` to `user_regions` could double-count users who have multiple region records (yes, this happens in our data). Without `DISTINCT` on user_id, the DAU count will be inflated for those users.
> 2. **Severity: medium.** Filter `WHERE event_date >= CURRENT_DATE - 7` uses session timezone. If this dashboard runs from a job in UTC and we report in EST, the boundary will be off by hours and "today's DAU" will look wrong in the morning.
> 3. **Severity: low.** No index hint or query plan check. For the date range we tested, this is fine. For longer ranges or larger tables, the planner may pick a hash join when a merge join would be better. Probably fine, worth noting.
> 4. **Severity: low.** Hardcoded region list in the CASE statement. Will silently bucket new regions as 'Other' until someone updates the query. Documented assumption, not a bug.

That's a real review. Issues 1 and 2 are bugs. Without the critic pass, they would have shipped.

### Variant B (fresh session)

A second Claude, with no context about the dashboard, reads the same query cold and might add: *"the CTE `daily_users` doesn't deduplicate within the day, so a user with multiple sessions counts as multiple events upstream. The downstream COUNT(DISTINCT user_id) saves you from this for DAU specifically, but if anyone ever copies this CTE for a different metric, they'll get inflated numbers."*

That's the kind of catch that comes from reading without context. It's a foot-gun for future readers.

### Variant C (adversarial)

> **Critic session:** "Try to break this query."
> 
> **Critic:** Inputs that break it: (a) a region rename mid-window (`'EMEA'` → `'EU'`), CASE doesn't bucket the renamed events; (b) backfilled data with `event_date` in the past but `created_at` recent, depending on which the WHERE clause uses; (c) a region with zero users in the window appears as 0 in some grouping engines and missing in others; the dashboard might silently drop it.
> 
> **Author session:** "Address each."
> 
> **Author:** (a) Real, will fix with a region-name mapping table. (b) Real, currently uses `event_date` which is the right field for our case, but worth a comment. (c) Real, need a `LEFT JOIN` to a regions list to ensure all regions appear.

You merge a much better query.

## Failure modes

**You don't actually read the critique.** It's tempting to run a critic pass, see "looks mostly fine," and merge. The pass is only valuable if you act on what it surfaces.

**Critic session has the same context.** If you paste the original prompt, the design discussion, and the code together, the critic anchors to the author's framing. Strip context. Just the artifact.

**Critic is too polite.** "This is well structured" is not useful. If the critic is producing validation, push harder: *"You have to find at least three concrete issues, even if minor."*

**Endless critique loops.** Two rounds of critic pass is usually enough. Three is the limit. Beyond that you're shaving yaks.

**Treating critic output as gospel.** The critic can be wrong. Especially in variant B, where it doesn't have full context, some "issues" are non-issues. The author session pushing back is part of the pattern, not a bug.

## Variations

**For specs and design docs:** the critic should ask "what's missing?" not just "what's wrong?" Specs fail by omission more than by error.

**For prompts:** if you're writing a prompt that will be used in a product, do a critic pass with a "find the failure mode" framing. *"What inputs would cause this prompt to produce a bad response?"*

**For your own work, not Claude's:** this pattern works equally well on human-written code. Paste your own draft and ask Claude for variant B. You'll catch things your own brain skipped.

**For PR review at speed:** variant A in the same Claude Code session that wrote the code is the fastest version. Use it as a default before every commit. The minute it costs is bought back the first time it catches a real bug.
