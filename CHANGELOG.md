# Changelog

## v0.2.0 — Isolated Foundry Workspace (in development)

### Added
- Phase 0: Automated workspace setup — DeTest now works on any repo regardless of framework
- Isolated working directory at /tmp/detest-{project}-{timestamp}/ — original repo never modified
- Dependency detection from foundry.toml, remappings.txt, package.json, hardhat.config.js/ts, and raw import statement scanning
- Built-in dependency mapping table (16 known packages → forge install targets)
- Interactive resolution for unknown dependencies — one prompt per unknown, user provides owner/repo or skips
- Partial scope fallback — if a dependency cannot be resolved, affected contracts are excluded and documented rather than blocking the entire run
- forge build verification before proceeding to test writing
- references/dependency-map.md — canonical mapping of npm/import names to GitHub repos
- references/workspace-setup-rules.md — precise rules for the setup phase

### Changed
- SKILL.md — new STEP 0 (workspace setup) inserted before existing STEP 1
- PRD.Resource.txt — new Section 14 (v0.2.0 workspace setup) and Section 15 (dependency resolution)
- README.md — prerequisites updated, no longer requires foundry.toml

### Fixed
- DeTest no longer fails on Hardhat repos
- DeTest no longer fails on plain contract folders with no framework

## v0.1.0 — Initial Release

### Added
- Full 8-step pipeline: banner → references → report → source → config → classify → process → verdict
- 4 testability categories: STANDARD, MOCK, FORK, UNTESTABLE
- 6 verdict types: CONFIRMED, UNCONFIRMED-ASSERTION, UNCONFIRMED-COMPILE, UNCONFIRMED-WRONG-PATH, INCONCLUSIVE, SKIPPED
- bug-class-map.md covering 46 bug classes
- testability-rules.md with 4-step classification algorithm
- verdict-rules.md with 8-node decision tree
- Max 3 iterations per finding, 1 per lead
- Isolated test output to test/detest/
