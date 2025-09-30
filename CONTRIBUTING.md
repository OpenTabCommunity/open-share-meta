# Contributing to Open Share

Thanks for contributing! Whether you’re fixing a typo or submitting a major feature, this guide covers what you’ll need to make a high-quality contribution.

## Code of Conduct
We adopt the Contributor Covenant. Please be respectful and constructive.

## How to start
1. Fork the repository.2. Create a feature branch: `git checkout -b feature/<short-description>`.
3. Follow the branch/commit naming conventions below.4. Run local tests and CI linting before opening a PR.

## Issue workflow
- Search for existing issues before opening a new one.
- Use templates (bug/feature/security) and provide reproducible steps, logs, and the environment.

## Branch & commit conventions
- Branch names: `feature/<issue#>-short`, `bugfix/<issue#>-short`, `docs/<topic>`.
- Commit messages: Conventional Commits (http://conventionalcommits.org). Example:
  - `feat(discovery): add mDNS TXT parsing and tests`
  - `fix(server): ensure CRL endpoints respond with correct cache headers`
- Keep commit messages under 72 characters for the subject line.

## Pull request checklist
- [ ] Linked issue or RFC (if applicable).
- [ ] Tests added/updated.
- [ ] Documentation updated as needed.
- [ ] CHANGELOG entry under `[Unreleased]`.
- [ ] CI green.
- [ ] At least one maintainer requested as reviewer.
- [ ] Squash/sanitize commits before merge.

## Testing expectations
- Unit tests for new logic.
- Integration tests for protocol interactions.
- Fuzz tests for parsers and handshake code where possible.
- Add platform-specific tests if the change affects discovery/transport.

## Code style and linters
- Rust: `rustfmt`, `clippy`.
- Python: `black`, `ruff`, `pytest`.
- CI will enforce style; please run locally.

## Documentation
- Update `docs/` for any user-facing or developer-facing changes.
- Add mermaid diagrams to `docs/diagrams.md` and export generated SVGs to `assets/diagrams/`.

## Security & crypto changes
- File an RFC and discuss in the Security channel.
- Provide rationale, test vectors, and performance benchmarks.
- Security-critical PRs require at least two maintainers and a formal review.

## Becoming a maintainer
- Active contributors with sustained quality contributions may be invited to become maintainers.
- Maintainer responsibilities: review PRs, triage issues, participate in release coordination.

## Recognition
- Contributor list updated per release.
- Significant contributors are listed as authors in release notes.

