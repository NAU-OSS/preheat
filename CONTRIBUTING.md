# Contributing to preheat

Thanks for looking. This file is the mechanics: where to put things, what I'll ask for, and what a contribution needs before it can merge. If you only read one paragraph, read the next one.

`preheat` is at the stage where the design is written and the code isn't. That makes some contributions more valuable than they'll ever be again. If you teach, tell me what a good student repo looks like. If you run a lab, tell me what your onboarding checklist is. If you've used cookiecutter or copier and have a grudge, write it down. Those go in [Discussions](https://github.com/NAU-OSS/preheat/discussions) under Ideas, and they will shape the built-in recipe more than any pull request could right now.

## Where things go

- **A bug, or something in the docs that's wrong:** an [issue](https://github.com/NAU-OSS/preheat/issues/new).
- **An idea, a question, or "should this exist?":** [Discussions](https://github.com/NAU-OSS/preheat/discussions). Ideas for proposals, Q&A for questions.
- **A change you've already written:** a pull request. Open an issue first if the change is more than a few dozen lines, so we can agree on the shape before you spend the time.
- **A security problem:** not a public issue. [SECURITY.md](SECURITY.md) has the private reporting path.

Every contribution is licensed under the same [MIT license](license.md) as the project. There's no contributor agreement to sign; opening a PR is the agreement.

## Reporting a bug

A good bug report lets me reproduce the problem without guessing. Include:

- What you ran, exactly. The full command, or the sequence of wizard answers.
- What you expected, and what happened instead. Paste the output; don't describe it.
- `preheat --version`, `uv --version`, and your OS.
- If a recipe was involved, the recipe (a link, or the `recipe.toml` inline). If a generated project was involved, its `.preheat.toml`, which records every answer.

If you're not sure whether something is a bug or a decision you disagree with, open the issue anyway and say that. I'd rather triage a non-bug than miss a real one. Issues get a `needs-triage` label when opened and lose it once I've looked; if yours has had the label for more than a week, a comment saying "still here" is welcome and not rude.

## Proposing a feature

Start a Discussion in Ideas, or open an issue with the `enhancement` label. Either way, the questions I'll ask are the same, so you can save a round trip by answering them up front:

- **Who is it for?** A student, an instructor, a lab, a recipe author, a maintainer of this repo. The answer changes what the default should be.
- **What does the wizard ask, and what does it do if the user just hits Enter?** If a feature adds a question, it needs a default and a paragraph of help text justifying that default. If you can't write the paragraph, that's a sign the feature should be a recipe rather than a question.
- **Is it in the "not building" list?** The [design doc](docs/design.md) has a short list of things I've decided not to do, with reasons. Proposals on that list need to argue with the reason, not just ask again.

Things I say yes to readily: recipes, better help text, anything that makes `preheat recipe check` catch more broken output, anything that removes a step from the class-assignment path. Things I say no to: new package managers, a web UI, engine plugins. Not because they're bad ideas; because this project is small on purpose.

## Contributing a recipe

Recipes are the contribution I most want and the one that needs the least Rust. A recipe is a `recipe.toml`, a `template/` directory of Jinja files, optional `hooks/`, and a `tests/` directory of answer fixtures. The format is documented in the [design doc](docs/design.md#recipes) for now and will get its own guide.

A recipe PR needs:

- `help` text on every question. One to three sentences that say why the default is the default. This is the single most-reviewed part of a recipe.
- At least one fixture in `tests/` per profile the recipe supports, and one for whatever combination of answers you think is most likely to break.
- `preheat recipe check ./your-recipe` passing. That renders each fixture and runs `uv sync --locked`, `uv format --check`, `uv check`, and `uv run pytest` inside the result. If any of those fail, the recipe doesn't merge, and CI will say so before I do.
- Templates that keep logic to a minimum. Conditionals belong in `recipe.toml` (`when = ...` on questions and files), not in nested `{% if %}` blocks inside templates. If a template needs more than a couple of conditionals, it's usually two files.

Recipes aimed at a specific course or department are welcome in this repo if they're general enough that someone else could adapt them. If they're truly specific, host them in your own repo and point `--recipe` at it; open a Discussion and I'll link to it from the docs.

## Contributing code

### Setting up

You need a Rust toolchain (stable, via [rustup](https://rustup.rs)) and [uv](https://docs.astral.sh/uv/). Nothing else. Then:

```
git clone https://github.com/NAU-OSS/preheat.git
cd preheat
cargo build
cargo test
```

For the Python package, which wraps the Rust core through PyO3:

```
uv sync
uv run maturin develop
uv run pytest
```

Git hooks are managed by [prek](https://github.com/j178/prek), a pre-commit reimplementation that doesn't need Python. `uv tool install prek && prek install` sets them up. They run the formatters and linters below, so if the hooks pass, the style checks in CI will too.

None of this works until the first code lands. When it does, this section will be tested against a fresh machine before the commands go in.

### Style

I'd rather the tools enforce style than have opinions in a document, so the rules are short.

**Rust.** `cargo fmt` with defaults, and `cargo clippy --all-targets -- -D warnings` clean. In `preheat-core`, no `unwrap()` or `expect()` outside tests; errors are typed with `thiserror` and returned. The core crate has no idea a terminal exists: anything that draws, prompts, or reads stdin lives in the `preheat` binary crate. If you find yourself importing ratatui in core, stop.

**Python.** `uv format` and `uv check`, which run Ruff and ty with the project's config. Public functions in the Python package have type hints and a docstring that says what the function does, not what its parameters are named.

**Templates.** Jinja, kept simple as described above. A comment at the top of any template file whose purpose isn't obvious from its path.

**TOML.** Questions in `recipe.toml` are ordered the way the wizard asks them. Every question has `id`, `prompt`, `default`, and `help`.

### Tests

- Anything in `preheat-core` that takes input and produces output has a unit test. Rendering is tested with snapshot tests (`insta`), so a change to a template shows up as a readable diff in review.
- Every change to the built-in recipe updates a fixture in `recipes/builtin/tests/` or adds one.
- The TUI is tested at the level of "given these keystrokes, these answers are collected." I don't expect pixel-level tests of the terminal, and I won't ask for them.
- CI runs `cargo test`, `cargo clippy`, `cargo fmt --check`, the Python tests, and `preheat recipe check` on every recipe in the repo. A PR needs green CI to merge. If you think a check is wrong, say so in the PR; sometimes it is.

If your PR changes behavior and adds no tests, I'll ask why. "It's hard to test" is an acceptable answer if you say what you tried.

### Documentation

- A change to the CLI updates `--help` text and the matching page in `docs/` in the same PR.
- A change to a design decision updates `docs/design.md`. That document is meant to be true, not historical. If you're overturning something it says, edit the section rather than appending a "however."
- Every PR that a user would notice gets a line under `Unreleased` in `CHANGELOG.md`, in the [Keep a Changelog](https://keepachangelog.com) format.
- Write documentation the way you'd explain it to someone sitting next to you. Second person, short sentences, say what to type. No feature lists that read like a brochure.

### Pull requests

- Branch from `main`. Small PRs merge faster; if you're doing two things, that's two PRs.
- Commit messages: a short imperative subject ("Add `when` support for directories"), and a body that says *why* if the diff doesn't make it obvious.
- Link the issue or Discussion the PR resolves. If there isn't one and the change is more than trivial, open one first; it's fine if it's one sentence.
- Draft PRs are welcome. Opening early and saying "not sure about the approach in `plan.rs`" gets you feedback before you've polished something that needs rethinking.
- I review every PR myself for now. I aim to respond within a week and usually manage a few days; if it's been longer, a ping is fine. One approval merges. I squash-merge, so don't worry about tidying your commit history.
- If you're new to the project, issues labeled `good first issue` are ones I've scoped to be finishable in an evening with no context beyond this file and the design doc. `help wanted` is for things I'd like done and won't get to soon.

## How we treat each other

This project has a [code of conduct](CODE_OF_CONDUCT.md), the Contributor Covenant, and it will be enforced. The short version: be kind, assume the other person is trying to help, and remember that the person on the other end of a review comment is probably a student doing this in the evening.

A few things specific to this project:

- **Disagreement about defaults is welcome and expected.** Half the design doc is opinions. If you think polars shouldn't be the default DataFrame library, say so, with a reason, and we'll have that argument in a Discussion where the next person can read it.
- **Non-code contributions are contributions.** Someone who writes a clear bug report, tests a recipe on Windows, or rewrites a confusing paragraph of help text is credited the same as someone who wrote Rust. Every release's notes list everyone who contributed to it, in any form.
- **Nobody is expected to be around.** This is a side project for me and it's a side project for you. If you start something and can't finish it, say so in the PR and someone else can pick it up. Unfinished work with a note is more useful than a PR that goes silent.
- **Reviews are about the change, not the person.** I'll ask for changes bluntly and I'll expect the same in return. If a review comment reads as anything other than "here's a way this could be better," I got the tone wrong; tell me.

Questions about any of this: open a Discussion. If something in this file is unclear, that's a documentation bug, and a PR fixing it is exactly the kind of first contribution I'm hoping for.
