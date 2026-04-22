# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/), and this project adheres to [Semantic Versioning](https://semver.org/) as described in [VERSIONING.md](VERSIONING.md).

## [Unreleased]

## [0.2.0] - 2026-04-22

### Added
- `drupal-commerce-9-to-10` — Commerce D9→D10 upgrade skill with a phase-structured runbook and deterministic recovery recipes for `__PHP_Incomplete_Class` in `DefaultTableMapping`, `commerce_stripe_update_8102`, `commerce_stripe_update_8104` (with required follow-up checklist before production rollout), `dblog_update_10100` (`MySQL server has gone away` on `ALTER TABLE watchdog`), and orphaned `system.schema` entries. Schema version overrides use integer serialization (`i:N;`) and prefer `\Drupal::keyValue('system.schema')->set()` to avoid brittle hand-written PHP string serialization (#14, #15)

### Known limitations
- The migration-module-group audit (Phase 1) and post-update migration-group integrity check (Phase 3) in `drupal-commerce-9-to-10` have not been validated against a real D7-to-D10 Commerce migration. Tracked in #16

## [0.1.1] - 2026-03-07

### Added
- `VERSIONING.md` — Versioning strategy, release standards, and quality criteria
- `CHANGELOG.md` — Project changelog following Keep a Changelog format
- Vision statement in README.md for Drupal AI-assisted development
- Document `--overwrite` flag for updating already-installed resources

## [0.1.0] - 2026-01-28

Initial collection of Drupal agent resources.

### Added

#### Skills
- `drupal-expert` — Drupal 10/11 development expertise (modules, themes, services, hooks)
- `drupal-security` — Security patterns and vulnerability prevention (XSS, SQL injection, access bypass)
- `drupal-migration` — Migrate API expertise (D7-to-D10, CSV imports, custom plugins)
- `ddev-expert` — DDEV local development (commands, Xdebug, custom services)
- `docker-local` — Custom Docker Compose patterns for non-DDEV projects

#### Commands
- `/drush-check` — Run health checks on a Drupal site
- `/module-scaffold` — Generate a new module with best-practice structure
- `/config-export` — Export Drupal configuration with review workflow
- `/security-audit` — Audit site for security vulnerabilities
- `/performance-check` — Analyze caching, queries, and optimization opportunities

#### Agents
- `drupal-reviewer` — Code review agent for Drupal (security, standards, performance, DI compliance)

#### Documentation
- `README.md` — Installation guide with Linux/WSL instructions
- `CLAUDE.md` — Development guidelines for contributing
- `ROADMAP.md` — Phased development plan (Drupal 11, PHP 8.4, testing, performance)
- `RENAME_INSTRUCTIONS.md` — Repository rename guide (agent-resources → drupal-agent-resources)

#### CI/CD
- `validate.yml` — GitHub Actions workflow for YAML frontmatter validation and link checking
