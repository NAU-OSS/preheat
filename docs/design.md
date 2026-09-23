# Design

This is the reasoning behind `preheat`, written before most of the code exists. Some of it will turn out to be wrong. When it does, I'll update this document rather than let it drift, because the point of writing it down is so that someone who joins the project in six months can tell the difference between a decision and an accident.

## The problem, stated narrowly

Setting up a Python data science project is a sequence of about a dozen decisions, most of which have a right answer in 2026 and none of which the tools make for you. Which Python. Whether to use a `src/` layout. Where data lives and how it stays out of git. Which formatter, which linter, which type checker, how they run. Whether notebooks are `.ipynb` or something that diffs. What goes in CI. What a README should say before there's anything to say.

Individually these are ten-minute questions. Together they're an afternoon, and the afternoon repeats every time you start a project, help a lab mate start one, or teach a class where thirty people need identical repositories on day one. The cost isn't the time exactly; it's that the setup gets skipped or half-done, and the project spends the next three months paying for it.

There are existing tools for this. The section near the end explains why I'm not using them. The short version is that they were designed before uv, and uv moved the floor.

## Constraints that shaped the design

**The tool must not need Python to run.** The machine that most needs a scaffolding tool is the one where Python is broken: three interpreters, a conda that was never activated, a `pip` that points somewhere surprising. A wizard that requires a working environment before it can help you build a working environment has failed the people it's for. This one rule drives most of the architecture.

**Contributing a template must not require reading Rust.** The valuable part of this project, long-term, is the collection of recipes: what a good class repo looks like, what a good lab repo looks like, what a specific department's conventions are. The people who know those things are instructors and researchers, not systems programmers. If the contributor path runs through a Rust build, there will be no contributors.

**The output must work on the first `uv sync`.** Not "mostly works," not "works after you fix the Python version." A recipe that produces a broken project is a bug, and the project needs a mechanism that makes that bug show up in CI rather than in a student's terminal.

**Opinionated defaults, every one of them explained.** The wizard should be fast for someone who accepts the defaults and educational for someone who doesn't. Every question has a `?` that says why the default is the default. If I can't write that paragraph, the default is wrong or the question shouldn't exist.

## Architecture

### Rust owns the executable

`preheat` is a Cargo workspace with two crates:

