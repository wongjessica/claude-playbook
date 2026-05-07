# Contributing

This repo is opinionated. Contributions are welcome, but the bar is high. The goal is to keep this useful, not to grow it indefinitely.

## What we want

**Workflow patterns** that pass these tests:

1. You've used it in real work, not just thought it up
2. You can describe a concrete situation where it applies and a concrete situation where it doesn't
3. You can name at least one failure mode you've personally hit
4. The pattern is generalizable beyond a single project or stack

**CLAUDE.md examples** that pass these tests:

1. The project shape is common enough that other engineers will recognize it
2. You're using a version of this in real work
3. You can articulate why each section is there and what's deliberately not there
4. It's opinionated, not a kitchen sink

**Decision guide additions** for surfaces or scenarios not yet covered. Same bar: real experience, clear reasoning, honest tradeoffs.

## What we don't want

- "Awesome list" style entries: link dumps without context
- Generic prompting tips you read elsewhere
- Patterns that work once but you can't reproduce
- AI-generated content without a human's hands-on validation
- Stack-specific optimizations that won't generalize

## How to contribute

1. Open an issue first if it's a new pattern or significant change. Describe the situation and the value before writing it up.
2. For typo fixes, broken links, or small improvements: open a PR directly.
3. New pattern PRs should include:
   - The pattern in the standard format (situation, pattern, why it works, worked example, failure modes, variations)
   - A note in your PR description about where you've used it
4. New CLAUDE.md examples should include:
   - The project shape
   - The CLAUDE.md content
   - Decisions and reasoning
   - What's deliberately not included
5. Be honest about what doesn't work. The failure-modes sections are often more valuable than the success patterns.

## Style

- No em dashes
- Prose over bullet lists where reasonable, but bullets when the structure is genuinely a list
- Concrete over abstract: "Snowflake permissions differ between dev and prod" beats "be aware of environment differences"
- Examples should be plausible, not necessarily literal. Don't paste your company's code.
- Keep sections short. If a pattern needs 1500 words, it's probably two patterns.

## Reviewing

PRs are reviewed for: usefulness, honesty, generalizability, and writing quality. We're more likely to ask for revisions than reject. Don't take notes personally; the goal is signal-to-noise.
