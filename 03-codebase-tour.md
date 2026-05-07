# Pattern: Codebase Tour

## The situation

You've just been dropped into a codebase. New job, new team, inherited project, open-source contribution. You need to be productive fast, but the README is two paragraphs and the docs are a graveyard. You could read every file, but you won't, and you shouldn't.

## The pattern

Use Claude as a structured tour guide. Three layers, in order, each building on the last.

### Layer 1: The map (what is this thing?)

Open Claude Code in the repo. First message:

> *"You're going to give me a tour of this codebase. Don't read everything yet. Start by reading the README, package.json (or pyproject.toml, Cargo.toml, etc.), the top-level directory listing, and any obvious entry-point files. Then answer: what does this project do, who is the user, what's the tech stack, and what are the top-level components?"*

You're looking for the executive summary. If Claude can't produce one in three paragraphs, the codebase has a documentation problem you should be aware of.

### Layer 2: The arteries (how does data flow?)

Pick one user-visible feature. Something concrete: "a user logs in," "a webhook is processed," "a report is generated." Then:

> *"Trace what happens, in code, when [feature] occurs. Start from the entry point (HTTP route, queue consumer, CLI command, whatever). Walk through every file the request touches in order. For each file, name the function and explain what it does in one sentence. Stop at the boundary (database, external API, response to client)."*

This is the load-bearing step. Tracing one feature end-to-end teaches you the conventions of the codebase: where does configuration live, how is the database accessed, how are errors handled, what's the testing pattern. After this, every other file makes more sense.

### Layer 3: The questions (where would I touch this?)

Now ask things only useful once you've done layers 1 and 2:

- *"If I needed to add a new field to [the thing from layer 2], list every file I'd likely need to touch and in what order."*
- *"What's the test pattern in this repo? Show me a representative test and explain its structure."*
- *"Where is the riskiest code? The code most likely to break, or where bugs would be most expensive."*
- *"What conventions exist that aren't documented? Things you'd only know after reading a lot of files."*
- *"Are there parts of this codebase that look like they were written by different people with different opinions?"*

That last question is surprisingly useful. Codebases with style fractures usually have political fractures behind them. Worth knowing before you propose a "cleanup."

## Why it works

The naive approach is to load everything into Claude's context and ask "explain this codebase." It produces a generic summary that you could have generated yourself by reading the directory listing. It teaches you nothing.

The structured tour works because each layer answers a different question:
- Layer 1 answers *what is this?*
- Layer 2 answers *how does it work?*
- Layer 3 answers *how do I work in it?*

Skipping layers feels efficient and isn't. Layer 3 questions give bad answers if you haven't done layer 2, because Claude will speculate based on file names instead of reading the actual flow.

## Worked example

(Touring a hypothetical Python webhook processing service.)

> **You:** Layer 1, please. Start with README, pyproject.toml, top-level directory.

> **Claude:** *(reads files)* This is `webhook-router`, a Python service that receives webhooks from third-party services and dispatches them to internal handlers. It uses FastAPI for HTTP, Pydantic for validation, and Redis for a delivery queue. Stack is Python 3.11, FastAPI, Pydantic, redis-py, pytest.
> 
> Top-level components:
> - `app/` (FastAPI app, routes, request handling)
> - `handlers/` (one module per webhook source: stripe, github, etc.)
> - `queue/` (Redis-backed retry/delivery queue)
> - `models/` (Pydantic schemas for each webhook payload)
> - `tests/` (mirrors source structure)

> **You:** Layer 2: trace what happens when a Stripe webhook arrives.

> **Claude:** *(reads files in order)*
> 
> 1. `app/main.py:42` - `POST /webhooks/{source}` route receives the request. Calls `validate_signature()` on raw body before parsing.
> 2. `app/auth.py:18` - `validate_signature()` looks up the secret for the source, computes HMAC, compares to header. Raises `InvalidSignature` if mismatch.
> 3. `app/main.py:51` - Parses payload using `models/stripe.py:StripeEventEnvelope`. Validation errors return 400 immediately.
> 4. `queue/enqueue.py:23` - `enqueue_event()` writes the validated event to a Redis list keyed by source. Returns 202 to Stripe.
> 5. *(separate worker process)* `queue/worker.py:34` - `process_loop()` polls the Redis list, pulls one event, dispatches to the right handler.
> 6. `handlers/stripe.py:12` - `handle()` dispatches by event type to specific functions like `handle_payment_succeeded()`.
> 7. `handlers/stripe.py:28` - `handle_payment_succeeded()` calls our internal billing service via httpx, retries on 5xx.
> 
> Boundary: HTTP call to internal billing service.

> **You:** Layer 3: if I added a new webhook source (Linear), what files would I touch and in what order?

> **Claude:** Order matters here because each step depends on the previous one being merged or stubbed:
> 
> 1. `models/linear.py` (new file): Pydantic schemas for Linear's webhook envelopes
> 2. `app/auth.py`: add Linear's signature validation method (Linear uses a different scheme than Stripe, check their docs)
> 3. `tests/test_models/test_linear.py`: schema tests with sample Linear payloads
> 4. `handlers/linear.py` (new file): the dispatch function and at least one specific handler
> 5. `tests/test_handlers/test_linear.py`: handler tests using the schemas above
> 6. Configuration: add the Linear webhook secret to whatever config system this uses *(quick read of config.py: it pulls from env vars via Pydantic Settings, see `app/config.py:8`)*
> 7. Documentation: add Linear to the list of supported sources in README
> 
> The route in `app/main.py` doesn't need to change because it's already source-agnostic via the `{source}` path parameter.

You now know enough about this codebase to take a ticket. That took maybe 10 minutes.

## Failure modes

**Skipping layer 1.** Tempting because it feels obvious. But you'll be surprised how often Claude's layer 1 summary reveals something you wouldn't have spotted: a deprecated component, an unused dependency, a fork of an older version.

**Asking too many features at layer 2.** Pick one feature. Trace it deeply. The pattern works because depth on one path teaches you more than shallow coverage of many.

**Letting Claude speculate at layer 2.** If Claude says "this probably calls X" without reading X, push back: *"Read X and confirm."* The whole point is grounded answers.

**Not writing anything down.** After the tour, write yourself a half-page note. The conventions, the entry points, the components. You'll forget half of it within a day if you don't.

**Touring without a goal.** This pattern is for "I'm about to do real work in this repo." If you're just curious, a tour is overkill and you'll forget it.

## Variations

**For very large codebases:** layer 2 trace too long? Pick a smaller feature, or trace only one slice (only the request path, not the worker path).

**For abandoned/legacy codebases:** add a layer 0 question: *"What's the last meaningful commit, who wrote most of this, and is there evidence of an in-progress refactor that was abandoned?"* Git history tells stories the README won't.

**For frontend codebases:** layer 2 should trace a user interaction (click → state change → API call → re-render), not a backend request.

**When you're returning to a codebase you used to know:** skip layer 1, do a fast layer 2 on the feature area you'll be working on, and spend most of your time in layer 3 questions about what's changed.
