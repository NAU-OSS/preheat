# Security

`preheat` is pre-alpha with no releases, so there is nothing to patch yet. This file exists so that the reporting path is settled before it's needed, and so that the one security-relevant design decision in the project is written down where people will find it.

## Reporting a vulnerability

Please don't open a public issue for anything you think is a security problem.

Use GitHub's private vulnerability reporting instead: on the repo's [Security tab](https://github.com/NAU-OSS/preheat/security), choose "Report a vulnerability." That opens a private thread between you and me, and if it turns out to be real, GitHub handles the advisory and CVE paperwork. If for some reason you can't use that, open a blank issue that says only "security, please contact me" and I'll reach out through GitHub.

What helps: what you did, what happened, what you think an attacker could do with it, and a proof of concept if you have one. What doesn't help: guessing at severity. I'll do that part.

What you can expect from me:

- An acknowledgment within three days.
- An honest assessment within two weeks of whether I agree it's a vulnerability, and if so, a rough fix timeline.
- Credit in the advisory and the changelog, unless you'd rather not be named.
- No legal threats, ever, for good-faith research. Poking at this tool to find holes is a favor.

Once there are releases, only the most recent minor version will get security fixes. I'm one person; I won't pretend to backport.

## The thing worth knowing about: recipes run code

A recipe is not just templates. It can include `hooks/pre_gen.py` and `hooks/post_gen.py`, which are Python scripts that `preheat` runs on your machine through `uv run --script`. That's a feature: a hook can initialize git, rewrite files based on your answers, or fetch a dataset. It's also exactly what it sounds like: running a recipe from a URL you don't trust is running someone's code.

Treat `--recipe gh:someone/something` the way you'd treat `pip install something` or `curl | sh`. If you wouldn't install a package from that person, don't run their recipe.

What `preheat` will do to make this less of a foot-gun, in rough order of when it lands:

1. **Show hooks before running them.** For any recipe that isn't built in, the review screen lists each hook, its inline dependencies from the PEP 723 header, and asks for confirmation. `--yes` does not skip this for remote recipes; there will be a separate `--trust-hooks` flag so that the decision is deliberate.
2. **Pin remote recipes to a commit.** `gh:org/repo@<sha>` is the recommended form for anything used in a class, so a recipe can't change under students between when the instructor reviewed it and when they run it. The `.preheat.toml` answers file records the resolved commit.
3. **No network access from templates.** Jinja rendering is sandboxed by construction: `minijinja` templates can't import, open files, or make requests. Only hooks can, and only after you've said yes.
4. **Hook dependencies come from uv's cache, not from the recipe.** A recipe can name dependencies; it can't ship wheels. What gets installed is whatever PyPI (or your configured index) serves for those names, which is the same trust you already extend to `uv add`.

None of this makes running an untrusted recipe safe. It makes it visible. If you find a way for a recipe to run code without going through the confirmation step, that's the highest-severity bug this project can have, and I'd like to hear about it through the channel above.

## Out of scope

- Vulnerabilities in uv, Ruff, ty, or any other tool `preheat` calls. Report those upstream; I'll happily bump the minimum version once they ship a fix.
- Security problems in projects *generated* by `preheat`, unless the generated file itself is the problem (a bad `.gitignore` that leaks `.env`, a CI workflow with an injection hole). Those I do want to know about.
- Anything requiring an attacker to already have write access to your machine or your repo.
