# preheat

A terminal wizard that sets up a data science project the way you'd set one up if you had an afternoon to spare and strong opinions. Built on [uv](https://docs.astral.sh/uv/), written in Rust, with the templates in plain Jinja so you never have to read the Rust. MIT licensed.

> **Status: pre-alpha.** There is nothing to install yet. What exists right now is a design, a recipe format, and this README. If you're here early, the [design doc](docs/design.md) is the most useful thing in the repo, and the issue tracker is where I'd love a second opinion.

## Why this exists

I'm a grad student at Northern Arizona University, and I have set up more analysis repositories than I care to count. Some were for my own research. A lot were for other people: a lab mate who needed a place to put a model, a class where the instructor wanted every student's submission to have the same shape, a workshop where thirty people needed a working environment in the first ten minutes or the rest of the session was lost.

Every one of those setups was the same forty-five minutes of the same decisions. Which Python. Where the data goes and how to keep it out of git. A `pyproject.toml` that actually works. Ruff config copied from the last project, then edited because the last project had a rule I hated. A README nobody would read. A `.gitignore` I always forgot something in. And then, if the project was for a class, doing the same thing again for a student whose laptop had three Pythons on it and none of them the right one.

Cookiecutter was supposed to solve this, and for a while it did. But cookiecutter templates can't ask a question conditionally, can't skip a file based on an earlier answer, and every hook needs a Python that already works, which is exactly the thing that isn't true on the machine that needs help most. Copier fixed some of that and added updates, but it still wants a working Python first. Neither one gives an instructor a way to say "these four answers are fixed, ask the student the other three."

uv changed what the floor looks like. It fetches its own Python, makes a lockfile without being asked, and now runs the formatter and type checker too. Once that's the foundation, the scaffolding tool on top of it has a much smaller job. `preheat` is my attempt at that smaller tool.

## What it does

You answer a handful of questions in your terminal and get a project directory that works on the first `uv sync`. The first question is what kind of project this is: a class assignment, a lab analysis, a reusable package, or the bare minimum. That choice sets sensible defaults for everything after it, and every later screen can be accepted with Enter. If you disagree with a default, there's a `?` on every question that explains why it's the default, so you can disagree on purpose.

What comes out: a `src/` layout, a pinned Python, a committed `uv.lock`, Ruff and ty wired in through `uv format` and `uv check`, a `data/` tree that git ignores, and, depending on what you picked, a notebook setup, pre-commit hooks, a `justfile`, CI, and the community files an open source project is supposed to have.

Two features are the reason I'm building this instead of forking something:

**Recipes with locked answers.** An instructor can publish a recipe that fixes some choices and hides those questions. Students see three questions instead of twelve, every submission has the same structure, and nobody has to write a setup guide.

**No Python required to run the wizard.** `preheat` is a single binary. It shells out to uv for everything Python-shaped, including its own hooks. If you can run `uvx`, you can run this.

## Installation

