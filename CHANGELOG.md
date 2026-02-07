# Changelog

## [1.0.0] - 2026-02-05

### Added

- Initial release as Claude Code plugin
- 11 slash commands: init, decompose, plan, execute, execute-all, status, verify, resume, retry, abort, create-adapter
- 8 specialized agents: orchestrator, executor, decomposer, dependency-analyzer, state-manager, verifier, team-lead, team-executor
- 7 domain adapters: software-development, research, system-design, spec-driven-development, prediction-market, mixpanel-analytics, template
- 11 YAML schemas for validation
- Core skill with framework knowledge
- Teams mode for dynamic scheduling via Agent Teams
- Adapter discovery via `${CLAUDE_PLUGIN_ROOT}` with copy-on-init pattern

### Changed

- Migrated from standalone `.claude/` configuration to proper plugin structure
- Agent names changed from `ptf-*` to `ptf:*` (plugin namespace)
- Commands changed from `/ptf:*` in `.claude/commands/ptf/` to plugin-namespaced commands
- Adapters are now copied to `.orchestrator/adapters/` during init (portable across projects)
- Removed `@.claude/` file references in favor of skill auto-loading
