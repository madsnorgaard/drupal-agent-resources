# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/), and this project adheres to [Semantic Versioning](https://semver.org/) as described in [VERSIONING.md](VERSIONING.md).

## [Unreleased]

### Added
- `VERSIONING.md` — Versioning strategy, release standards, and quality criteria
- `CHANGELOG.md` — Project changelog following Keep a Changelog format
- Vision statement in README.md for Drupal AI-assisted development

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
