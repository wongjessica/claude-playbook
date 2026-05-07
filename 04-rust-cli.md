# CLAUDE.md for a Rust CLI

## Project shape

- Single Rust binary, distributed via Cargo and prebuilt releases
- `clap` for argument parsing
- Integration tests using `assert_cmd` and `predicates`
- Probably the kind of tool people install with `cargo install` or download from a GitHub release page

This shape is common for developer tools: linters, formatters, build helpers, terminal apps.

## The CLAUDE.md

```markdown
# Project Context

This is `[tool-name]`, a CLI tool that [does X]. Distributed as a single binary via crates.io and GitHub releases. Used by developers in their local terminals and in CI.

## Stack

- Rust, edition 2021, minimum supported version (MSRV) is 1.75
- `clap` v4 with derive macros for argument parsing
- `anyhow` for application errors, `thiserror` for library-style errors in `lib`
- `serde` + `serde_json` for serialization
- `tokio` only if we need async (we currently don't, this is sync)
- `tracing` for diagnostics, `tracing-subscriber` configured to write to stderr

## Project layout

- `src/main.rs` - argument parsing and the dispatch to subcommands
- `src/cli.rs` - `clap` definitions
- `src/lib.rs` - public library surface (re-exports the modules we want to be testable)
- `src/<command>.rs` - one file per subcommand
- `tests/` - integration tests using `assert_cmd`

We have a `lib.rs` even though we ship a binary, because integration tests benefit from importing pieces of the binary's logic.

## Conventions

- Run `cargo fmt` before commit. CI fails on unformatted code.
- Run `cargo clippy -- -D warnings` before commit. Clippy warnings are errors.
- Public functions in `lib.rs` get doc comments. Private helpers don't need them.
- Prefer `&str` over `&String`, `&Path` over `&PathBuf`. Use the unsized form.
- Errors: in `main` and CLI dispatch, use `anyhow::Result`. In `lib.rs` modules that might be reused, use `thiserror`-defined enums.
- Don't `unwrap()` or `expect()` in non-test code unless there's a clear invariant comment immediately above explaining why it can't fail.

## CLI design rules

- Every subcommand has `--help` text. Write it as if explaining to someone who's never used the tool.
- Output to stdout for "the result", stderr for "diagnostics". Users will pipe stdout.
- Support `--json` on commands where it makes sense, for scripting use.
- Default output should be readable in a terminal: colors when stdout is a TTY, plain when piped. Use `clap`'s `--color` flag and the `is-terminal` crate.
- Exit codes: `0` for success, `1` for normal errors, `2` for usage errors (clap handles this), `>2` for tool-specific failures (document them).

## Testing

- Unit tests in the same file as the code they test, in a `#[cfg(test)] mod tests` block
- Integration tests in `tests/`, one file per subcommand (`tests/cmd_foo.rs`, `tests/cmd_bar.rs`)
- Use `assert_cmd::Command::cargo_bin("[tool-name]")` to invoke the binary
- Use `predicates` for output assertions
- Snapshot tests for help text and complex output (via `insta`)
- Run all tests: `cargo test`
- Run integration only: `cargo test --test '*'`

## Performance

- This tool runs interactively. Cold start matters.
- Avoid bringing in heavy dependencies. Each one adds compile time and binary size.
- Profile with `cargo flamegraph` before optimizing. Don't guess.

## Releases

- We use `cargo-release` for version bumps and tags
- GitHub Actions builds prebuilt binaries for macOS (x86 + ARM), Linux (x86 + ARM), Windows
- Release process: `cargo release patch --execute` (or `minor`/`major`)
- The CHANGELOG is hand-maintained in `CHANGELOG.md`, follow Keep-a-Changelog format

## What not to do

- Do not add async unless we have a real I/O parallelism need. Async adds complexity that's not paying for itself in this codebase.
- Do not add a heavy framework (e.g. swap clap for something else). clap is fine.
- Do not panic except via `unreachable!()` for impossible cases or `expect()` with a clear invariant. User-facing errors are returned, not panicked.
- Do not use `println!` for diagnostics. Use `tracing::info!` etc., which go to stderr.
- Do not check for `tty`-ness manually. Use the `is-terminal` crate.
- Do not break the public CLI interface in patch releases. Breaking changes are minor or major bumps with a CHANGELOG entry.

## When in doubt

- Look at how an existing subcommand handles the same thing
- For new dependencies, prefer crates that are already in `Cargo.lock` indirectly (saves compile time and disk)
- For error messages: write the message a confused user would want to read, not the message convenient for the developer
```

## Decisions and reasoning

**The "lib.rs even though we ship a binary" pattern is called out explicitly** because Claude won't always default to it, and if you let Claude split things ad hoc, you'll end up with logic duplicated between `main.rs` and tests. The pattern saves real pain later.

**CLI design rules are their own section.** Most CLAUDE.md examples online don't have one because most projects aren't CLIs. For a CLI, this is half of the user-facing quality. Stdout-vs-stderr discipline alone is a recurring source of bugs in CLI tools.

**Performance gets one paragraph, with the key principle: profile, don't guess.** This pre-empts the failure mode where Claude suggests a "performance improvement" that's actually neutral or harmful.

**The "What not to do" section is heavy on temptations specific to Rust.** Adding async, swapping the arg parser, panicking instead of returning errors. These are real things Claude will propose if not warned off.

**Releases get a section because most CLI tools don't release often enough to remember the steps.** Documenting the release process in CLAUDE.md means Claude can help you do a release without you re-learning the ritual.

## What's deliberately not included

- **A `Cargo.toml` listing.** Claude reads it on demand.
- **Specific subcommand documentation.** Each subcommand has its own `--help`. Pre-loading would duplicate that and stale fast.
- **CI configuration.** Lives in `.github/workflows/`. Mentioning it in CLAUDE.md is enough.
- **Architecture diagrams.** A CLI of this shape doesn't need one. If yours has plugins, scripting, or daemon mode, add a small architecture section.
- **Benchmarking setup.** Until performance is a measured problem, the "profile, don't guess" line is enough.

## How to adapt this to your project

- If your CLI is async (e.g. it talks to a server, it does parallel I/O), flip the async stance and document the async runtime patterns
- If your tool is library-first and CLI is a thin wrapper, swap the emphasis: more on the `lib.rs` API, less on CLI design
- If you support plugins or extensions, that's a whole section worth documenting
- The CLI design rules are general best practices for terminal tools. Even if some don't apply to your project, the discipline of having a section forces you to think about the user-facing surface

## A note on Rust-specific Claude behavior

Rust is a language where Claude is often *correct but unidiomatic*. The compiler will catch many errors, but Claude will sometimes:

- Reach for `Box<dyn Trait>` when generics would be cleaner
- Use `unwrap()` in code that should propagate errors
- Allocate strings unnecessarily (`String` where `&str` would do)
- Write loops where iterator chains would be clearer (or vice versa)

Calling out these tendencies in your CLAUDE.md ("prefer `&str`, prefer iterator chains for transforms") nudges output toward idiomatic Rust without you having to correct it every session.