- `preheat-core`: the recipe schema, the question graph (including which questions to show given earlier answers), the file plan (which template files to render given the answers), rendering, and orchestration of `uv`.
- `preheat`: the binary. A [ratatui](https://ratatui.rs) TUI for the interactive path and a `clap` CLI for the non-interactive one. Both call into `preheat-core`; the TUI is a thin layer that collects answers and shows a preview.

This gives a single static binary that starts in milliseconds and cross-compiles for Linux, macOS, and Windows with `cargo-dist`. I looked hard at [Textual](https://textual.textualize.io/), which is the best Python TUI framework by a wide margin, and would have been more fun for me to write. But Textual needs an interpreter and an environment before the first frame draws, and that violates the first constraint. If the "no Python required" rule ever gets relaxed, Textual is the obvious front end to add.

### Templates are Jinja, not Rust

Rendering uses [minijinja](https://github.com/mitsuhiko/minijinja), which implements Jinja2 syntax closely enough that a cookiecutter template author can read a `preheat` template without learning anything. This is the second constraint in practice. The files a contributor touches are `recipe.toml`, a `template/` directory full of Jinja, and optionally a Python hook. Rust is for the handful of people maintaining the engine.

I considered [Tera](https://keats.github.io/tera/), which is more idiomatic in the Rust ecosystem, and rejected it for exactly one reason: its syntax diverges from Jinja in small ways that would bite template authors coming from cookiecutter. Familiarity beats idiom here.

### Where Python lives

Python appears in three places, and in all three it arrives through uv.

1. **Hooks.** A recipe may include `hooks/pre_gen.py` (validate answers, fail early) and `hooks/post_gen.py` (anything that's easier in Python than in a template: renaming files, initializing git, writing a first notebook). Hooks are [PEP 723](https://peps.python.org/pep-0723/) inline-metadata scripts, and the engine runs them with `uv run --script`. uv fetches an interpreter and the hook's dependencies into its cache; the hook author never creates an environment and the user never sees one.

2. **The Python package.** `preheat` on PyPI is built with [maturin](https://www.maturin.rs/) and ships the binary alongside a thin PyO3 module exposing the core API: load a recipe, validate answers, render to a directory. This is how `uvx preheat` works, and it's the batch path for instructors: loop over a roster, call `preheat.render(...)` forty times, push forty repos.

3. **The generated project.** Obviously. Everything below about defaults is about this layer.

### uv does the Python-shaped work

The engine never touches `pip`, never creates a venv, never downloads an interpreter. It calls `uv init`-adjacent operations (the recipe writes `pyproject.toml` directly, then calls `uv lock` and `uv sync`), `uv python list` to populate the version picker with what's actually available, `uv run --script` for hooks, and `uv format` / `uv check` when validating a recipe. If uv gains a capability, `preheat` gains it for free. If uv changes a default (as it did in 0.12, when `uv init` switched to a `src/` layout and the `uv_build` backend), the built-in recipe tracks it.

## Recipes

A recipe is a directory:

```
my-recipe/
├── recipe.toml
├── template/
│   ├── pyproject.toml
│   ├── README.md
│   ├── src/{{ package_name }}/__init__.py
│   └── ...
├── hooks/
│   ├── pre_gen.py     (optional)
│   └── post_gen.py    (optional)
└── tests/
    ├── defaults.toml
    └── class-minimal.toml
```

`recipe.toml` declares questions, conditional files, and metadata:

```toml
[recipe]
name = "nau-analysis"
description = "Data science project, the way we do it at NAU"
min_preheat = "0.1"

[[question]]
id = "profile"
prompt = "What kind of project is this?"
type = "choice"
choices = ["class", "lab", "package", "minimal"]
default = "lab"
help = """
Sets defaults for everything that follows. 'class' strips CI, hooks, and
docs. 'package' adds a build backend and publishing config. You can
override any individual default on later screens.
"""

[[question]]
id = "dataframe"
prompt = "DataFrame library"
type = "choice"
choices = ["polars", "pandas"]
default = "polars"
help = "Polars is faster and stricter about types. Pick pandas if your course materials use it."

[[question]]
id = "dvc"
prompt = "Track data with DVC?"
type = "bool"
default = false
when = "profile == 'lab'"

[[file]]
path = "template/dvc.yaml"
when = "dvc"

[[file]]
path = "template/.github/"
when = "ci"
```

Three things here that cookiecutter can't express: a question shown only when an earlier answer makes it relevant (`when` on a question), a file or directory included only under some answers (`when` on a file), and structured help text attached to the question rather than buried in a README. Copier has the first two. Neither has the fourth thing, which is the reason this project exists:

```toml
[[question]]
id = "python"
locked = "3.13"

[[question]]
id = "notebook"
locked = "marimo"
```

A locked question is not shown and its value is fixed. An instructor forks the built-in recipe, locks the answers that matter for the course, and publishes it. Students run `uvx preheat new --recipe gh:NAU-OSS/sta574-f26 hw1` and answer three questions. Every submission has the same shape, and the instructor's "how to set up your environment" document becomes one line.

Answers are written to `.preheat.toml` in the generated project. That file makes a scaffold reproducible (`preheat new --answers .preheat.toml`), and it's the hook for a future `preheat update` that re-renders a project against a newer version of its recipe.

## The wizard

Six screens. Each has a `?` key that opens the help text for the focused question. Enter accepts the default and moves on, so a user who trusts the recipe gets through in six keystrokes.

1. **Profile.** The one question that sets all the other defaults.
2. **Identity.** Project name, package name (derived, editable), one-line description, author (pre-filled from `git config`).
3. **Python and tooling.** Python version (choices come from `uv python list`), whether to install git hooks, whether to add a `justfile`.
4. **Analysis stack.** DataFrame library, notebook tool, plotting library, data versioning.
5. **Plumbing.** Tests, docs, CI, devcontainer, license, community files.
6. **Review.** A file tree of what will be written and the rendered `pyproject.toml`, side by side. Nothing is written until this screen is confirmed.

Then the engine renders, runs `uv lock` and `uv sync`, runs `post_gen.py` if present, and prints what to do next.

### What the wizard asks, and what it doesn't

| Question | Options | Default | Reasoning |
|---|---|---|---|
| Profile | class, lab, package, minimal | lab | Sets everything else. |
| Python version | from `uv python list` | newest with full wheel coverage for numpy/polars/pandas | Pinned in `.python-version`; `requires-python` set one minor lower so the project is installable by people slightly behind. |
| DataFrame library | polars, pandas | polars | Lazy execution, real types, much faster. pandas stays because that's what most courses teach; either way pyarrow is included. |
| Notebooks | marimo, Jupyter, none | marimo | marimo notebooks are `.py` files that diff cleanly and have no hidden execution order. Choosing Jupyter adds jupytext pairing for the same reason. |
| Plotting | matplotlib+seaborn, plotnine, altair | matplotlib+seaborn | The default is the one every tutorial assumes. plotnine is there specifically for people coming from R and ggplot2. |
| Data versioning | none, DVC, git-lfs | none for class, DVC for lab | DVC is the right tool for a lab and a burden for a homework assignment. |
| Git hooks | prek, none | prek for lab, none for class | [prek](https://github.com/j178/prek) is a Rust reimplementation of pre-commit with no Python dependency, installable via `uv tool`. Hooks are the right discipline for a lab and a confusing surprise in the first week of a course. |
| Task runner | just, none | just | `just setup` is easier to teach than a paragraph of commands. `rust-just` is on PyPI, so `uv tool install rust-just` works. |
| CI | GitHub Actions, none | on for lab and package | `astral-sh/setup-uv`, then `uv sync --locked`, `uv format --check`, `uv check`, `uv run pytest`. |
| Docs | none, mkdocs-material, quarto | none | Quarto is offered because statisticians want reports, not API docs. |
| Devcontainer | yes, no | no | Useful for classes on shared or locked-down machines. |
| License | MIT, Apache-2.0, BSD-3-Clause, none | MIT | |
| Community files | full, minimal | full for lab, minimal for class | README, CONTRIBUTING, CODE_OF_CONDUCT, issue templates. The tool practices what it preaches. |

Things that are *not* asked, because there's a right answer:

- **Layout.** `src/<package>/`, `notebooks/`, `data/raw`, `data/interim`, `data/processed` (git-ignored, each with a `.gitkeep` and a README explaining what belongs there), `reports/figures/`, `tests/`. This is cookiecutter-data-science's structure with the directories nobody uses removed.
- **Linting, formatting, type checking.** Ruff with a deliberately small rule set (`E F I UP B SIM NPY PD`) and ty, both run through `uv format` and `uv check`. Not optional, because the projects that skip this are the ones that need it.
- **Lockfile.** `uv.lock` is committed. Always.
- **Secrets.** A `.env.example` is generated, `.env` is ignored, and the README says so.
- **Testing.** pytest is always configured, even for a class project. An empty `tests/` with a passing smoke test costs nothing and means "add a test" is a two-minute task instead of a setup task.

## Validation

`preheat recipe check <recipe>` is the contract. For each answer fixture in the recipe's `tests/` directory, it renders the project to a temp directory and runs, in order: `uv sync --locked`, `uv format --check`, `uv check`, `uv run pytest`. If any step fails for any fixture, the recipe is broken. The built-in recipe runs this in CI for every PR, and the recipe-author guide will tell third-party recipe authors to do the same.

This is the mechanism behind the "works on the first `uv sync`" constraint. It's not clever, and it's the part I'm most confident in.

## Distribution

Three channels, in the order I'll ship them:

1. GitHub Releases with prebuilt binaries via `cargo-dist`, plus a shell installer.
2. PyPI via maturin, so `uvx preheat` and `uv tool install preheat` work. This is the channel that matters, because it's the one a student who already has uv can use without reading anything.
3. `cargo install preheat`, for people who have a Rust toolchain and prefer it.

Homebrew can come later if anyone asks.

## Why not the existing tools

**cookiecutter** is the incumbent and I've used it for years. Its `cookiecutter.json` is flat: no conditional questions, no conditional files, no help text. Hooks require a working Python. Templates can't be updated after generation. It's maintained but not evolving. Most of what I want is a set of features it was never designed to have.

**copier** is what I'd use if I weren't building this. It has conditional questions, conditional files, and a real update mechanism, and its `copier.yml` was a direct influence on `recipe.toml`. What it lacks: it's a Python tool, so it needs a Python before it can help; it has no concept of locked answers for a publishing instructor; and its TUI is a sequence of prompts rather than screens with a preview. If `preheat` fails, copier is the fallback and I'd be happy with it.

**`uv init`** is the base layer, not a competitor. It creates a minimal, correct project. It does not create `data/` directories, configure Ruff rules, add a notebook, write CI, or ask you anything. `preheat` calls it (or writes what it would write) and does the rest.

**GitHub template repositories** get you a fixed tree with no questions and no rendering. Fine for a course with exactly one shape of assignment; wrong the moment you have two.

## What I'm deliberately not building

- **A plugin system for the engine.** Recipes are the extension point. If a recipe can't do something, the answer is a feature in the engine, not a plugin API.
- **A web UI.** The terminal is where uv lives.
- **Support for conda, poetry, pip-tools, or pdm.** This tool is opinionated about uv. Someone else can fork it and swap the backend; the recipe format doesn't care.
- **Rendering arbitrary languages.** The engine could scaffold an R or Julia project, and someone may write that recipe. The built-in recipe and the validation harness are Python-only, and I'm not going to pretend otherwise.

## Open questions

- **Recipe discovery.** `gh:` URLs are enough for v0. A registry, or even a curated list in this repo, is a question for when there are more than three recipes.
- **`preheat update` semantics.** Copier's approach (three-way merge against the previous render) is the right idea and hard to get right. It's last on the roadmap for a reason.
- **How much the TUI should explain.** The `?` help is the current answer. A "tour" mode for first-time users is tempting and probably a distraction.
- **The name of the fourth profile.** "minimal" is accurate and boring.

## Risks

The obvious one: this whole design assumes uv stays the standard. uv's maintainer, Astral, was acquired by OpenAI in March 2026, with a stated commitment to keeping the tools open source. I take that at face value, and I also note that the recipe format has no dependency on uv beyond a handful of shell commands. If the ground shifts, the engine changes; the recipes don't.

The less obvious one: I'm a single maintainer with a thesis. The [governance doc](../GOVERNANCE.md) will say how a second maintainer gets added, and I intend to mean it.
