# Changelog

Every change a user would notice goes here, most recent first. The format is [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and versions follow [Semantic Versioning](https://semver.org/) once there are versions to follow it. Until the first release, everything lives under Unreleased.

## [Unreleased]

### Added

- Design document (`docs/design.md`) covering the architecture, the recipe format, what the wizard asks and why, and what the project deliberately won't do.
- README with the project's reasoning, planned installation channels, usage examples for the interactive, non-interactive, instructor-recipe, and scripted paths, and where to get help.
- Contributing guide, code of conduct (Contributor Covenant 2.1), security policy, and this changelog.
- MIT license.

### Decided

- Rust for the executable and rendering engine; Jinja-compatible templates via minijinja; Python only through uv (hooks as PEP 723 scripts, a maturin-built package for scripted use).
- Recipes carry their own answer fixtures, and `preheat recipe check` must pass before a recipe merges.
- Remote recipes will require explicit confirmation before hooks run. See `SECURITY.md`.

[Unreleased]: https://github.com/NAU-OSS/preheat/commits/main
