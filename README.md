# preheat

A terminal wizard that sets up a data science project the way you'd set one up if you had an afternoon to spare and strong opinions. Built on [uv](https://docs.astral.sh/uv/), written in Rust, with the templates in plain Jinja so you never have to read the Rust. MIT licensed.

> **Status: pre-alpha.** There is nothing to install yet. What exists right now is a design, a recipe format, and this README. If you're here early, the [design doc](docs/design.md) is the most useful thing in the repo, and the issue tracker is where I'd love a second opinion.

## Why this exists

I'm a grad student at Northern Arizona University, and I have set up more analysis repositories than I care to count. Some were for my own research. A lot were for other people: a lab mate who needed a place to put a model, a class where the instructor wanted every student's submission to have the same shape, a workshop where thirty people needed a working environment in the first ten minutes or the rest of the session was lost.

Every one of those setups was the same forty-five minutes of the same decisions. Which Python. Where the data goes and how to keep it out of git. A `pyproject.toml` that actually works. Ruff config copied from the last project, then edited because the last project had a rule I hated. A README nobody would read. A `.gitignore` I always forgot something in. And then, if the project was for a class, doing the same thing again for a student whose laptop had three Pythons on it and none of them the right one.

Cookiecutter was supposed to solve this, and for a while it did. But cookiecutter templates can't ask a question conditionally, can't skip a file based on an earlier answer, and every hook needs a Python that already works, which is exactly the thing that isn't true on the machine that needs help most. Copier fixed some of that and added updates, but it still wants a working Python first. Neither one gives an instructor a way to say "these four answers are fixed, ask the student the other three."

uv changed what the floor looks like. It fetches its own Python, makes a lockfile without being asked, and now runs the formatter and type checker too. Once that's the foundation, the scaffolding tool on top of it has a much smaller job. `preheat` is my attempt at that smaller tool.

## What it does

You run one command:

```
uvx preheat new my-analysis
```

A wizard walks you through a handful of screens. The first one asks what kind of project this is: a class assignment, a lab analysis, a reusable package, or the bare minimum. That choice sets sensible defaults for everything after it, and every later screen can be accepted with Enter. If you disagree with a default, there's a `?` on every question that explains why it's the default, so you can disagree on purpose.

At the end you get a directory with a `src/` layout, a pinned Python, a committed `uv.lock`, Ruff and ty wired in through `uv format` and `uv check`, a `data/` tree that git ignores, and, depending on what you picked, a notebook setup, pre-commit hooks, a `justfile`, CI, and the community files an open source project is supposed to have. It works on the first `uv sync`. That's the whole promise.

Two features are the reason I'm building this instead of forking something:

**Recipes with locked answers.** An instructor can publish a recipe that fixes some choices and hides those questions. Students run `uvx preheat new --recipe gh:NAU-OSS/sta574-f26` and see three questions instead of twelve. Every submission has the same structure, and nobody has to write a setup guide.

**No Python required to run the wizard.** `preheat` is a single binary. It shells out to uv for everything Python-shaped, including its own hooks. If you can run `uvx`, you can run this.

## How it's built

Rust for the executable, because a tool whose job is to fix your Python environment shouldn't need one. The TUI is [ratatui](https://ratatui.rs). Templates are Jinja2-compatible ([minijinja](https://github.com/mitsuhiko/minijinja)) and recipes are a TOML file plus a `template/` directory, so contributing a recipe means editing the same kind of files you'd edit in a cookiecutter template. A thin Python package, built with maturin, wraps the same engine so you can stamp out forty student repos from a CSV in a loop.

The [design doc](docs/design.md) has the full argument, the recipe format, the list of what the wizard asks and why, and the things I decided not to build.

## Roadmap

Roughly in order. See the [issues](https://github.com/NAU-OSS/preheat/issues) for the current state.

1. Recipe format and the rendering engine, with `preheat recipe check` so a recipe can prove it produces a working project.
2. The wizard, with one built-in recipe covering the four profiles.
3. Non-interactive mode (`--answers`, `--yes`) and the `.preheat.toml` answers file, so a scaffold is reproducible.
4. Recipes from git URLs, and locked answers.
5. PyPI distribution via maturin so `uvx preheat` works.
6. `preheat update`, for re-rendering an existing project after a recipe changes.

## Contributing

I would like help, and not only with code. If you teach a course and would tell me what your ideal student repo looks like, that's a contribution. If you've fought with cookiecutter and have a list of grievances, that's a contribution. Recipe ideas, name suggestions for the profiles, disagreement with any of the defaults in the design doc: all welcome, in the issues or in Discussions.

A `CONTRIBUTING.md` with the mechanics (dev setup, how recipes are tested, what a good PR looks like) is coming with the first code. Until then, opening an issue is the right move for anything.

## License

[MIT](license.md). The short reason: every layer this project sits on is MIT or MIT-compatible (uv, Ruff, ty, ratatui), and so is nearly everything else in the NAU-OSS organization, so it's the license that lets `preheat` fit in without anyone reading fine print. The longer reason: a scaffolding tool exists to be adopted, and its output belongs to whoever ran it. A copyleft license would raise the question of whether a project generated from a recipe inherits obligations from the recipe, and I'd rather that question never come up. Recipes you write and projects you generate are yours.

## Maintainer

Chris Reger, NAU. Part of the [NAU-OSS](https://github.com/NAU-OSS) organization.
