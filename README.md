# Claude Playbook

> The senior engineer's notebook for getting absurd value out of Claude.

This is not an awesome list. There are no link dumps here. Every page is a tested pattern, a working config, or a hard-won lesson about what actually works (and what fails) when you use Claude as a daily tool.

## Who this is for

Engineers who already use Claude.ai, Claude Code, or the API and want to go from "this is cool" to "this is changing how I ship." If you're trying to figure out which surface to use, how to structure your CLAUDE.md, or why your long sessions keep going off the rails, you're in the right place.

## What's inside

### [Decision Guide](./decision-guide)
Stop wondering whether to open Claude.ai, run Claude Code, or hit the API. One flowchart, clear rules, and the tradeoffs nobody talks about.

### [Workflow Patterns](./workflow-patterns)
Named, repeatable patterns for the things engineers actually do: refactoring, debugging, learning a new codebase, code review. Each pattern has the situation, the steps, the failure mode, and a worked example.

| Pattern | Use when |
|---|---|
| [Spec-First Refactor](./workflow-patterns/01-spec-first-refactor.md) | You're rewriting something non-trivial and want Claude to stay on the rails |
| [Bug Bisect Loop](./workflow-patterns/02-bug-bisect-loop.md) | A bug is real but the cause is unclear and Claude keeps guessing wrong |
| [Codebase Tour](./workflow-patterns/03-codebase-tour.md) | New repo, no docs, need to ramp fast |
| [Critic Pass](./workflow-patterns/04-critic-pass.md) | Claude's first answer looks good but you suspect it's not |
| [Plan Then Execute](./workflow-patterns/05-plan-then-execute.md) | Multi-step task where Claude tends to skip ahead |

### [CLAUDE.md Cookbook](./claude-md-cookbook)
Real CLAUDE.md configs for real project shapes. Copy, adapt, ship.

| Config | Project shape |
|---|---|
| [Python Data Pipeline](./claude-md-cookbook/01-python-data-pipeline.md) | Snowflake, Airflow, batch ETL |
| [Next.js Full-Stack](./claude-md-cookbook/02-nextjs-fullstack.md) | App Router, Prisma, server actions |
| [Polyglot Monorepo](./claude-md-cookbook/03-monorepo.md) | Multiple stacks under one repo |
| [Rust CLI](./claude-md-cookbook/04-rust-cli.md) | Single binary, clap, tested |
| [Research Codebase](./claude-md-cookbook/05-research-codebase.md) | Notebooks, experiments, fast iteration |

## The principles behind everything here

1. **Context is the bottleneck, not capability.** Most "Claude got it wrong" moments are actually "Claude didn't have the context to get it right." Patterns here are mostly context engineering.

2. **Structure beats prompting.** A clean CLAUDE.md, a clear plan file, and a tight feedback loop will out-perform any clever prompt tweak.

3. **Short sessions beat long ones.** Long conversations rot. Most patterns here favor focused, scoped sessions over sprawling chats.

4. **Plans are cheap. Re-running is cheap. Bad code is expensive.** When in doubt, get Claude to plan first, then execute.

5. **Failure modes matter more than success patterns.** Knowing when not to use a pattern is often more valuable than knowing the pattern itself.

## Contributing

Got a pattern that works? See [CONTRIBUTING.md](./CONTRIBUTING.md). The bar is "tested in real work, with a concrete example, and you can articulate the failure mode."

## License

MIT. Use anything here however you want.
