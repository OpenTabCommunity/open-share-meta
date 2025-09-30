# Releases & Versioning

This document defines release practices for Open Share across repositories.

## Semantic versioning
- Repositories use SemVer (MAJOR.MINOR.PATCH).
- Protocol versions are independent and included in messages (`protocol_version`).

## Release types
- **Patch**: Critical/fix release. Fast track via hotfix branch and cherry-pick.
- **Minor**: New backward-compatible features.
- **Major**: Incompatible changes; requires migration docs and cross-repo coordination.

## Release checklist
1. Changelogs: Update `CHANGELOG.md` under `[Unreleased]`.
2. Tests: All CI tests green, interop suite passes.
3. Documentation: Docs and examples updated.
4. Tagging: `git tag -s vX.Y.Z -m "Release vX.Y.Z"`.
5. GitHub Release: Draft notes with links to PRs and checksums for artifacts.
6. Publish: Binaries, crates, pip packages, or container images.
7. Announce: Discussions, Discord, and social channels.

## Cross-repo coordination
- Protocol updates should be released first and consumers released after interop testing.
- Use joint milestones and a release coordinator to manage dependencies.

## Emergency & backport policy
- Critical security patches: create hotfix branch from latest stable tag, cherry-pick & create patch release.
- Backports are done by maintainers with an explicit maintenance branch (e.g., `release/1.x`).

## Automation
- Recommend `semantic-release` or GitHub Actions for changelog generation and tagging.
- Use signed tags and GPG-signed release assets where possible.