The only prerequisite is [uv](https://docs.astral.sh/uv/getting-started/installation/). `preheat` handles the rest, including fetching a Python if you don't have one.

Once there's a release, the intended way to run it is without installing anything:

```
uvx preheat new my-analysis
```

If you'd rather have it on your PATH:

```
uv tool install preheat
```

Prebuilt binaries for Linux, macOS, and Windows will be attached to each [GitHub release](https://github.com/NAU-OSS/preheat/releases), and `cargo install preheat` will work for anyone with a Rust toolchain who prefers it.

**Right now, none of those commands work**, because there is no release. Cloning the repo gets you the design document and this README. When the first tagged version lands, the commands above will be real and this paragraph will go away. If you want to know when that happens, watch the repo for releases.

## Usage

The interface below is what I'm building toward. It's settled enough to document; it isn't runnable yet.

**Interactive.** The default. Six screens, a preview of what will be written, then it writes.

```
uvx preheat new my-analysis
```

**Take the defaults for a profile.** Skips the wizard entirely. Useful when you already know what you want and just need the directory.

```
uvx preheat new hw3 --profile class --yes
```

**Reproduce a scaffold.** Every generated project gets a `.preheat.toml` recording the answers. Feed it back in to get the same structure again, on another machine or for another project.

```
uvx preheat new my-other-analysis --answers .preheat.toml
```

**Use an instructor's recipe.** The recipe pins the answers that matter for the course and hides those questions. The student sees only what's left.

```
uvx preheat new hw1 --recipe gh:NAU-OSS/sta574-f26
```

**Generate many projects from a script.** The Python package wraps the same engine, for anyone who needs to stamp out a repo per student.

```python
import csv
import preheat

recipe = preheat.Recipe.load("gh:NAU-OSS/sta574-f26")
with open("roster.csv") as f:
    for row in csv.DictReader(f):
        recipe.render(
            answers={"project_name": f"hw1-{row['netid']}", "author": row["name"]},
            dest=f"out/hw1-{row['netid']}",
        )
```

**Check that a recipe works.** Renders every answer fixture in the recipe's `tests/` directory and runs `uv sync`, `uv format --check`, `uv check`, and `uv run pytest` on each. This is what CI runs on every recipe PR.

```
uvx preheat recipe check ./my-recipe
```

## How it's built

Rust for the executable, because a tool whose job is to fix your Python environment shouldn't need one. The TUI is [ratatui](https://ratatui.rs). Templates are Jinja2-compatible ([minijinja](https://github.com/mitsuhiko/minijinja)) and recipes are a TOML file plus a `template/` directory, so contributing a recipe means editing the same kind of files you'd edit in a cookiecutter template. A thin Python package, built with maturin, wraps the same engine for the scripted use above.

The [design doc](docs/design.md) has the full argument, the recipe format, the list of what the wizard asks and why, and the things I decided not to build.

## Roadmap

Roughly in order. The [issues](https://github.com/NAU-OSS/preheat/issues) have the current state of each.

1. Recipe format and the rendering engine, with `preheat recipe check` so a recipe can prove it produces a working project.
2. The wizard, with one built-in recipe covering the four profiles.
3. Non-interactive mode (`--answers`, `--yes`) and the `.preheat.toml` answers file.
4. Recipes from git URLs, and locked answers.
5. PyPI distribution via maturin so `uvx preheat` works.
6. `preheat update`, for re-rendering an existing project after a recipe changes.

Things I've decided not to do are listed in the design doc, so nobody has to open an issue to find out.

## Getting help

- **Something's broken or missing:** [open an issue](https://github.com/NAU-OSS/preheat/issues/new). At this stage, an issue that says "I tried to imagine using this for my class and here's where it falls apart" is as useful as a bug report.
- **A question, or you're not sure it's an issue:** [Discussions](https://github.com/NAU-OSS/preheat/discussions). Q&A for questions, Ideas for everything else.
- **Something security-related:** please don't open a public issue. A `SECURITY.md` with a private reporting path is coming with the first code; until then, contact me directly through GitHub.

I read everything. I don't promise to reply within a day, but I do promise to reply.

## Contributing

I would like help, and not only with code. If you teach a course and would tell me what your ideal student repo looks like, that's a contribution. If you've fought with cookiecutter and have a list of grievances, that's a contribution. Recipe ideas, name suggestions for the profiles, disagreement with any of the defaults in the design doc: all welcome, in the issues or in Discussions.

A `CONTRIBUTING.md` with the mechanics (dev setup, how recipes are tested, what a good PR looks like) and a code of conduct are coming with the first code. Until then, opening an issue is the right move for anything.

## License

[MIT](license.md). The short reason: every layer this project sits on is MIT or MIT-compatible (uv, Ruff, ty, ratatui), and so is nearly everything else in the NAU-OSS organization, so it's the license that lets `preheat` fit in without anyone reading fine print. The longer reason: a scaffolding tool exists to be adopted, and its output belongs to whoever ran it. A copyleft license would raise the question of whether a project generated from a recipe inherits obligations from the recipe, and I'd rather that question never come up. Recipes you write and projects you generate are yours.

## Maintainer

Chris Reger, Northern Arizona University. Part of the [NAU-OSS](https://github.com/NAU-OSS) organization. The fastest way to reach me about this project is an issue or a Discussions post; both land in my inbox.
